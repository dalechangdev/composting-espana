# Sketch — compost ↔ SIEX data flow

Companion to [../research/](../research/). Everything asserted about the national
schema is sourced there; everything here is design.

## 1. The thesis in one line

The CUE already forces a farmer to name their compost supplier by regulated identifier
and to declare their crop's N/P/K deficit before the season — so the compost market's
matching data is *already being filed by law*, just not connected. We build the
connection.

## 2. Actors

```
  ┌──────────────────┐                             ┌──────────────────┐
  │ COMPOST PRODUCER │                             │      FARMER      │
  │  · NIMA (waste)  │                             │  · NIF titular   │
  │  · REGFER opnum  │                             │  · Nº explotación│
  │  · batch assays  │                             │  · DGC parcels   │
  └────────┬─────────┘                             └────────┬─────────┘
           │ product passport                    consent /  │
           │ + delivery events                representación│
           ▼                                                ▼
      ╔═══════════════════════════════════════════════════════════╗
      ║                    OUR DIGITAL LAYER                      ║
      ║   passport registry · matching · dose calc · CUE writer   ║
      ╚═══════════════════════╤═══════════════════════════════════╝
                              │ acts under the authorisation of
                              ▼
                  ┌────────────────────────┐
                  │  ENTIDAD HABILITADA    │  (co-op / asesoría partner)
                  └───────────┬────────────┘
                              │ IUWS: cert (OV) → JWT
                              ▼
        ┌──────────────────────────────────────────────┐
        │  REA + CUE  — one deployment per Comunidad    │
        │  https://{servidor}:{puerto}/IUWS/api/...     │
        └───────────────────────┬──────────────────────┘
                                ▼
                            ┌───────┐
                            │ SIEX  │  (national)
                            └───────┘
```

Our legal position: a **CUE comercial** (authorised commercial notebook software),
operating on behalf of one or more **entidades habilitadas**, each holding signed
*representación* from its farmers. Per Anexo VI §2.5.1 this is the "centralised service"
case — our certificate identifies us as the software company, and each payload must
declare which entidad habilitada it is acting for, with **one entity per call**.

> Alternative worth pricing: skip the commercial-CUE authorisation initially and have
> the partner entidad habilitada call IUWS with their own certificate, us supplying only
> the prepared payload. Slower per transaction, far faster to first revenue, and it
> defers 17 regional authorisation processes. See §8.

## 3. Canonical objects

| Object | Owner | Maps to |
|---|---|---|
| `ProducerOrg` | us | NIMA + REGFER + NIF |
| `CompostBatch` | producer | → `MaterialFertilizante` composition fields |
| `BatchAssay` | lab | %N, %P₂O₅, %K₂O, micros, heavy metals, density, moisture, C:N |
| `FarmLink` | farmer + entidad | *representación* scope + expiry |
| `Parcel` | REA (read-only) | `CodigoDGC`, surface, crop, secano/regadío |
| `NutrientDemand` | CUE `PlanAbonado` (read) | UF N / P₂O₅ / K₂O, objetivo, cultivo precedente |
| `Delivery` | producer | tonnes, date, batch, destination farm |
| `Application` | farmer/applicator | → `Fertilizacion` record |
| `CueSubmission` | us | `IdAjenaFerti`, `IdPeticion`, status, per-record errors |

`IdAjena*` values are ours to mint and must be **stable and persisted** — they are the
only handle for later correction or `Borrar`.

## 4. Flows

### A. Onboarding — consent before data

Farmer signs *representación* (FEGA Anexo VIII model) to the partner entidad habilitada;
entidad registers in its region; our software is on that region's authorised
commercial-CUE list (Anexo X). Store scope + expiry; **refuse any write outside it.**
This is the slowest step and it is not a technical one.

### B. REA pull — establish parcel identity

```
POST /IUWS/api/comunicacion/{appCue}          → JWT
GET  /IUWS/api/exportarREA/{NIF}_{CIF Gestora} → parcels, crops, CodigoDGC
```

