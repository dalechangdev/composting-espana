# Research — compost ↔ SIEX digital layer (Spain)

Background research for a digital layer connecting compost producers to farmers,
built around the Spanish *plan de abonado* obligation and the SIEX farm data system.

Research date: **2026-09-23**. All regulatory deadlines below have already passed —
this is a live compliance market, not a future one.

| File | Contents |
|---|---|
| [01-marco-normativo.md](01-marco-normativo.md) | The royal decrees: what they require, of whom, from when |
| [02-siex-rea-cue.md](02-siex-rea-cue.md) | SIEX architecture, REA, CUE, *entidades habilitadas*, commercial CUE providers |
| [03-interfaz-unico-ws.md](03-interfaz-unico-ws.md) | The Interfaz Único Común web service: endpoints, auth, request flow |
| [04-modelo-datos-fertilizacion.md](04-modelo-datos-fertilizacion.md) | Exact CUE data model for fertilization, plan de abonado and soil analytics |
| [05-fuentes.md](05-fuentes.md) | Every source consulted, with links |
| [sources/](sources/) | Text extracted from the official FEGA annexes (see 05-fuentes.md for provenance) |

## The one-paragraph version

Spain's [RD 1051/2022](https://www.boe.es/buscar/act.php?id=BOE-A-2022-23052) requires
farms to hold a *plan de abonado* (fertilization plan) and to log every fertilizer
application in a digital field notebook (CUE) within one month. A sibling decree,
[RD 1054/2022](https://www.boe.es/buscar/doc.php?id=BOE-A-2022-23054), builds the
state system that receives this data (SIEX) and defines who may write to it on a
farmer's behalf: *entidades habilitadas* using authorised commercial software. FEGA
publishes a real REST API (the *Interfaz Único Común*) for exactly that. The CUE
fertilization record already carries fields for the supplier of manure/waste-derived
fertilizing material — including its NIMA waste-operator code — and the plan de
abonado record already carries the crop's N/P/K requirement. Those two facts are the
opening for a compost-side product: the data model has a compost-shaped hole in it.
