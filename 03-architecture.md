# Architecture

Scope: contracts and boundaries. Internal structure is delegated — see `06`.

## Context

One Hub on the site LAN, served to tablets over HTTP. A capture tool on the control room network is reached by a file and answers over HTTP. The acceptance portal is reached by a person, not by the system. Nothing crosses the internet.

```
  tablet browser (Chrome/Safari, no install)
        |  HTTP, session cookie, 22 routes
        v
  +-----------------------------+          SMB share           +------------------+
  |  Hub  (Node 22, Windows)    |  ---- trigger file ---->     |  capture tool    |
  |  - web app + API            |  <--- POST PNG over HTTP --- |  (WinCC screen)  |
  |  - record store (SQLite)    |                              +------------------+
  |  - evidence store (files)   |
  |  - compile + bundle         |
  +-----------------------------+
        |
        |  one zip, downloaded by a person
        v
  ACC portal  <--- uploaded by hand ---  signing engineer
```

Scope is one site, one SAT. The diagram carries no internet path and no client name.

## Components

| Component | Responsibility | Owns data | New or existing |
|---|---|---|---|
| Hub web service | Serves the browser app and the 22 API routes; enforces the one capability check | none directly | new |
| Record store | Steps, evidence metadata, forms, editions, amendments, audit, jobs | `steps`, `evidence`, `evidence_derivative`, `form`, `form_schema`, `edition`, `amendment`, `audit`, `user`, `identity`, `session`, `service_principal`, `screenshot_request`, `daily_report`, `compile_job` | new |
| Evidence store | Bytes named by their SHA-256, plus derived document copies | `evidence/<sha256>`, `derived/<sha256>` | new |
| Repository module | The only place SQL exists; runs the claim, seal and amendment transitions in a transaction | none | new |
| Template engine | Renders one workbook from a pinned schema version and the step list | none | new |
| Bundle builder | Assembles the zip from `bundle.layout.json` and writes the manifest | the bundle and `bundle_sha` | new |
| Browser client | Grid, step screen, camera hand-off, capture spool, screenshot polling | the spool in IndexedDB only, never the record | new |
| Capture tool | Existing WinCC screen capture; gains a request id and an HTTP post | its own local image folder | existing, changed |
| Setup command | Seeds accounts, issues the tool token, loads the schema and the instrument list | initial rows | new |
| Instrument-list importer | Reads the project's instrument or IO list from a spreadsheet and creates one step per row through the setup path | `steps` | new |

## External contracts

| System | Direction | Operation and payload shape | Auth | Failure mode | Limits |
|---|---|---|---|---|---|
| Tablet browser | in | 22 routes; JSON bodies; evidence as multipart; one error shape `{"error":{"code","message","details"}}` | session cookie, `HttpOnly`, `SameSite=Lax`, sliding 12 h | 401 on expired session; one error shape on every failure | 3000 px long edge, 8 MB after client encode |
| Device camera | in | `<input type="file" capture>`; no video stream, no permission API | device OS | user cancels; no file returned | one photo per invocation |
| SMB capture share | out | one trigger file per request, carrying `request_id` | filesystem ACL on the share | `500 trigger_unwritable`, no request row created | one file per request; 90 s expiry |
| Capture tool | in | `POST /evidence/screenshot` with `request_id`, `tool`, `tool_version`, PNG bytes | service principal token in `Authorization` | 401 bad token, 404 unknown request, 409 expired, 409 already served | 10 MB PNG |
| ACC portal | out | a person uploads one zip; no API call from the system | client-side credentials, outside this system | a person sees the rejection and corrects by hand | not known, and not on the critical path |
| Control room PC | not called | the Hub never reads images from the share | — | a stale trigger file stays traceable by name | — |

The trigger goes one way through the share because that is what a network owner will allow. The image comes back over HTTP because that is the leg where a confirmation, a timestamp and an error are worth having. Both legs then take the same evidence path.

## Data

| Entity | Owner | Key fields | Retention | Personal data |
|---|---|---|---|---|
| `user` | Hub | id, username, display name, role, password hash, active | life of the dataset (R61) | yes — a person's name |
| `identity` | Hub | provider, subject, user_id | life of the dataset | yes |
| `session` | Hub | id, user_id, device_id, created_at, expires_at | 12 h, then dead on restart | yes, weakly |
| `service_principal` | Hub | id, label, token hash, active | life of the dataset | no |
| `form_schema` | Hub | id, form_id, version, payload (columns, sections, slots, layout) | immutable once pinned | no |
| `form` | Hub | id, sat_id, level, title, pinned schema_id | life of the dataset | no |
| `steps` | Hub | id, form_id, step_no, tag, status, revision, values_json, lease, device_id, client_seq | life of the dataset | no |
| `evidence` | Hub | id, step_id, slot_id, kind, sha256, bytes, mime, dimensions, captured_at, revision | append-only; never deleted | possibly — a site photo may show a person |
| `evidence_derivative` | Hub | evidence_id, purpose, sha256, bytes, dimensions | regenerable | same as its original |
| `screenshot_request` | Hub | id, step_id, tag, expires_at, served_evidence_id | life of the dataset | no |
| `edition` | Hub | id, form_id, number, schema_id, content_sha, sealed_at, sealed_by, superseded | never deleted, never edited | yes — who sealed |
| `amendment` | Hub | id, form_id, edition_id, reason, step_ids, opened_at, closed_at | life of the dataset | yes — who opened |
| `audit` | Hub | at (server time), actor, device_id, entity, action, detail_json | life of the dataset | yes — who did it |
| `daily_report` | Hub | id, day, sat_id, body | life of the dataset | no |
| `compile_job` | Hub | id, sat_id, idem_key, state, bundle_path, bundle_sha | life of the dataset | no |

