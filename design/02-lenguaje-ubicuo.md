# Ubiquitous language — compost ↔ SIEX digital layer

Companion to [01-flujo-datos.md](01-flujo-datos.md). That sketch names actors, canonical
objects, flows and components; this note fixes **one meaning per word** for them, in the
Domain-Driven Design sense, so that conversation, documents and (later) code use the same
vocabulary.

Everything here is proposal. Nothing below asserts a new fact about the law or the SIEX
schema: regulated terms are anchored to the note in [../research/](../research/) that
sources them, and where the research does not establish something it is listed under
[Open questions](#8-open-questions) rather than defined. Written 2026-10-13 against the
research dated 2026-09-23 (Anexo V v3.11, Anexo VI v3.11.4).

## 1. Language rules

1. **Regulated concepts keep their Spanish name.** If the law or the FEGA spec names a
   thing, we use that name and do not translate it in conversation: *representación*,
   *entidad habilitada*, *plan de abonado*, *explotación*. A translation is a second word
   for the same thing, and it will drift.
2. **Schema names are verbatim and in code formatting** (`Fertilizacion`, `CodigoDGC`,
   `IdAjenaFerti`). They name the wire format, not our model.
3. **Our own concepts are English, PascalCase when they are model objects**
   (`CompostBatch`, `FarmLink`). These are ours to define, and they are defined here.
4. **One word, one meaning, per bounded context.** Where 01 uses one word for two
   things, §7 picks a winner. The "Avoid" column lists the losers.
5. **A mirror is not the original.** `Parcel` is our read-only copy of a REA fact;
   `NutrientDemand` is our reading of a `PlanAbonado`. We say "the Parcel" for ours and
   "the DGC" / "the `PlanAbonado`" for the state's.

## 2. Bounded contexts

Eight contexts, grouped by which side of the market they face. The 01 component that
implements each is in the last column.

| Context | Purpose | Core terms | Component in 01 §6 |
|---|---|---|---|
| **Supply** | What a producer has to offer, and proof of what is in it | Producer, `CompostBatch`, `BatchAssay`, Product Passport | Passport registry |
| **Holding** | Who farms what, where: our mirror of REA | Holder, Holding, `Parcel`, DGC | (IUWS client, read side) |
| **Representation** | Who may act for whom, within what limits | *Representación*, *entidad habilitada*, `FarmLink` | Consent store |
| **Demand** | What nutrients are being sought, where and when | `NutrientDemand`, Demand Surface | Demand surface |
| **Dosing** | How much of a batch a parcel should receive, and why | Limiting Rate, Legal Ceiling, Recommendation | Dose engine |
| **Fulfilment** | What physically moved and what was physically spread | `Delivery`, `Application` | — (none named) |
| **Filing** | Getting an Application into the CUE and keeping it correct | `CueSubmission`, Activity Record, Rejection | IUWS client, Reconciliation queue |
| **Catalogue** | The SIEX code lists every other context is validated against | Catalogue, Catalogue Version, Catalogue Code | Catalogue service |

A ninth, **Soil Evidence** (flow G in 01), has vocabulary but no canonical object yet; it
is in §4.9 and flagged in §8.

### Context map

```
              ┌───────────┐
              │ CATALOGUE │  upstream of Supply, Dosing, Filing
              └─────┬─────┘  (published language: SIEX catalogue codes)
        ┌───────────┼──────────────────┐
        ▼           ▼                  ▼
   ┌────────┐   ┌────────┐        ┌────────┐      ┌──────────────────────┐
   │ SUPPLY │──▶│ DOSING │───────▶│ FILING │─────▶│ REA + CUE (per CCAA) │
   └────┬───┘   └────────┘        └────────┘ ACL  │  via IUWS            │
        │           ▲                 ▲           └──────────┬───────────┘
        │       ┌───┴────┐            │                      │ ACL
        │       │ DEMAND │◀───────────┼──────────────────────┤
        │       └────────┘            │                      ▼
        │                       ┌─────┴──────┐          ┌─────────┐
        └──────────────────────▶│ FULFILMENT │◀─────────│ HOLDING │
                                └────────────┘          └─────────┘

   REPRESENTATION gates every arrow that crosses into REA + CUE, in either direction.
```

- **REA + CUE is an upstream we conform to and cannot influence.** Holding, Demand and
  Filing each translate it at the boundary (anti-corruption layer): schema names stop
  there and our terms start. There are 17 deployments of one spec
  ([research/02](../research/02-siex-rea-cue.md)), so "the CUE" always means *a given
  Comunidad's* CUE.
- **Catalogue is a published language**, not ours: we version and consume it.
- **Representation is not a peer context but a guard.** No read or write crosses the
  IUWS boundary without a `FarmLink` in scope (01 §4.A, §6).

## 3. Actors

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| **Producer** | An organisation that makes compost and places it on the market. Always the *compost* producer. Modelled as `ProducerOrg` | NIMA, REGFER, NIF — [research/01](../research/01-marco-normativo.md), [research/04](../research/04-modelo-datos-fertilizacion.md) | "supplier" (see Supplier Identity), "plant", bare "producer" for a farmer |
| **Holder** | The person legally responsible for a Holding and for its *plan de abonado*; the one who grants *representación* | *titular*; RD 1051/2022 art. 4.2 — [research/01](../research/01-marco-normativo.md) | "owner", "customer" |
| **Farmer** | Informal synonym for Holder, acceptable in conversation and marketing. In the model and in anything about consent or liability, say Holder | — | "grower", "productor" |
| ***Productor particular*** | A Holder authenticating to IUWS directly, with no company certificate. An IUWS role, **not** a Producer in our sense | `comunicacionProductor` — [research/03](../research/03-interfaz-unico-ws.md) | "producer" |
| ***Entidad habilitada*** | The entity authorised, per Comunidad, to read a Holder's REA and complete their CUE as their representative. Our partner, never us | [research/02](../research/02-siex-rea-cue.md) | "entity", "partner" (alone), "gestora" (see §8) |
| **Applicator** | Whoever physically spreads the material: the Holder themselves or a service company. When a company, it is the `EmpresaServicios` and is identified by its *Código REGFER* | Anexo V field 476 — [research/04](../research/04-modelo-datos-fertilizacion.md) | "spreader", "contractor" |
| **Lab** | The laboratory that issues a `BatchAssay` | — | — |
| **Digital Layer** | Us: the system described in 01. In Phase 2 its legal form is a *CUE comercial* | [research/02](../research/02-siex-rea-cue.md) | "the platform", "the CUE" (that is the notebook) |
| ***CUE comercial*** | Authorised commercial notebook software, entered on a Comunidad's list. A legal status the Digital Layer may hold, per Comunidad | Anexo X — [research/02](../research/02-siex-rea-cue.md) | "the app" |

## 4. Glossary by context

### 4.1 Supply

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| `ProducerOrg` | The Producer as we hold it: legal name plus its regulated identifiers | 01 §3 | — |
| **Supplier Identity** | The single identifier by which a Producer is named in a `Fertilizacion` record: exactly one of NIMA, NIF or REGA. A value object, chosen once per Producer and material | Anexo V fields 458–461 — [research/04](../research/04-modelo-datos-fertilizacion.md) | "producer ID" |
| `CompostBatch` | A quantity of compost from one Producer that shares one composition, and so one `BatchAssay`. The unit a Delivery draws from | 01 §3 | "lot", "product", bare "batch" (see §7) |
| `BatchAssay` | A Lab's analysis of one `CompostBatch`: macronutrients, micronutrients, heavy metals, density, moisture, C:N | 01 §3 | "analysis", "analítica" (that is soil — §4.9) |
| **Composition** | The part of a `BatchAssay` that the CUE can carry, expressed as % by weight | `Macronutrientes`, `Micronutrientes`, `MetalesPesados` — [research/04](../research/04-modelo-datos-fertilizacion.md) | "specs" |
| **Material Class** | The SIEX catalogue classification of what the batch *is*; determines which Composition fields are mandatory | `Material`, `DetalleMaterial` — [research/04](../research/04-modelo-datos-fertilizacion.md) | "product type" |
| **Product Passport** | A `CompostBatch` together with its `BatchAssay`, normalised to CUE units and pre-validated against a Catalogue Version. What a Producer "registers" (01 §4.D) | 01 §4.D | "spec sheet", "datasheet" |
| **Sludge-derived** | A batch whose origin triggers `AplicacionLodos` and its documentation requirement | Anexo V field 473 — [research/04](../research/04-modelo-datos-fertilizacion.md) | — |

### 4.2 Holding

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| **Holding** | A farm as registered in REA, identified by its *Nº explotación*. The thing a CUE belongs to | *explotación* — [research/02](../research/02-siex-rea-cue.md), [research/03](../research/03-interfaz-unico-ws.md) | "farm" in the model, "account" |
| **REA** | The Comunidad's registry of Holdings: the master data. We read it; we never write it | [research/02](../research/02-siex-rea-cue.md) | — |
| **DGC** | The state's identifier for a parcel, issued only in the REA export. The sole legal handle for a CUE write | `CodigoDGC` — [research/03](../research/03-interfaz-unico-ws.md) | "SIGPAC reference", "recinto" |
| `Parcel` | Our read-only mirror of one DGC: its code, surface, crop and *secano*/*regadío*. Valid for one campaign (§8) | 01 §3 | "field", "plot" |
| **REA Pull** | Exporting a Holder's REA to create or refresh their Parcels. Precedes everything else for that Holding | `exportarREA` — 01 §4.B | "sync" |
| ***Unidad de producción*** | The unit to which the *plan de abonado* obligation attaches. A legal term; not yet a model object (§8) | RD 1051/2022 art. 4.2 — [research/01](../research/01-marco-normativo.md) | using it as a synonym for Parcel |
| ***Comunidad Autónoma*** | The region operating a given REA + CUE deployment. The unit of authorisation, endpoint and conformance | [research/02](../research/02-siex-rea-cue.md) | "region" in the model (fine in prose), "CCAA" in prose |

### 4.3 Representation

There are four distinct permissions in this domain. Never say "authorisation" or
"consent" without saying which.

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| ***Representación*** | The Holder's signed authorisation for an *entidad habilitada* to act for them on REA and CUE | Anexo VIII — [research/02](../research/02-siex-rea-cue.md) | "consent", "mandate", "power of attorney" |
| `FarmLink` | Our record of one *representación*: Holder, *entidad habilitada*, Scope and Expiry | 01 §3 | "consent record", "link" |
| **Scope** | What a `FarmLink` covers. Any read or write outside it is refused | 01 §4.A | "permissions" |
| **Expiry** | The date after which a `FarmLink` authorises nothing | 01 §4.A | — |
| **Entity Registration** | The *entidad habilitada*'s own registration with a Comunidad. Theirs, not the Holder's and not ours | [research/02](../research/02-siex-rea-cue.md) | "authorisation" |
| **Commercial Authorisation** | The Digital Layer's entry on a Comunidad's list of *CUE comerciales* | Anexo X — [research/02](../research/02-siex-rea-cue.md) | "authorisation", "certification" |
| **Acting Entity** | The single *entidad habilitada* declared in an IUWS call. Exactly one per call | Anexo VI §2.5.1 — [research/02](../research/02-siex-rea-cue.md) | "tenant" |

### 4.4 Demand

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| ***Plan de abonado*** | The fertilization plan a Holder must draw up. The legal obligation, and the `PlanAbonado` record that files it | RD 1051/2022 arts. 4.2, 6 — [research/01](../research/01-marco-normativo.md); [research/04](../research/04-modelo-datos-fertilizacion.md) | "fert plan", "nutrient plan" |
| **Nutrient Requirement** | The N, P₂O₅ and K₂O a crop needs, in *unidades fertilizantes*, as declared in a `PlanAbonado` | `NecesidadUFN`, `NecesidadUFP2O5`, `NecesidadUFK2O` — [research/04](../research/04-modelo-datos-fertilizacion.md) | "deficit" |
| `NutrientDemand` | Our reading of one `PlanAbonado` for one Parcel: Nutrient Requirement, production target, preceding crop, plan date. **Per parcel, never aggregated** | 01 §3 | bare "demand" |
| **Demand Pull** | Exporting a Holding's CUE to derive its `NutrientDemand`s | `exportar` — 01 §4.C | — |
| **Demand Surface** | The aggregate of many `NutrientDemand`s by municipality, crop and window. What Producers buy. **Aggregated, never per parcel** | 01 §4.C | "heatmap", "the demand data" |

### 4.5 Dosing

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| **Match** | A candidate pairing of one `CompostBatch` with one `Parcel`, before any quantity is decided | 01 §4.E | "lead", "order" |
| **Availability Coefficient** | The fraction of a nutrient in the batch that the crop can use in the first season (`k_avail`, `k_P`, `k_K`). Always sourced, always recorded with its provenance. **No default exists** | 01 §4.E | "efficiency", "factor" |
| **Limiting Rate** | The dose, in t/ha, at which one nutrient's Requirement is exactly met. One each for N, P and K | 01 §4.E | "N rate" |
| **Legal Ceiling** | A dose limit imposed by regulation rather than by agronomy. A hard stop, not an input to be weighed | 01 §4.E | "cap", "limit" |
| **Recommended Dose** | The lowest of the Limiting Rates and Legal Ceilings, in t/ha. What the dose engine outputs | 01 §4.E | "computed rate", "proposed rate", "rate" |
| **Binding Constraint** | Whichever Limiting Rate or Legal Ceiling set the Recommended Dose | 01 §4.E | "bottleneck" |
| **Headroom** | For each constraint that did not bind, how far the Recommended Dose sits below it | 01 §4.E | "slack", "margin" |
| **Recommendation** | A Recommended Dose with its Binding Constraint, Headroom and every coefficient used. A proposal: whether it may ever be more than that depends on §8 | 01 §4.E | "prescription", "plan" |

### 4.6 Fulfilment

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| `Delivery` | Compost physically arriving at a Holding: tonnes, date, batch, destination. A logistics fact; creates no legal obligation | 01 §3 | "shipment", "order" |
| **Delivery Note** | The paper document that accompanies a Delivery. A source of data, and the place Supplier Identity is most often copied wrong | 01 §5 | *albarán* is fine in conversation |
| `Application` | Compost physically spread on one or more Parcels on a known date at a known dose. **The legal trigger**: it starts the Filing Window | RD 1051/2022 art. 5 — [research/01](../research/01-marco-normativo.md) | "spread", "delivery" |
| **Applied Dose** | The dose actually spread, as recorded in an Application. Distinct from Recommended Dose (§8) | `Dosis` — [research/04](../research/04-modelo-datos-fertilizacion.md) | "rate" |
| **Confirmed** | Of an Application: attested by the Holder or Applicator as having happened. Only a Confirmed Application may be filed | 01 §4.F | "done", "closed" |

### 4.7 Filing

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| **CUE** | A Holding's digital field notebook, held by its Comunidad. The thing we file into | [research/02](../research/02-siex-rea-cue.md) | "notebook software" (that is a *CUE comercial*) |
| **IUWS** | The web service in front of REA and CUE. One spec, one deployment per Comunidad | Anexo VI — [research/03](../research/03-interfaz-unico-ws.md) | "the SIEX API" |
| **Activity Record** | One entry in a CUE. We compose `Fertilizacion` and `PlanAbonado`, and possibly `Analitica`; we never touch the other kinds | [research/04](../research/04-modelo-datos-fertilizacion.md) | "row", "entry" |
| **Fertilization Record** | The Activity Record for one Application: one `Fertilizacion` per Parcel | 01 §4.F | "the CUE entry" |
| **Foreign Id** | The identifier we mint for an Activity Record. Minted once, persisted, never reused; the only handle for correcting or retracting it | `IdAjena*` — [research/03](../research/03-interfaz-unico-ws.md) | "external id", "our id" |
| **Submission Call** | One `POST /crear/`, carrying Activity Records for exactly one Comunidad and one Acting Entity | [research/03](../research/03-interfaz-unico-ws.md) | "batch" (see §7), "upload" |
| `CueSubmission` | Our durable record of one Submission Call and everything that follows from it: its `IdPeticion`, status and per-record outcomes | 01 §3 | "request", "job" |
| **Outcome** | What the CUE did with one Activity Record: Imported or Rejected. Known only by polling, and per record, not per call | `comprobarEstado` — [research/03](../research/03-interfaz-unico-ws.md) | "response", "result" |
| **Rejection** | An Outcome in which the CUE refused one Activity Record, with its message. Normal, expected, and not an error in our system | [research/03](../research/03-interfaz-unico-ws.md) | "failure", "bounce", "error" |
| **Reconciliation** | Taking a Rejection to resolution: a person corrects the cause and the record is Resubmitted under the same Foreign Id | 01 §4.F, §6 | "retry" (implies no human) |
| **Retraction** | Removing a filed Activity Record by resubmitting its Foreign Id with `Borrar` set | [research/03](../research/03-interfaz-unico-ws.md) | "delete" |
| **Filing Window** | The one month after an Application within which it must be recorded | RD 1051/2022 art. 5 — [research/01](../research/01-marco-normativo.md) | "deadline", "SLA" |
| **Escalation** | Handing an unresolved Rejection to the *entidad habilitada* before the Filing Window closes | 01 §4.F | — |

Proposed lifecycle of a Fertilization Record, using only states 01 already implies:

```
Composed ──▶ Submitted ──▶ Imported ──▶ (Retracted)
                 │
                 └──▶ Rejected ──▶ Resubmitted ──▶ …
                          │
                          └──▶ Escalated
```

### 4.8 Catalogue

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| **Catalogue** | A SIEX code list. Plural in practice: material, method, unit, treatment and others | Anexo VII — [research/04](../research/04-modelo-datos-fertilizacion.md) | "enum", "lookup", "constants" |
| **Catalogue Code** | One value from a Catalogue. Meaningless without its Catalogue Version | [research/04](../research/04-modelo-datos-fertilizacion.md) | "ID" |
| **Catalogue Version** | The published revision a code was taken from. Stored with every Product Passport and Activity Record | 01 §7.5 | — |
| **Pre-validation** | Checking a Passport or Activity Record against a Catalogue Version before submission. Reduces Rejections; does not replace Outcomes | 01 §4.D | "validation" (the CUE validates; we pre-validate) |

### 4.9 Soil Evidence

| Term | Definition | Anchor | Avoid |
|---|---|---|---|
| **Soil Analysis** | A laboratory analysis of a Parcel's soil, as filed in the CUE | `Analitica`, `ParametrosSuelo` — [research/04](../research/04-modelo-datos-fertilizacion.md) | "assay" (that is compost — §4.1) |
| **SOM Trend** | The change in soil organic matter on a Parcel across campaigns | `MateriaOrganica` — 01 §4.G | "soil health score" |
| **Supplied Parcel** | A Parcel that has received at least one Application of a given Producer's compost. The population a Producer's SOM Trend is computed over | 01 §4.G | "customer parcel" |

## 5. Aggregates and invariants

Rules 01 already states, attached to the object that must enforce them. Each should be
sayable in the language above without reaching for a schema name.

| Aggregate root | Contains | Invariants |
|---|---|---|
| `ProducerOrg` | Supplier Identity | Exactly one of NIMA, NIF, REGA is used to name the Producer in a record (01 §5) |
| `CompostBatch` | `BatchAssay`, Material Class | A batch has one Composition. A Product Passport names the Catalogue Version it was pre-validated against (01 §7.5) |
| `FarmLink` | Scope, Expiry | Nothing is read or written for a Holding without a `FarmLink` in Scope and unexpired (01 §4.A) |
| Holding | `Parcel`s | No Parcel exists without a DGC from a REA Pull. Parcels are re-pulled at campaign rollover and on any REA change (01 §4.B) |
| `NutrientDemand` | — | Refers to exactly one Parcel. Never edited by us: it changes only when the `PlanAbonado` does |
| Recommendation | Limiting Rates, Legal Ceilings | Every Availability Coefficient has a recorded source. A Legal Ceiling is never exceeded. The Binding Constraint is always stated (01 §4.E) |
| `Application` | Applied Dose, Parcels | Refers only to Parcels with a DGC ("REA before CUE", 01 §7.2). Must be Confirmed before filing |
| `CueSubmission` | Fertilization Records, Outcomes | One Comunidad and one Acting Entity per Submission Call (01 §7.3). A Foreign Id is never reused. A Submission is not finished until every record has an Outcome (01 §7.4) |

## 6. The language in use

A test of the vocabulary: flow F of 01, told without ambiguity.

> A **Producer** registers a **Product Passport** for a `CompostBatch`. The **Demand
> Surface** shows Requirement for N in a municipality; a **Match** pairs the batch with a
> **Holder**'s `Parcel`, and the dose engine issues a **Recommendation** whose **Binding
> Constraint** is the N **Limiting Rate**. A `Delivery` arrives. Days later the
> **Applicator** spreads it and the `Application` is **Confirmed**, which opens the
> **Filing Window**. Because the Holder's `FarmLink` is in **Scope**, we compose one
> **Fertilization Record** per Parcel, each with a new **Foreign Id**, and send them in a
> **Submission Call** naming the **Acting Entity**. Polling returns one **Outcome** per
> record: two **Imported**, one **Rejection** on a **Catalogue Code**. It goes to
> **Reconciliation**, is corrected and **Resubmitted** under the same Foreign Id. Had it
> stayed unresolved, it would have gone to **Escalation** before the window closed.

## 7. Ambiguities in 01, resolved

Words 01 uses for more than one thing, or more than one word for the same thing.

| In 01 | Problem | Resolution |
|---|---|---|
| "producer" | Compost producer throughout, but IUWS's *productor particular* is a farmer ([research/03](../research/03-interfaz-unico-ws.md)) | **Producer** is always compost. The IUWS role is always written *productor particular*, in Spanish |
| "batch" | `CompostBatch` (§3) and "batching by `(region, entidad habilitada)`" (§6) | **Batch** is compost only. The IUWS grouping is a **Submission Call** |
| "delivery" / "application" / "confirmed spread" | Flow F is titled "Delivery → CUE write" and §5 "delivery note → `Fertilizacion`", but the record is of spreading, not arrival | `Delivery` and `Application` are different events. Only a Confirmed `Application` is filed and only it starts the Filing Window |
| "recommended" / "proposed rate" / "computed rate" / `Dosis` | Four names; and §5 maps the computed value straight into `Dosis` | **Recommended Dose** is what we propose; **Applied Dose** is what happened. See §8 for which one is filed |
| "parcel" / DGC / *recinto* / *unidad de producción* | Used loosely as one thing | **DGC** is the state's identifier, `Parcel` is our mirror. *Unidad de producción* is a separate legal term (§8) |
| "consent" / *representación* / "authorisation" | "Authorisation" covers four different permissions | *Representación*, Entity Registration, Commercial Authorisation, and the access model of Anexo IX (§8). Never bare "authorisation" |
| "CUE" | The notebook (§1) and our legal position, "a CUE comercial" (§2) | **CUE** is the notebook. The software status is always *CUE comercial* |
| "entity" / "entidad" / "partner" | Three words, one role | *Entidad habilitada* in full; **Acting Entity** for the one named in a call |
| "product passport" | A flow (§4.D) and a component (§6), but not in the canonical objects (§3) | **Product Passport** = `CompostBatch` + `BatchAssay`, normalised and pre-validated |
| "demand" | A per-parcel object and a regional aggregate | `NutrientDemand` is per parcel; **Demand Surface** is the aggregate. Bare "demand" is avoided |
| "assay" / "analítica" | Both are lab results | **Assay** is of compost; **Soil Analysis** is of soil |
| "record" / "activity" / "submission" | Interchangeable in §4.F | **Activity Record** is one entry; `CueSubmission` is one call's worth of them |
| "rejection" / "bounce" / "error" | Three words | **Rejection**. "Error" is reserved for faults in our own system |
| "region" / "Comunidad" / "CCAA" / "deployment" | Four words | ***Comunidad Autónoma*** in the model; "region" tolerated in prose |
| "us" / "our digital layer" | The system has no name | **Digital Layer** |

## 8. Open questions

Places where a term cannot be pinned down from what `research/` currently establishes.
None is answered here.

- **Recommended Dose vs Applied Dose.** 01 §5 maps the computed rate to `Dosis`. If the
  Applicator spreads a different amount, which is filed? This is the vocabulary face of
  the liability question in 01 §9, and it decides whether "Recommendation" may ever be
  renamed something stronger.
- **Which Supplier Identity a composting plant uses.**
  [research/04](../research/04-modelo-datos-fertilizacion.md) describes NIF as applying to
  a *centro gestor* and NIMA to wastes applied as fertilizing material, calls a composting
  plant a *centro gestor* with a NIMA, and records the two as mutually exclusive. 01 §5
  lists both as options. The rule for choosing is not sourced.
- **The Producer's REGFER number has no place in a Fertilization Record.** 01 §2–§3 give
  the Producer a REGFER identity, but the only *Código REGFER* field
  [research/04](../research/04-modelo-datos-fertilizacion.md) documents (476) identifies
  the service company. So "REGFER" names two different parties depending on context:
  the Producer's registration (ours to hold) and the Applicator's (filed).
- ***Unidad de producción* ↔ DGC.** The plan obligation attaches to the former
  ([research/01](../research/01-marco-normativo.md)); `PlanAbonado` references the latter
  ([research/04](../research/04-modelo-datos-fertilizacion.md)). The mapping between them
  is not in the research, so `NutrientDemand` "per Parcel" is an assumption.
- **"Campaign".** 01 §4.B and §4.G use it as the validity period of a Parcel and the
  interval of the SOM Trend. It is not defined in any research note.
- ***CIF Gestora*.** A path parameter of both export methods
  ([research/03](../research/03-interfaz-unico-ws.md)) with no definition in the research.
  The local extract (`research/sources/anexoVI.txt`, line 348) describes it as the CIF of
  the *empresa gestora* making the request; that needs verifying against Anexo VI
  v3.11.4 and recording in research/03 before we decide whether *gestora* is a synonym
  for Acting Entity, for the Digital Layer, or a third thing.
- **Anexo IX, *autorización de acceso REA-CUE*.** Listed in
  [research/02](../research/02-siex-rea-cue.md) but not described. Whether it is part of
  `FarmLink`'s Scope or a fourth permission with its own term is unknown.
- **Scope has no stated shape.** 01 says to store it and refuse writes outside it, but
  not what it ranges over: Holdings, activity types, read vs write, dates.
- **Material Class for compost.** The term is defined; its values are blocked on Anexo
  VII, as is which Composition fields are mandatory
  ([research/05](../research/05-fuentes.md), Gaps).
- **Soil Evidence has no canonical object.** Flow G reads `Analitica` but 01 §3 has no
  corresponding object, and no component in §6 owns it.
- **Fulfilment has no component.** `Delivery` and `Application` are canonical objects
  with no owner in 01 §6.
- **Abonado en verde.** [research/04](../research/04-modelo-datos-fertilizacion.md) flags
  it as a legitimate branch in which the material blocks are empty. 01 does not mention
  it, so the language has no term for a Fertilization Record without a `CompostBatch`.