**Hard prerequisite.** `CodigoDGC` exists only in the REA export; SIGPAC coordinates
were deliberately removed from the activity descriptors to stop exactly the kind of
mis-referencing we would otherwise cause. Cache DGCs per campaign; re-pull on campaign
rollover and on any REA change.

### C. Demand pull — the matching signal

```
GET /IUWS/api/exportar/{Nº explotación}_{CIF Gestora}  → CUE incl. PlanAbonado[]
```

Yields, per parcel: `NecesidadUFN`, `NecesidadUFP2O5`, `NecesidadUFK2O`,
`ObjetivoProduccion`, `CultivoPrecedente`, plan date. Aggregate to a **regional
nutrient-demand surface**: kg of N/P/K sought, by municipality, by crop, by window.

That is the product. A compost producer today guesses where demand is; this measures it
from filings the farmer is legally obliged to make.

### D. Product passport — the producer side

Producer registers each `CompostBatch` with its assay. We normalise to the exact units
the CUE wants (composition as **% by weight**; density in `kg/m3 | g/m3 | t/m3 | kg/L |
mg/L`) and pre-validate against the SIEX `Material Fertilizante` catalogue, including
which composition fields are mandatory for that material class.

> Blocked on Anexo VII — see research gaps. Until we have the catalogue we cannot know
> which of `Macronutrientes`/`Micronutrientes`/`MetalesPesados` are obligatory for
> compost specifically. Get this before writing the validator.

### E. Match and dose

Given a parcel's `NutrientDemand` and a batch assay, the recommended rate is the
*minimum* of several ceilings — this is the core logic and the part that must be
defensible to an agronomist:

```
kg nutrient per tonne of product  =  %nutrient × 10        (% of fresh matter)

N-limited rate   (t/ha) = NecesidadUFN     / (%N    × 10 × k_avail)
P-limited rate   (t/ha) = NecesidadUFP2O5  / (%P2O5 × 10 × k_P)
K-limited rate   (t/ha) = NecesidadUFK2O   / (%K2O  × 10 × k_K)

recommended = min(N-limited, P-limited, K-limited, legal ceilings)
```

`k_avail` is the first-season availability coefficient for organic N — compost N is
largely organic and mineralises slowly, so this is well below 1 and it dominates the
result. **Do not invent it.** Source it per region from the official code of good
agricultural practice and, where the SIEX catalogues encode it, from there. Same for
`k_P`, `k_K`. Flag every recommendation with the coefficient used and its provenance.

Legal ceilings to encode as hard stops:
- **170 kg N/ha/year from organic manures in zonas vulnerables** (Nitrates Directive,
  as transposed regionally) — verify the exact regional transposition.
- Heavy-metal limits by product class (RD 506/2013 classes A/B/C), checked against the
  batch's `MetalesPesados`.
- Sewage-sludge-derived material triggers `AplicacionLodos` and its own documentation
  requirement (CUE field 473, RD 1310/1990).

Output: a proposed rate in `t/ha`, the binding constraint, and the headroom on the others.

### F. Delivery → CUE write

On a confirmed spread, compose one `Fertilizacion` record per parcel and POST:

```
POST /IUWS/api/crear/     → { IdPeticion, IdCuaderno }
GET  /IUWS/api/comprobarEstado/{IdPeticion}  → per-activity results
```

**Import is asynchronous and partial failure is normal.** The spec's own examples are
rejections for bad catalogue units and bad parcel references. We need a durable
submission record, a poll loop, and a human-facing queue for records that bounce.

The one-month legal window (RD 1051/2022 art. 5) becomes an SLA with an escalation:
day 0 submit, day 7 alert on unresolved rejections, day 21 escalate to the entidad.

### G. Feedback loop

Pull `Analitica.ParametrosSuelo` over successive campaigns — `MateriaOrganica`,
`NitrogenoTotal`, `Ph`, `Conductividad`, texture. The farmer must have this data anyway
(art. 6 requires soil organic matter in the plan). Trend SOM on parcels a producer has
supplied and the producer gets an evidence-backed sales argument assembled from their
customers' own regulatory filings. This is the retention mechanism, and it costs one
extra read.

## 5. Field mapping — delivery note → `Fertilizacion`

