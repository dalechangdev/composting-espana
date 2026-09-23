# 02 — SIEX, REA, CUE and who is allowed to write to them

## The three acronyms

| Acronym | Full name | What it is |
|---|---|---|
| **SIEX** | Sistema de Información de Explotaciones agrícolas y ganaderas y de la producción agraria | National system: a set of interconnected databases and administrative registries characterising every farm in Spain |
| **REA** | Registro autonómico de Explotaciones Agrícolas | Per-region farm registry: holder, parcels, crops, tenure. The *master data* |
| **CUE** | Cuaderno digital de Explotación agrícola | Per-farm digital field notebook: the *activity log* — treatments, fertilization, harvest, grazing |

All three created by RD 1054/2022. Crucially, **SIEX is national but REA and CUE are
operated by each Comunidad Autónoma.** There are 17 implementations behind one common
interface spec. This is the single biggest architectural fact for our project.

## Legal layer defining content and timing

- **Orden APA/204/2023, de 28 de febrero** — minimum content of REA and CUE, plus the
  chronology for onboarding data sources into SIEX:
  https://www.boe.es/buscar/act.php?id=BOE-A-2023-5699
- **Resolución de 25 de mayo de 2023** (Secretaría General de Agricultura y
  Alimentación) — delegates the technical detail to a coordination document published
  on the FEGA site, updated at least annually:
  https://www.boe.es/diario_boe/txt.php?id=BOE-A-2023-13035

Notably the Orden's CUE section does **not** enumerate separate fields for manure or
compost; it points at RD 1051/2022 and says the fertilization block must carry what
that decree requires, including the *plan de abonado*. The actual field list lives in
the FEGA technical annexes, not in the BOE.

## FEGA technical documentation (the real spec)

Index: https://www.fega.gob.es/es/siex/documentacion-tecnica-agricola-siex

Current agricultural version: **v3.11, deployed November 2025**.

| Annex | Title | Format |
|---|---|---|
| — | Documento Técnico v3.11 | PDF |
| III | Cronología de incorporación de fuentes de información en SIEX | — |
| IV | Glosario / acrónimos | — |
| **V** | **Definición de variables v3.11** — every REA and CUE field, flagged Obligatorio/Voluntario | XLSX |
| **VI** | **Interfaz Único Común v3.11.4** — the web service | DOCX |
| VII | Catálogos SIEX — standardised code lists | — |
| VIII | Modelo de autorización de representación | DOCX |
| IX | Modelo de autorización de acceso REA-CUE | DOCX |
| X | Registro de autorizaciones para el uso de aplicaciones comerciales | PDF |

> **Planning note:** FEGA states the separate agricultural and livestock technical
> documents will be replaced by a single *horizontal* technical document at **SIEX
> v5.9.0, September 2027**. Anything we build against v3.x needs a migration budget
> with that date on it.

Annexes V and VI are the two that matter; both are extracted into
[sources/](sources/).

## Who may write to a farmer's CUE

This is the governance model our "authorized intermediary" plan has to fit into.
Three roles:

### 1. Entidad habilitada
A natural or legal person acting **as the farm holder's representative**, authorised to
read that holder's REA and to complete/consult their CUE. Registered per region.
Requires signed authorisation from the farmer (FEGA Annex VIII is the model form;
regions such as Andalucía accept a *declaración responsable* asserting possession of
written authorisation, or an online form).

In practice these are cooperatives, agricultural advisory firms (*asesorías*), and
ATRIAs.

### 2. Empresa desarrolladora de software / CUE comercial
A software vendor whose product is entered on the region's authorised list of
commercial CUE tools (procedure in Annex X). Its certificate identifies the vendor,
not the farm.

### 3. Productor particular
An individual farmer authenticating directly, without a company certificate — the
`comunicacionProductor` path in the web service.

### The combination rule

From Annex VI §2.5.1, verbatim in substance:

- If the tool is **installed at the client**, the certificate may carry the domain and
  the *entity* identifier, so the server can check which farms are authorised.
- If the tool is a **centralised service** serving many entities, the certificate
  carries the domain and the *software company* identifier — and the notebook
  descriptor must then carry, as an additional field, the **entidad habilitada**
  responsible for the farms in that payload. **Only one entity's notebooks may be sent
  per call.**

So a SaaS intermediary is explicitly anticipated and explicitly constrained: our
payloads get partitioned by entidad habilitada.

## State of play

Regional adoption is uneven. Reporting from 2024 onwards notes that the registries of
software companies and enabled entities keep growing but are not at full coverage, and
that round-tripping — export to a commercial tool *and* upload back into the regional
CUE — had at that point only been achieved end-to-end in **Castilla y León**.

Treat per-region interface conformance as an empirical question to be tested, not a
given. See the phasing in [design/01-flujo-datos.md](../design/01-flujo-datos.md).