Bytes never enter these tables (R49). The step values are JSON keyed by schema column id, on purpose: a column per form would make a new schema a migration, and the generality claim is that a new form is a schema row.

**Retention position.** v1 keeps everything, including superseded editions and evidence for replaced photos. Who takes the site dataset away and what is destroyed is the client's later decision (deferred register item 11). Nothing in this build deletes.

## Auth

Three identities matter: the engineer account, the device, and the service principal.

- The engineer signs in against a local password provider (`argon2`) and receives a server-side session. The session row means a Hub restart logs everyone out, which is planned for rather than discovered during a multi-day test.
- `device_id` is a browser-generated UUID in `localStorage`, sent on every write. `client_seq` is that device's counter. Together they make a v2 replay possible without a schema change.
- The capture tool holds a service principal token, issued once by the setup command. The token is stored hashed, checked only on the tool route, and never logged.
- One role, one capability map, one `requireCapability(user, cap)` call per route. No handler reads a role directly (R2).
- The signer does not sign in. The signer signs in ACC or on paper, outside the app (D20).

`identity(provider, subject, user_id)` exists so a person can hold both a local and an Entra identity later. No v1 code path depends on that.

## NFR numbers

| Property | Target | Source |
|---|---|---|
| Grid read, 100 steps | under 200 ms p95 on the site laptop | R58, build plan B1 |
| Concurrent devices | 3, up to 3 engineers, one lease holder per step | R59, scope interview Q9 |
| Image caps | 3000 px long edge, 8 MB after encode, 20 MB pre-decode reject, 10 MB tool PNG | R60, interface spec |
| Lease length | 12 hours | decision D8 |
| Screenshot expiry | 90 seconds, poll every 2 seconds | interface spec |
| Session length | 12 hours sliding, invalidated by restart | R3 |
| Availability target | none stated. Single process; a restart is a planned interruption and logs everyone out | decision D18 |
| Retention | keep everything in v1 | R61 |
| Internet at run time | none required | R62, decision D5 |

## Environments

| Environment | Purpose | Data | Who deploys |
|---|---|---|---|
| Developer machine | Build and unit tests | fixtures | each developer |
| Rehearsal target | The full flow against a WinCC test target or a recorder, never the live control room PC | fixtures and sample forms | platform, with the developer who owns the tool change |
| Site machine (Hub) | The pilot itself, fixed LAN address | real site data only, never removed by us | platform |
| Tablet | The tested client device; phone is best effort | the spool only | the engineer on the day |

There is no staging environment and no cloud environment in this delivery. Both are v2.

## Pinned decisions

Pinned means expensive to reverse. Every row cites a source or it would be an assumption.

| Decision | Source |
|---|---|
| A standalone product outside Project SCALE; no SSM identifiers, no SCALE component | architecture README D1, 25 November 2026 |
| The Hub keeps a clean API edge: record store, evidence store, and a read API in front of both. No versions, lineage, permissions or publication status | architecture README D2 |
| The workbook is generated from the master record using `exceljs` | architecture README D4; alignment 2026-11-26 |
| The SAT format is a single standard, defined once and versioned. The instrument list supplies steps and tags; layout comes from the master record | alignment 2026-11-26, A11 |
| Ingestion proposes and a person commits. No candidate writes to the record, calls the API, or creates a form | build plan F3 and F4; R64 |
| Web-first: the client is a browser app served by the Hub. The tablet installs nothing and holds no local database | architecture README D5 |
| Per-engineer accounts issued by YNY; a normal authenticated session; no directory service; the signer does not sign in | architecture README D6; decision D17 |
| Screenshot path is two legs: trigger file out through the share, PNG back over HTTP to the Hub | architecture README D7; interface spec, Screenshot |
| The step claim is decided online by the Hub with a 12 hour lease; no offline claim in v1 | architecture README D8 |
| The browser keeps an IndexedDB spool for in-flight capture only; it is never the record | architecture README D10 |
| Evidence is content addressed: the file name is the SHA-256 of the bytes | architecture README D11 |
| Bytes are files, never BLOBs in the database | architecture README D12 |
| A form is immutable once sealed; corrections are amendments producing the next edition of that form; a compile never seals | architecture README D13; interface spec, seven rules |
| The master record is the form schema, per project, versioned, immutable once any form pins it | architecture README D16; interface spec, Configuration shape |
| SQLite behind a repository module; no SQL in route handlers; WAL and a busy timeout | architecture README D18; build plan A3, A4 |
| The capture tool is silent: it posts the PNG and nothing else; request state is derived | architecture README D19 |
| v1 ends at a sealed bundle; no e-signature and no in-app acceptance record | architecture README D20; scope interview Q8 |
| One bundle per SAT, containing every form and its evidence, with a manifest that flags incomplete steps | scope interview Q15 and Q16 |
| Compile runs on the Hub on site, with no internet, using a local xlsx writer | scope interview Q18 |
| The step list is project data loaded into `steps` at setup, not schema | interface spec, Glossary and Open items |
| One role, one capability map, no RBAC, one SAT per Hub, so form routes carry no SAT id | scope interview Q9; assumption A6 |
| Runtime has no AI, no OCR and no model call; "AI application" is not the title of the deliverable | scope interview Q11 and Q12 |

## Delegated

Module and file layout, internal interfaces, test strategy, CI wiring, logging, client component structure, sheet geometry constants and user-facing copy are handed to the receiving team. See `06` for the rows, their deciders and their phases. They are not restated here.