| Source | CUE field | Note |
|---|---|---|
| Application date | `FechaInicio` / `FechaFin` | `dd/MM/yyyy` |
| Parcel | `DGCs[].CodigoDGC` + `Superficie` | from REA pull (§B) |
| Batch material class | `MaterialFertilizante.Material` | SIEX catalogue |
| Batch subtype | `.DetalleMaterial` | catalogue; inorganic/organomineral groups |
| Producer legal name | `.EmpresaSuministradora` | **Obligatorio** for manures/wastes |
| Producer **NIMA** | `.NIMA` | waste-derived → NIMA. **Mutually exclusive** with NIF/REGA |
| Producer NIF (centro gestor) | `.NifEmpresa` | alternative to NIMA/REGA |
| Livestock origin code | `.REGA` | alternative, for manures |
| Composting process | `.TratamientoEstiercoles` | catalogue |
| Assay N/P/K | `.Macronutrientes[]` | `{TipoMacroN, Porcentaje}` |
| Assay micros | `.Micronutrientes[]` | |
| Assay metals | `.MetalesPesados[]` | |
| Bulk density | `.Densidad` + `.UnidadesMedida` | liquids |
| Computed rate (§E) | `AplicacionMaterialFertilizante.Dosis` + `.Unidad` | `kg/ha \| L/ha \| m3/ha \| t/ha` |
| Application type/method | `.TipoFertilizacion`, `.MetodoFertilizacion` | catalogue |
| Sludge flag | `.AplicacionLodos` | |
| Spreader company | `.EmpresaServicios` + **Código REGFER** | REGFER obligatory since 2026 |
| Machinery | `EquipoAplicador.*` | mostly voluntary |
| Our record id | `IdAjenaFerti` | mint, persist, never reuse |

The exclusivity rule on NIF/REGA/NIMA is a validation the UI must enforce at entry —
it is the field most likely to be wrong when filled from a paper delivery note.

## 6. Components

| Component | Responsibility |
|---|---|
| **Catalogue service** | Sync SIEX catalogues (Anexo VII), versioned. Not a constants file — several documented rejections are catalogue violations, and the annexes revise at least annually |
| **IUWS client** | Per-region endpoint registry, cert + JWT lifecycle, batching by `(region, entidad habilitada)`, async status polling |
| **Consent store** | Representación scope + expiry; authorises every read and write |
| **Passport registry** | Batches, assays, unit normalisation, catalogue pre-validation |
| **Dose engine** | §E, with per-region coefficients and legal ceilings, fully auditable |
| **Reconciliation queue** | Rejected activities, human resolution, resubmission |
| **Demand surface** | Aggregated `PlanAbonado` analytics for producers |

## 7. Design constraints inherited from the spec

1. **17 deployments, one spec.** Conformance varies; as of 2024 reporting, full
   round-trip was confirmed in Castilla y León. Pick launch regions on measured
   conformance, not on market size.
2. **REA before CUE, always.** No DGC, no write.
3. **One entidad habilitada per call.**
4. **Async, partially-failing writes.** Reconciliation is core, not an edge case.
5. **Catalogue-coupled.** Version the catalogue with every stored record so we can
   explain a past submission.
6. **September 2027**: FEGA replaces the separate agricultural/livestock technical
   documents with a horizontal one at SIEX v5.9.0. Budget the migration now.

## 8. Phasing

**Phase 0 — read-only, no authorisation needed from us.** Partner entidad habilitada
calls IUWS with their own certificate and shares REA + `PlanAbonado` exports. We build
the demand surface and sell that to compost producers. Tests the thesis with no
regulatory surface of our own.

**Phase 1 — assisted write.** We compose and validate `Fertilizacion` payloads; the
partner posts them. Proves the mapping and the dose engine against real rejections.

**Phase 2 — commercial CUE.** Pursue our own authorisation (Anexo X) region by region,
starting wherever Phase 1 showed the cleanest conformance.

