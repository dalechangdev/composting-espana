# 04 — The CUE fertilization data model

Sources: FEGA **Anexo V v3.11** (field definitions, with Obligatorio/Voluntario flags)
and **Anexo VI v3.11.4** (JSON descriptor + worked examples). Extracts in
[sources/](sources/).

Everything below is the *existing* national schema. It is worth reading closely,
because it already contains the hooks a compost marketplace needs.

## `Fertilizacion` — one record per application

```json
{
  "IdAjenaFerti": 1000,
  "Borrar": false,
  "FechaInicio": "02/11/2023",
  "FechaFin": "03/11/2023",
  "DeclaraBuenasPracticas": true,
  "GestionsostInsu": "34",
  "BuenasPracticas": [ { "TipoBPF": 2 }, { "TipoBPF": 4 } ],
  "MaterialFertilizante": {
    "Material": 23,
    "DetalleMaterial": 1376,
    "EmpresaSuministradora": "ppsl",
    "NifEmpresa": "nif-0001",
    "REGA": "3",
    "NIMA": "3",
    "TratamientoEstiercoles": 2,
    "Macronutrientes":  [ { "TipoMacroN": 2, "Porcentaje": 23 } ],
    "MetalesPesados":   [ { "TipoMetalP": 3, "Porcentaje": 3  } ],
    "Micronutrientes":  [ { "TipoMicroN": 3, "Porcentaje": 23 } ],
    "Densidad": 3,
    "UnidadesMedida": 23
  },
  "AplicacionMaterialFertilizante": {
    "NombreProducto": "prd limpia",
    "AplicacionLodos": true,
    "TipoFertilizacion": 1,
    "MetodoFertilizacion": 1,
    "Dosis": 3.4,
    "Unidad": 3,
    "EmpresaServicios": "empr-0001"
  },
  "EquipoAplicador": {
    "EquipoPropio": true,
    "IdEquipoAplicador": "equp-apl-ferti-0010",
    "TipoMaquinariaUNE": 1,
    "TipoEnergia": 3,
    "NumRepeticiones": 3,
    "Duracion": 3
  },
  "Fertirrigacion": { "SistemaRiego": 1, "Cantidad": 1, "Unidad": 2, "OrigenAgua": [...],
                      "NumContador": "3", "DosisN": 4, "UnidadDosisN": 3,
                      "DosisP": 3, "UnidadDosisP": 3, "TipoEnergia": 1 },
  "DGCs": [ { "CodigoDGC": 464351, "Superficie": 0.13 } ]
}
```

### The supplier-identity fields — this is the opening

Anexo V, fields 458–462:

| # | Field | Obligation | Definition |
|---|---|---|---|
| 458 | **Empresa suministradora** | Obligatorio | Name of the company supplying the **manures or wastes** applied as fertilizing material |
| 459 | **NIF** (if *centro gestor*) | Obligatorio | NIF if a managing centre. **Mutually exclusive with REGA/NIMA** |
| 460 | **REGA** | Obligatorio | REGA code of the manure-supplying company. Mutually exclusive with NIF/NIMA |
| 461 | **NIMA** | Obligatorio | NIMA code where **wastes are applied as fertilizing material**. Mutually exclusive with NIF/REGA |
| 462 | Tratamiento de estiércoles | Voluntario | Treatment the applied manures have received, per SIEX catalogue |

A composting plant is a *centro gestor* of waste with a **NIMA**. The national schema
therefore already requires the farmer to name our customer — by regulated identifier —
every time compost is spread. Today that identifier is typed in by hand, from a
delivery note, weeks later, by someone who may not have it. That is the gap.

### Composition fields

`Macronutrientes`, `Micronutrientes`, `MetalesPesados` are all `%` by weight, plus
`Densidad` for liquids. Anexo V marks them *Voluntario* in general but adds: the fields
that are **obligatory depend on the material**, per the "Campos a registrar" column of
the **Material Fertilizante catalogue** (Anexo VII). Heavy metals are called out as
applying to *lodos*.

> A compost producer already runs these analyses for its own product compliance. The
> farmer needs the same numbers for the notebook. Nobody is moving them between the two.
> This is a copy-paste problem dressed up as a regulatory one.

### Dose and application

