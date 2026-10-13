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
| Boundary enforcement | **eslint-plugin-boundaries** |
| Hosting, Phases 0 and 1 | Backend as a **container** on a managed platform; **Supabase** for PostgreSQL; EU region |

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
- **Hosting must suit a long-lived process.** pg-boss workers poll continuously, so
  the backend cannot run on a platform that scales to zero. Prisma and pg-boss both
  hold database sessions, so check which Supabase connection mode each needs when
  scaffolding.
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
- **The front end is a client of the API only.** No server rendering; a public
  marketing site, if one is wanted, is a separate concern.

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

- Which managed platform runs the backend container in Phases 0 and 1.
- Whether Supabase is used for anything beyond PostgreSQL. Until decided, it is the
  database only.

### Workspace layout

Directory names are English, following language rule 3 in
[02-lenguaje-ubicuo.md](02-lenguaje-ubicuo.md) §1.

| Path | What it is | Bounded context | First needed |
|---|---|---|---|
| `apps/api` | The NestJS backend | — | Phase 0 |
| `apps/api/src/holding` | NestJS module | Holding | Phase 0 |
| `apps/api/src/demand` | NestJS module | Demand | Phase 0 |
| `apps/api/src/representation` | NestJS module | Representation | Phase 0 |
| `apps/api/src/supply` | NestJS module | Supply | Phase 1 |
| `apps/api/src/fulfilment` | NestJS module | Fulfilment | Phase 1 |
| `apps/api/src/filing` | NestJS module | Filing | Phase 1 |
| `apps/api/src/iuws` | NestJS module | none: transport only | Phase 2 |
| `apps/web` | The React front end | — | Phase 0 |
| `packages/dosing` | Framework-free package | Dosing | Phase 1 |
| `packages/catalogue` | Framework-free package | Catalogue | Phase 1 |
| `packages/shared-kernel` | Framework-free package | none: shared value objects | Phase 0 |

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
- **Soil Evidence has no module** until it has a canonical object (02 §8).