**Phase 3 — advisory.** Art. 6.6 makes technical advice mandatory one year after the
plan obligation in zonas vulnerables, two elsewhere — roughly **1 September 2027** for
the general case. `PlanAbonado.Herramienta` is a boolean the regulator put in
specifically to record that a digital advisory tool was used. Being that tool, with a
compost bias, is the durable position.

## 9. Open questions

- **Anexo VII catalogues** — blocks the passport validator and the dose engine. Get first.
- Regional N-availability coefficients: official source per CCAA?
- Exact regional transposition of the 170 kg N/ha zonas vulnerables ceiling.
- Is there an IUWS sandbox? Anexo VI references an *entorno de integración*.
- Anexo X: real cost and elapsed time to authorise a commercial CUE, per region.
- Who is contractually liable for a wrong CUE entry we composed — us, the entidad, or
  the holder? This determines how much the dose engine may decide versus merely propose.
- Implementation choices deferred or still open: see "Deferred" and "Not yet decided"
  in §10.
- What evidence of *representación* does each launch region accept, and does a
  document signed with AutoFirma qualify? Not covered by the research.
- May a private company's site integrate AutoFirma, and under what licence terms?
  Unverified; check before building the `identity` module's signing flow.
- Which certificate issuers and certificate types (personal, company representative,
  DNIe) do we accept, and what is the official source for the trusted-issuer list?
- In a co-official-language interface, are regulated terms such as *representación*
  translated or kept in Spanish? [02-lenguaje-ubicuo.md](02-lenguaje-ubicuo.md) §1
  rules only on Spanish and English.
- Which municipality code does a parcel carry in the REA export? The local extract of
  the Anexo V `REA` sheet (section 12, "Información de la Delimitación Gráfica")
  describes it as the municipality "de la referencia SIGPAC", while the holder's
  address fields use the INE list. If the two schemes differ, the demand surface (§4.C)
  needs a sourced mapping before parcels can be joined to municipality boundaries.
  Verify against Anexo V v3.11 and record in `research/`.
- Does Scaleway hold the certifications a region may require (see the next item)?
  Unverified.
- Does a region require a commercial CUE's own infrastructure to meet a named security
  standard under Anexo X? This decides hosting from Phase 2.

## 10. Implementation stack

Decided 2026-10-14. No code exists yet; this records the choices so the first scaffold
follows them.

| Concern | Choice |
|---|---|
| Language | **TypeScript**, for every component in §6 and for the web front end |
| Package manager | **pnpm**, using pnpm workspaces |
| Monorepo build orchestration | **Turborepo** |
| Backend framework | **NestJS**, which implies Node.js as the runtime |
| Database | **PostgreSQL** with the **PostGIS** extension |
| Data-access layer | **Prisma** |
| Durable jobs | **pg-boss**, a job queue stored in PostgreSQL |
| Web front end | **React** single-page app built with **Vite** |
| Decimal arithmetic | **decimal.js** |
| Workspace layout | **Hybrid**: framework-free packages for pure domain logic, NestJS modules for everything else. Table below |
| Authentication | **Better Auth**, running inside the backend, with users stored in our PostgreSQL through Prisma |
| Certificate identity and signing | **AutoFirma** signature flow, validated by the backend |
| Document storage | **Scaleway Object Storage** in the **Paris** region, reached over the S3 protocol |
| API style | **REST**, with an **OpenAPI** contract generated from the NestJS backend and a typed client generated from that contract |
| Client generator | **Orval** |
| Front-end routing | **TanStack Router** |
| Front-end data fetching | **TanStack Query** |
| Tables | **TanStack Table** |
| Maps | **MapLibre GL** |
| Base map | **None at first**; later a Spain extract of OpenStreetMap as a single **Protomaps** file on Scaleway |
| Municipality boundaries | The national mapping agency's (**CNIG**) official municipal boundary lines, loaded into PostGIS |
| Component kit | **shadcn/ui**, which brings in **Tailwind CSS** |
| Forms and validation | **React Hook Form** with **Zod** |
| Backend testing | **Vitest**, with **Testcontainers** for a real PostgreSQL and **fast-check** for property-based tests |
| Front-end testing | **Vitest** with **React Testing Library** for components; **Playwright** for end-to-end journeys |
| Interface language | **Spanish** first; co-official languages added by launch region |
| Translation library | **react-i18next**, with the browser's built-in number and date formatting |
| Boundary enforcement | **eslint-plugin-boundaries** |
| Code hosting | **GitHub** |
| Continuous integration and deployment | **GitHub Actions** runs the checks; **Render** deploys once they pass |
| Logging and error tracking | **Pino** structured logs; **Sentry**, EU-hosted, for errors |
| Audit trail | **Append-only table** in PostgreSQL, written in the same transaction as the change |
| Transactional email | **Scaleway Transactional Email** |
| Hosting, Phases 0 and 1 | Backend as a **container** on **Render**; **Supabase** for PostgreSQL; both in **Frankfurt** |