| Field | Obligation | Valid units |
|---|---|---|
| Dosis de fertilizante (469) | Obligatorio | — |
| Unidad (470) | Obligatorio | `kg/ha`, `L/ha`, `m3/ha`, `t/ha` |
| Tipo de fertilización (471) | Obligatorio | SIEX catalogue |
| Método de aplicación (472) | Obligatorio | SIEX catalogue |
| Documento de aplicación de lodos (473) | Obligatorio | boolean |
| **Código REGFER (476)** | **Obligatorio** | "empresa de servicios, a partir de su entrada en vigor en 2026" |

### Good-practice escape hatch

Anexo VI change log, v3.11: *"Los bloques Aplicación material fertilizante y Material
fertilizante pierden su obligatoriedad indicando que lo son siempre que no se indique
una buena práctica de abonado en verde."* And Anexo V field 455: if *Abonado en verde*
is declared, the remaining fertilization fields are left empty. Worth handling — it is a
legitimate branch, not an error.

## `PlanAbonado` — the fertilization plan itself

```json
{
  "IdAjenaPlan": 1000,
  "Borrar": false,
  "NecesidadUFN": 34.3,
  "NecesidadUFP2O5": 12,
  "NecesidadUFK2O": 4,
  "ObjetivoProduccion": 2000,
  "CultivoPrecedente": 121,
  "FechaAsesoramiento": "01/09/2024",
  "FechaGeneracion": "01/09/2024",
  "Herramienta": true,
  "DGCs": [ { "CodigoDGC": 2831790 }, { "CodigoDGCAjena": 1000 } ]
}
```

All of `NecesidadUFN`, `NecesidadUFP2O5`, `NecesidadUFK2O`, `ObjetivoProduccion`,
`CultivoPrecedente`, `Herramienta` and the plan date are **Obligatorio** (Anexo V fields
492–498). `Herramienta` is the declaration that a digital sustainable-nutrition advisory
tool was used — a flag the regulator put there specifically to track digital adoption.

> This block is a **demand signal in kg/ha of N, P₂O₅ and K₂O, per parcel, with a
> production target and a date — filed by law, ahead of the season.** It is the most
> valuable object in the whole schema for a compost producer, and it is machine-readable.

## `Analitica` — soil parameters

```json
"ParametrosSuelo": {
  "MateriaOrganica": 12.3, "Arena": 12.3, "Limo": 12.3, "Arcilla": 12.3,
  "Ph": 12.3, "FosforoAsimilable": 12.3, "PotasioAsimilable": 12.3,
  "NitrogenoTotal": 12.3, "Conductividad": 12.3
}
```

With `MaterialAnalizado`, `Fecha`, `Laboratorio`, `Nif`, `NumBoletin`, `TiposAnalisis`,
`TiposSustancias`, and DGC references.

`MateriaOrganica` tracked per parcel over time is the evidence base for compost's actual
agronomic claim. Art. 6 of RD 1051/2022 requires soil organic matter data in the plan
anyway, so the farmer is obliged to have it. A producer who can show SOM trending up on
parcels they supplied has a defensible sales argument built from the customer's own
regulatory filings.

## Other activity blocks in the same descriptor

`ActividadesExplotacion` also carries `TratamientosFitosanitarios`, `ComercializacionVD`
(sales), `Pastoreo` (grazing, incl. third-party animals), `CubiertaVegetal`, `Riego`,
`Siembra/Plantación`, `Cosecha`. We only write to `Fertilizacion`, `PlanAbonado` and
possibly `Analitica` — but we must not clobber the rest, which is what the per-record
`IdAjena*` + `Borrar` design is for.

## Catalogue dependency

`Material`, `DetalleMaterial`, `TipoFertilizacion`, `MetodoFertilizacion`, `Unidad`,
`TipoBPF`, `TratamientoEstiercoles`, `CultivoPrecedente`, `TipoMacroN`, `TipoMicroN`,
`TipoMetalP`, `TipoMaquinariaUNE`, `TipoEnergia`, `SistemaRiego`, `OrigenAgua` are all
**SIEX catalogue codes** (Anexo VII), versioned with the technical document and updated
at least annually. Several of the rejection messages in Anexo VI are catalogue
violations. Catalogue sync is a first-class subsystem, not a constants file.
