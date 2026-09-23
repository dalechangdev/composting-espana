# 03 — Interfaz Único Común (IUWS): the actual API

Source: FEGA **Anexo VI, Interfaz Único Común, v3.11.4**, published 19/11/2025.
Extracted text: [sources/anexoVI.txt](sources/anexoVI.txt).

## Shape

REST over HTTPS, JSON payloads. Base path `/IUWS/api/`. **Each Comunidad Autónoma hosts
its own deployment** — the spec writes endpoints as `https://servidor:puerto/IUWS/api/...`
because there is no single host. We will maintain a per-region endpoint registry.

## Security model

Designed to meet **ENS (Esquema Nacional de Seguridad) nivel medio**. Two factors:

1. **Qualified certificate** — a *persona jurídica* certificate (type **OV**, see
   Annex 3 of the spec) issued by a *Prestador Cualificado de Servicios de Confianza*,
   installed on the calling application server. The server validates that the
   certificate's **domain is registered/authorised** in the system.
   - Note: the requirement to include the `O` field in the certificate was **removed**
     (v2.0.1); Annex 4 of the spec covers how communication works without it.
2. **JWT session token** — obtained from the `comunicacion` method, then sent on every
   subsequent call. Once a JWT is held, neither the certificate nor the earlier custom
   headers need be resent; TLS plus JWT claims carry authentication.

For *productores particulares* no certificate is required on the `comunicacionProductor`
call; the method delegates to a validation app deployed per region, and the resulting
JWT carries the producer's NIF.

## Methods

| Purpose | Endpoint | Notes |
|---|---|---|
| Authenticate (entity / commercial tool) | `/IUWS/api/comunicacion/{nombreAppCue}` | Returns JWT. `nombreAppCue` = registered name of the commercial CUE |
| Authenticate (individual producer) | `/IUWS/api/comunicacionProductor/{nombreAppCue}/{peticion}` | Returns JWT containing the producer's NIF; no certificate |
| Export REA (farm master data) | `/IUWS/api/exportarREA/{NIF titular}_{CIF Gestora}` | e.g. `.../exportarREA/53442255P_V83192450` |
| Export CUE (notebook) | `/IUWS/api/exportar/{Nº explotación}_{CIF Gestora}` | e.g. `.../exportar/ES130000008847_V83192450` |
| Import activities into CUE | `/IUWS/api/crear/` | **POST**. Also performs deletion via a `Borrar` flag per record |
| Check import status | `/IUWS/api/comprobarEstado/{Id Petición}` | Import is **asynchronous** |
| Interface version | `/IUWS/api/version` | |

### Import is asynchronous

`POST /crear/` responds with *"El fichero se ha recibido correctamente para ser
procesado"* plus an **Id Petición** and an **IdCuaderno**. Validation results arrive
later via `comprobarEstado`, as a per-activity list of messages. Real examples from the
spec:

> "No se importa la actividad Fertilización porque para el código Rea: ES130888008847,
> con fecha de tratamiento: 19/12/2023, la unidad: 5 no está dentro de las unidades
> disponibles"

> "No se importa la actividad Fertilización porque para el código Rea: ES130888008847
> con fecha de inicio: 19/12/2023, la referencia al recinto: 341806 no es válida"

Implication: **partial success is the normal case.** Some activities in a batch import,
others are rejected on catalogue or parcel-reference grounds. Any client needs a
reconciliation loop, not a fire-and-forget POST.

Error classes seen in the spec include `Certificado no válido, no cumple con las
exigencias` and `Dominio del certificado no registrado en el sistema`.

## Parcel identity: `CodigoDGC`

A DGC is the graphical crop declaration unit. Originally activities referenced parcels
by full SIGPAC coordinates (Provincia, Municipio, Agregado, Zona, Polígono, Parcela,
Recinto). As of interface v1.4.4 those seven fields were **removed and replaced by a
single `CodigoDGC`**, explicitly *"con el objeto de evitar posibles errores de
identificación"*.

`CodigoDGC` is **obtained from the REA export**. There is also `CodigoDGCAjena` for
referencing a DGC by the caller's own identifier.

> This forces the integration order: you cannot write a fertilization activity until you
> have exported that farm's REA and learned its DGC codes. REA export is a hard
> prerequisite, not an optimisation.

## Idempotency

Each activity block carries an `IdAjena*` field (`IdAjenaFerti`, `IdAjenaPlan`,
`IdAjenaVenta`, `IdAjenaAna`, `IdAjenaPastoreo`…) — the *caller's* identifier for the
record — plus a `Borrar` boolean. This is the correlation key for reconciliation and the
mechanism for retraction. Our system must mint and persist these.

## Multi-tenancy constraint

Restated from Annex VI §2.5.1 because it shapes our batching: a centralised commercial
tool must declare the **entidad habilitada** in the notebook descriptor, and **only
notebooks belonging to a single entity may be sent per call**. Batch boundaries are
therefore `(region, entidad habilitada)`, not `(region)`.