Consequences for the design:

- **One repository, one API app.** The bounded contexts in
  [02-lenguaje-ubicuo.md](02-lenguaje-ubicuo.md) §2 become NestJS modules inside a single
  backend app, one module per context. Two start as standalone packages with no NestJS
  dependency: **Dosing**, so the dose engine (§4.E) can be tested and audited in
  isolation and reused by the front end, and **Catalogue**, because Supply, Dosing and
  Filing all validate against it. Another context is extracted into a package only when
  it gains a second consumer.
- **Module boundaries are enforced by lint.** Inside one app nothing stops one
  context's module importing another's internals, so eslint-plugin-boundaries
  restricts cross-context imports from the first commit. A module may import another
  only through its public entry point, and nothing imports `iuws` except Holding,
  Demand and Filing.
- **The backend runs as two Render services from one container image.** A web
  service serves the API and a background worker runs the pg-boss jobs. The workers
  poll continuously, so neither may be on a plan that suspends when idle. Prisma and
  pg-boss both hold database sessions, so check which Supabase connection mode each
  needs when scaffolding.
- **Names in code come from the ubiquitous language.** Model objects use the English
  PascalCase names in 02 (`CompostBatch`, `FarmLink`); schema names (`Fertilizacion`,
  `CodigoDGC`) appear only in the modules that translate at the IUWS boundary.
- **No floating-point arithmetic in the dose engine.** Doses, percentages and
  coefficients are decimal.js values end to end and are stored in `numeric` columns,
  which Prisma returns as decimal.js values rather than JavaScript numbers.
- **The submission lifecycle (§4.F) is a state machine we write.** pg-boss supplies
  durable jobs and scheduling, not workflows: the poll loop and the day 7 and day 21
  checks are jobs, and the record states are ours to model. A job is enqueued in the
  same database transaction as the domain change that caused it, so a `CueSubmission`
  cannot exist without its poll job or the reverse.
- **PostGIS and Prisma do not fully meet.** Prisma is not expected to model geometry
  columns as first-class types, so the demand surface's (§4.C) spatial columns and
  queries will likely go through raw SQL. Verify against the Prisma version in use
  when scaffolding, and keep that SQL inside the Demand module.
- **Authentication says who; Representation says what for.** Better Auth establishes
  a user's identity and their membership of a producer or an entidad habilitada.
  Whether that user may read or write for a given farm is decided only by a
  `FarmLink` in the Representation module (§4.A), never by an auth role alone.
- **Certificates supplement password login; they do not replace it.** Every account
  can sign in with a password and a second factor. A user who holds a Spanish digital
  certificate can additionally sign a challenge with AutoFirma; once the backend has
  validated that signature, the certificate's identity is attached to the account as
  a stronger proof.
- **The same flow can sign documents.** Because AutoFirma produces a signature, not
  just a login, it can be used to sign a *representación* and keep the signed
  document as evidence beside its `FarmLink`. Whether a region accepts a document
  signed this way is not established (§9), so this is a capability, not yet a flow.
- **Signature validation is ours to get right.** The backend checks the issuer,
  revocation status and certificate type of every signature. The list of trusted
  issuers and the accepted certificate types must come from an official source, not
  from a constant in code (§9).
- **Certificate support is desktop-first.** AutoFirma is an application the user
  installs. The expected certificate users are entidad habilitada staff at a desk,
  not farmers on a phone.
