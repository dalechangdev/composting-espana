# 01 — Regulatory framework

## The decree you asked for

**Real Decreto 1051/2022, de 27 de diciembre, por el que se establecen normas para la
nutrición sostenible en los suelos agrarios.**

- Consolidated text (authoritative, kept current): https://www.boe.es/buscar/act.php?id=BOE-A-2022-23052
- Original publication (BOE, 29/12/2022): https://www.boe.es/buscar/doc.php?id=BOE-A-2022-23052

### Amendment history (as shown in the consolidated text header)

| Norm | Date | Link |
|---|---|---|
| Real Decreto 934/2025 | 22/10/2025 | https://www.boe.es/diario_boe/txt.php?id=BOE-A-2025-21211 |
| Real Decreto 840/2024 | 28/08/2024 | https://www.boe.es/diario_boe/txt.php?id=BOE-A-2024-17371 |
| Real Decreto 92/2024 | 24/01/2024 | — |
| Original text | 29/12/2022 | https://www.boe.es/buscar/doc.php?id=BOE-A-2022-23052 |

RD 934/2025 is the one that matters most for us: it rewrote the articles on the
*cuaderno de explotación*, the *plan de abonado*, and the use of **estiércoles y
abonos orgánicos** — i.e. precisely our subject matter — and set the current deadlines.

## What it obliges

### Art. 4.2 — the plan de abonado

The holder is responsible for *"la elaboración y aplicación de un plan de abonado en
cada unidad de producción"*.

**Deadlines (both already in force as of today, 2026-09-23):**

| From | Applies to |
|---|---|
| **1 January 2026** | Irrigated (*regadío*) production units where the crop is sown or planted between 1 March and 30 June |
| **1 September 2026** | Everything else |

**Exemptions** — production units that do *not* need a plan:
- (a) those used only for pasture that is not fertilized;
- (b) those not exceeding **10 ha**, if rainfed (*secano*) or dedicated only to
  pasture or forage crops for own consumption.

> Market-sizing note: exemption (b) is conditioned on *secano*. Irrigated units are in
> scope regardless of size. This is a much broader population than "farms over 10 ha".

### Art. 5 — the record-keeping obligation

Fertilization operations must be recorded in the *Cuaderno Digital de Explotación
Agrícola* (per RD 1054/2022) **within one month** of the operation, on paper or
digitally.

> This one-month window is the product trigger. Every compost delivery that gets
> spread creates a CUE entry someone has to write, correctly, within 30 days.

### Art. 6 — content of the plan de abonado

The plan must include:
- unambiguous identification of the parcels;
- soil data, including **organic matter content** and nutrient content;
- rainfall volume and available water resources;
- nutrient doses, application timing, **type of fertilizing material**, application
  method and machinery;
- mitigation measures for ammonia and greenhouse gas emissions.

**Art. 6.6 — mandatory technical advice (*asesoramiento*):** becomes obligatory one
year after the plan obligation in *zonas vulnerables*, two years elsewhere. So:
roughly **from 1 September 2027** for non-vulnerable zones, and already phasing in for
vulnerable ones. An advisor-facing product has a hard deadline attached to it.

## The sibling decree that builds the plumbing

**Real Decreto 1054/2022, de 27 de diciembre** — establishes SIEX (the national farm
information system), the REA (regional farm registry) and the CUE (digital field
notebook): https://www.boe.es/buscar/doc.php?id=BOE-A-2022-23054

Amended by **Real Decreto 34/2025, de 21 de enero**:
https://www.boe.es/buscar/doc.php?id=BOE-A-2025-998

Detail in [02-siex-rea-cue.md](02-siex-rea-cue.md).

## REGFER — the register a compost producer must be in

Separate from SIEX: the **Registro general de fabricantes y otros agentes económicos de
productos fertilizantes (REGFER)**.

- In force since **1 January 2026**; application window ran 17 March – 1 July 2026.
- Since **1 July 2026**, registration is a precondition to operate legally in the sector.
- Covers manufacturers, importers, distributors, **fertilizer application service
  companies**, and **fertilization advisors**.
- Administered regionally (Andalucía, Extremadura, Canarias etc. each run their own
  registration procedure).

Two consequences for this project:
1. A compost producer placing fertilizing material on the market is a REGFER-registered
   economic operator — that registration is an identity we can key off.
2. The CUE fertilization record has a **`Código REGFER`** field for the service company,
   marked *Obligatorio* "a partir de su entrada en vigor en 2026" (Anexo V, field 476).
   Anyone spreading compost as a service now has to be identified by it in the notebook.

## Regulatory reading of the opportunity

The decrees create three distinct obligations that all touch a compost transaction:

1. The farmer must **plan** N/P/K needs before applying (Art. 6).
2. The farmer must **record** what was applied, at what dose, from which supplier,
   within a month (Art. 5).
3. The compost producer must be a **registered operator** (REGFER) and, where the
   material is waste-derived, carry a NIMA waste-operator code that the CUE record
   demands.

Today each of these is satisfied by a different actor with a different piece of paper.
Nothing connects them.
