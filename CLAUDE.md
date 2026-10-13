# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this repo is

Research and design notes for a digital layer connecting compost producers to farmers
in Spain, built on the *plan de abonado* obligation (RD 1051/2022) and the SIEX farm
data system (RD 1054/2022). It is Markdown only for now: there is no code, build, test
or lint step yet.

## Planned stack

Decided 2026-10-14, not yet scaffolded. The product is Spain-only. Rationale and
consequences are in `design/01-flujo-datos.md` §10; read it before scaffolding, and ask
before scaffolding.

| Concern | Choice |
|---|---|
| Language | TypeScript |
| Package manager | pnpm workspaces. Never npm or yarn |
| Monorepo | Turborepo |
| Backend | NestJS |
| Database | PostgreSQL with PostGIS |
| Data access | Prisma |
| Durable jobs | pg-boss |
| Web front end | React single-page app on Vite |
| Decimal arithmetic | decimal.js |
| Authentication | Better Auth, users in our PostgreSQL. Not Supabase Auth |
| Certificate identity and signing | AutoFirma signature flow, validated by the backend |
| Document storage | Scaleway Object Storage (Paris) over the S3 protocol, behind one storage interface |
| API style | REST; OpenAPI contract generated from NestJS; typed client generated from the contract |
| Client generator | Orval, output to `packages/api-client` |
| Front-end libraries | TanStack Router, TanStack Query, TanStack Table, MapLibre GL |
| Component kit and forms | shadcn/ui with Tailwind CSS; React Hook Form with Zod |
| Map data | No base map at first (later a Protomaps extract of Spain on Scaleway); CNIG municipal boundaries in PostGIS |
| Backend testing | Vitest, Testcontainers (real PostgreSQL with PostGIS), fast-check |
| Front-end testing | Vitest with React Testing Library; Playwright for end-to-end |
| Interface language | Spanish first; co-official languages by launch region |
| Translation library | react-i18next, with built-in `Intl` formatting |
| Boundary enforcement | eslint-plugin-boundaries |
| Code hosting | GitHub |
| CI and deployment | GitHub Actions for checks; Render deploys once they pass; Prisma migrations run before each release |
| Logging and errors | Pino; Sentry (EU-hosted) |
| Audit trail | Append-only PostgreSQL table, written in the same transaction as the change |
| Transactional email | Scaleway Transactional Email |
| Hosting, Phases 0–1 | Container on Render (web service plus background worker); Supabase for PostgreSQL; both in Frankfurt |

When code is added:

- Follow the workspace layout table in `design/01-flujo-datos.md` §10: `apps/api` with
  one NestJS module per bounded context, `apps/web`, and the framework-free packages
  `packages/dosing`, `packages/catalogue`, `packages/shared-kernel` and the
  generated `packages/api-client` (never edit it by hand). `iuws` and
  `identity` are technical modules, not contexts. Create a
  directory only when its phase starts. Do not extract another context into a package
  until it has a second consumer.
- Identifiers use that note's vocabulary. Check its "Avoid" columns before naming
  anything, and add a term there before introducing a new domain word in code.
- Never use JavaScript numbers for doses, percentages or coefficients: decimal.js in
  code, `numeric` in the database.
- The front end calls the backend only through the generated client. Do not
  hand-write API types or import from `apps/api`.
- Test database code against a real PostgreSQL through Testcontainers, not a mocked
  Prisma. Use Vitest everywhere; do not add Jest.
- No hard-coded user-facing text: every string goes through a translation key, with
  Spanish as the source language. Format numbers and dates with `Intl`, never by hand.
- Anything done under a `FarmLink` writes an audit row in the same transaction. Audit
  rows are never updated or deleted.
- Never log NIFs or document contents; log internal identifiers.
- Do not load the CNIG boundary dataset until its licence, version and download date
  are recorded in `research/05-fuentes.md`.
- Authentication establishes who a user is; only a `FarmLink` in the Representation
  module authorises acting for a farm. Do not put farm access in auth roles.
- Trusted certificate issuers and accepted certificate types must come from an
  official source. Do not hard-code a list from memory.
- Certificate key storage and Phase 2 hosting are deferred. Do not choose either
  without asking. Supabase is the database only: no Supabase Auth or Storage.
- Replace this section with the real build, test and lint commands.

## Layout

- `research/` — sourced facts about the regulation and the national schema. Numbered
  notes `01`–`05`, indexed by `research/README.md`.
- `research/sources/` — plain-text extracts of the official FEGA annexes (Anexo V
  sheets `CUE` and `REA`, Anexo VI). Large; grep them rather than reading them whole.
- `design/` — proposals built on the research. Numbered the same way. `01` is the data
  flow and implementation stack; `02` is the ubiquitous language.
- `README.md` — top-level overview and glossary.

## Rules for content

- **Keep research and design separate.** Claims about the law or the SIEX schema go in
  `research/` with a source. `design/` holds proposals and cites `research/` rather
  than restating sources.
- **Source every regulatory or schema claim.** Cite the article, annex section or field
  number, and add any new source to `research/05-fuentes.md`. The BOE consolidated text
  and the FEGA annexes are authoritative; the items under "Context and commentary" in
  `05-fuentes.md` are secondary and must be verified before relying on them.
- **The extracts in `research/sources/` are derived, not authoritative.** Use them to
  find things, then cite the original annex and version (currently Anexo V v3.11 and
  Anexo VI v3.11.4). Do not edit the extracts by hand.
- **Do not invent values.** Catalogue codes, agronomic coefficients (for example the
  N-availability factor), legal ceilings, regional endpoints and deadlines must come
  from a source. If one is missing, say so and record it under "Gaps" in
  `research/05-fuentes.md` or "Open questions" in the design doc.
- **Date what you find.** The existing research is dated 2026-09-23. The annexes are
  revised at least annually and the decrees have been amended several times, so record
  the consultation date and document version for anything new, and check whether a
  newer version exists before extending a note.

## Known gaps

Anexo VII (the SIEX catalogues) has not been obtained. It blocks anything that depends
on `Material Fertilizante` codes or on which composition fields are mandatory for
compost. The full list is at the end of `research/05-fuentes.md`.

## Conventions

- Prose in English. Spanish legal and technical terms stay in Spanish, italicised on
  first use (*plan de abonado*, *entidad habilitada*). Field and object names from the
  schema are kept verbatim in code formatting (`CodigoDGC`, `PlanAbonado`).
- Files are named `NN-slug-in-spanish.md`. When adding a research note, add it to the
  tables in `research/README.md` and the root `README.md`.
- Wrap lines at about 90 characters. Use tables for field lists and mappings.
- Dates are written `YYYY-MM-DD` in prose; `dd/MM/yyyy` appears only where quoting the
  CUE payload format.