- **No hosting constraint follows.** A signature flow needs no control over the TLS
  connection, so it works on Render. Login by
  browser client certificate would need that control and is left for Phase 2, if
  users ask for it.
- **Documents live outside the database.** Lab reports, delivery notes and signed
  *representación* forms go to Scaleway Object Storage; PostgreSQL holds only their
  references. The backend reaches the store through one storage interface, so the
  provider can change without touching the contexts. Files are private and are served
  through short-lived links the backend issues after its `FarmLink` check.
- **Signed forms should be stored write-once.** A signed *representación* is evidence,
  so its bucket should use versioning and object locking with a retention period.
  Confirm Scaleway's current support, and the retention period required, when
  scaffolding.
- **Supabase is the database only.** Neither Supabase Auth nor Supabase Storage is
  used.
- **The front end is a client of the API only.** No server rendering; a public
  marketing site, if one is wanted, is a separate concern.
- **The OpenAPI contract is the API's single description.** It is generated from the
  backend, never written by hand, and the front end calls the API only through the
  client generated from it. A contract that is out of date with the backend should
  fail the build. Being language-neutral, the same contract is what a producer's or a
  cooperative's own software would integrate against.
- **Orval generates more than the client.** From the OpenAPI contract it produces the
  typed functions, the TanStack Query hooks the screens use, and mock handlers for the
  component tests. All of it lands in `packages/api-client`.
- **Filters live in the URL.** The demand surface is explored by municipality, crop
  and window (§4.C); with TanStack Router those are typed URL parameters, so a view
  can be linked to and reloaded.
- **The demand surface is drawn without a base map at first.** Municipalities coloured
  by demand read well on a plain background. When users need roads and towns for
  orientation, a Spain-only OpenStreetMap extract is added as one Protomaps file in
  Scaleway Object Storage, so the map depends on no outside service at runtime.
- **Boundaries are reference data with a source.** The CNIG dataset's licence,
  version and download date go in `research/05-fuentes.md` before it is loaded. It is
  stored at full detail in PostGIS and simplified for display. Joining parcels to it
  depends on the municipality-code question in §9.
- **Interface components are ours to maintain.** shadcn/ui copies component source
  into `apps/web`; it is not a dependency that updates itself.
- **Form rules come from the API contract where they can.** Orval can generate Zod
  schemas from the OpenAPI contract, so a form and the backend check the same rule.
  Two rules from this design live in forms: exactly one of NIMA, NIF and REGA (§5),
  and dose fields that accept a decimal comma.
- **The database sits beside the backend.** Render and Supabase are both placed in
  Frankfurt so that queries do not cross regions. Documents are in Paris, where the
  distance does not matter. Confirm both providers still offer these regions.
- **Migrations run before each release.** Prisma's migrations are applied as a step
  of the Render deploy, before the new version starts, and a failed migration stops
  the release.
- **The product is Spain-only.** Maps, boundary data, locale handling and hosting
  choices need to serve Spain and nothing else; nothing should be generalised for
  other countries.
- **The audit trail is domain data, not logging.** Every read or write made under a
  `FarmLink`, every recommendation issued and every submission is recorded in an
  append-only table with who acted, for which farm, under which `FarmLink`, and why.
  It is written in the same transaction as the change it describes, so the two cannot
  disagree. Logs go to Pino and may be discarded; audit rows may not.
- **Logs must not carry personal data.** NIFs and document contents stay out of Pino
  and Sentry; log the internal identifier instead.
- **Tracing is deferred.** OpenTelemetry is added at Phase 2, when the system starts
  calling IUWS itself and a request's path across regions is worth following.
- **One test runner across the repository.** Vitest runs the packages, the backend and
  the front-end components. NestJS relies on decorator metadata that Vitest's default
  compiler does not emit, so the backend's test setup needs a compiler plugin; confirm
  the current recipe when scaffolding.
- **Database tests run against a real PostgreSQL with PostGIS**, started by
  Testcontainers, not against a mocked Prisma. The raw spatial SQL in Demand and the
  transactional enqueue with pg-boss cannot be trusted otherwise.
