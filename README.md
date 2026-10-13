# compost-espana

Research and design for a digital layer connecting compost producers to farmers in
Spain, built on top of the *plan de abonado* obligation and the SIEX farm data system.

**Status:** research and design only — there is no code in this repository yet.

**Planned stack:** TypeScript in a pnpm workspace monorepo built with Turborepo; a
NestJS backend using Prisma on PostgreSQL with PostGIS, pg-boss for durable jobs and
decimal.js for dose arithmetic; a React front end built with Vite. See
[design/01-flujo-datos.md](design/01-flujo-datos.md#10-implementation-stack).

## The idea

Spain's [RD 1051/2022](https://www.boe.es/buscar/act.php?id=BOE-A-2022-23052) requires
farms to hold a *plan de abonado* (fertilization plan) and to log every fertilizer
application in a digital field notebook (CUE) within one month.
[RD 1054/2022](https://www.boe.es/buscar/doc.php?id=BOE-A-2022-23054) builds the state
system that receives this data (SIEX) and defines who may write to it on a farmer's
behalf. FEGA publishes a REST API, the *Interfaz Único Común* (IUWS), for that purpose.

The CUE fertilization record already carries fields for the supplier of waste-derived
fertilizing material, including its NIMA waste-operator code, and the plan de abonado
record already carries each crop's N/P/K requirement. The compost market's matching
data is already being filed by law, just not connected. This project is about building
that connection.

## Contents

| Path | Contents |
|---|---|
| [research/](research/) | Background research, dated 2026-09-23. Start with [research/README.md](research/README.md) |
| [research/01-marco-normativo.md](research/01-marco-normativo.md) | The royal decrees: what they require, of whom, from when |
| [research/02-siex-rea-cue.md](research/02-siex-rea-cue.md) | SIEX architecture, REA, CUE, *entidades habilitadas*, commercial CUE providers |
| [research/03-interfaz-unico-ws.md](research/03-interfaz-unico-ws.md) | The IUWS web service: endpoints, auth, request flow |
| [research/04-modelo-datos-fertilizacion.md](research/04-modelo-datos-fertilizacion.md) | CUE data model for fertilization, plan de abonado and soil analytics |
| [research/05-fuentes.md](research/05-fuentes.md) | Every source consulted, with links |
| [research/sources/](research/sources/) | Text extracted from the official FEGA annexes (Anexo V, Anexo VI) |
| [design/01-flujo-datos.md](design/01-flujo-datos.md) | Design sketch: actors, canonical objects, data flows, field mapping, components, phasing, implementation stack |
| [design/02-lenguaje-ubicuo.md](design/02-lenguaje-ubicuo.md) | Ubiquitous language: bounded contexts, glossary per context, invariants, resolved ambiguities |

Claims about the national schema are sourced in `research/`; everything in `design/`
is proposal.

## Glossary

| Term | Meaning |
|---|---|
| **SIEX** | National information system for agricultural and livestock holdings |
| **REA** | Per-region farm registry: holder, parcels, crops. The master data |
| **CUE** | Per-farm digital field notebook: the activity log |
| **IUWS** | *Interfaz Único Común*, the REST API in front of REA and CUE |
| **FEGA** | The national agency that publishes the SIEX technical specification |
| **Plan de abonado** | Fertilization plan required per production unit |
| **Entidad habilitada** | Entity authorised to read and write REA/CUE on a farmer's behalf |
| **NIMA** | Waste-operator identifier; how a compost producer is named in a CUE record |
| **DGC** | Parcel identifier issued in the REA export; required for any CUE write |

SIEX is national, but REA and CUE are operated separately by each of the 17
Comunidades Autónomas behind one common interface spec.

## Proposed phasing

1. **Phase 0, read-only.** A partner entidad habilitada shares REA and plan de abonado
   exports; build a regional nutrient-demand surface for compost producers.
2. **Phase 1, assisted write.** Compose and validate fertilization payloads; the
   partner submits them.
3. **Phase 2, commercial CUE.** Pursue our own authorisation, region by region.
4. **Phase 3, advisory.** Become the digital advisory tool recorded in the plan de
   abonado.

Details and rationale are in [design/01-flujo-datos.md](design/01-flujo-datos.md#8-phasing).

## Open questions

The main blocker is the **Anexo VII catalogues**, which are needed before the product
passport validator or the dose engine can be written. The full list, including regional
N-availability coefficients, the IUWS sandbox, authorisation cost and liability for
composed CUE entries, is in
[design/01-flujo-datos.md](design/01-flujo-datos.md#9-open-questions).