- **The dose engine's invariants are tested as properties.** fast-check generates
  inputs to check rules that must hold for every case, above all that a recommended
  dose never exceeds a legal ceiling (§4.E). These sit beside, not instead of, worked
  examples an agronomist has approved.
- **End-to-end tests are few and chosen.** Playwright covers the journeys where a
  failure is costly, starting with resolving a rejected record (§4.F). AutoFirma is a
  desktop application no browser test can drive, so the signing step is replaced by a
  stand-in in tests and needs a manual check before release.
- **No user-facing text is hard-coded.** Every string goes through a translation key
  from the first screen, so adding a language is translation work, not a rewrite.
- **Numbers and dates follow the user's locale.** Spanish uses a decimal comma, and a
  misread dose is a real error, so no number is formatted or parsed by hand in the
  front end. The API itself always exchanges locale-neutral values.

Deferred:

- **Where the qualified certificate's private key is held** (§2). Not needed in Phases
  0 and 1, where the partner entidad habilitada calls IUWS with its own certificate
  (§8). Decide at Phase 2; the starting point is a secrets manager that loads the key
  into the backend process at start, behind an interface so that a proxy or a
  hardware-backed key can replace it.

- **Hosting from Phase 2.** It depends on the key-storage decision above and on what
  security requirements a region imposes on a commercial CUE under Anexo X, which the
  research does not yet establish. Keeping the backend a plain container (below) is
  what makes this deferrable.

Not yet decided:

- Which co-official languages are added, which follows from the launch regions.

### Workspace layout

Directory names are English, following language rule 3 in
[02-lenguaje-ubicuo.md](02-lenguaje-ubicuo.md) §1.

| Path | What it is | Bounded context | First needed |
|---|---|---|---|
| `apps/api` | The NestJS backend | — | Phase 0 |
| `apps/api/src/holding` | NestJS module | Holding | Phase 0 |
| `apps/api/src/demand` | NestJS module | Demand | Phase 0 |
| `apps/api/src/representation` | NestJS module | Representation | Phase 0 |
| `apps/api/src/identity` | NestJS module | none: authentication and signature validation | Phase 0 |
| `apps/api/src/supply` | NestJS module | Supply | Phase 1 |
| `apps/api/src/fulfilment` | NestJS module | Fulfilment | Phase 1 |
| `apps/api/src/filing` | NestJS module | Filing | Phase 1 |
| `apps/api/src/iuws` | NestJS module | none: transport only | Phase 2 |
| `apps/web` | The React front end | — | Phase 0 |
| `packages/dosing` | Framework-free package | Dosing | Phase 1 |
| `packages/catalogue` | Framework-free package | Catalogue | Phase 1 |
| `packages/shared-kernel` | Framework-free package | none: shared value objects | Phase 0 |
| `packages/api-client` | Generated package | none: typed client for the OpenAPI contract | Phase 0 |

Build in phase order: the layout is fixed now, but a directory is created only when
its phase starts.

- **`iuws` is transport, not translation.** It owns the per-region endpoint registry,
  the certificate and the JWT lifecycle (the "IUWS client" of §6). Translating between
  schema names and our terms stays in Holding, Demand and Filing, as 02 §2 requires.
  It is not needed before Phase 2: in Phase 0 the partner's exports arrive as files and
  are loaded through an import path in Holding and Demand, and in Phase 1 Filing
  composes payloads that the partner posts (§8).
- **`shared-kernel` holds value objects only**: regulated identifiers, units and
  decimal quantities that more than one context must agree on. No entities, no
  services, no NestJS. Anything added to it needs a reason two contexts cannot each
  own their copy.
- **`identity` is not a bounded context either.** It wraps Better Auth and the
  AutoFirma signature validation, and answers one question: who is this user, and
  how strongly is that proven. It knows nothing about farms; Representation consumes
  its answer.
- **`api-client` is generated output.** It is produced from the backend's OpenAPI
  contract and is never edited by hand. `apps/web` depends on it and on nothing inside
  `apps/api`.
- **Soil Evidence has no module** until it has a canonical object (02 §8).
