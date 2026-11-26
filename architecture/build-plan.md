# SAT PoC v1 build plan

> Reference annex to the plan pack. The plan (`../00-index.md`) is the single source of truth: the work-package acceptance tests in `05` condense these task tests, and on any difference the plan applies.

Status: draft, 25 November 2026. Written from the [interface spec](architecture-v1-interface-spec.md) and the [architecture package](architecture/README.md). No effort estimates and no dates: this fixes **what is built, in what order, and how each item is known to be done**.

Written for two developers splitting the build, and written so a coding agent can take one task without inventing requirements.

## How to use this plan

- **A task is done when its "done when" test passes**, not when the code looks finished. Where a check needs a human eye, it says so.
- **If a task looks ambiguous, stop and raise it.** Do not pick an interpretation and keep going. Any new route, table or field needs a spec change first, because the spec is what the second developer is coding against.
- **Every agent-authored change needs a named human reviewer** before merge. Name the reviewer on the task, not after.
- **No task may add scope.** The deferred list is in [decisions-and-open-items.md](decisions-and-open-items.md); if a task seems to need something from that list, the plan is wrong, not the list.

## Preconditions

These are inputs from other people. A task that needs one is marked **blocked**. Nothing else waits on them.

| Input | Owner | If it is late |
|---|---|---|
| Wi-Fi client isolation confirmed in writing | Site network owner | Nothing can be tested on site. Design does not change; the pilot cannot run. |
| Capture tool change: date and a test target that is not the live control PC | Jien Ming | Screenshot integration can be built and tested against a stub; the real leg stays unproven. |
| ACC bundle tree and file naming confirmed | Whoever uploads to ACC | Build against `bundle.layout.json` and the spec's provisional tree; the change is a config edit. |
| Step-list loading decision | Deferred | A1 accepts a hand-made import fixture; the loader is a separate task when decided. |
| Hub machine and fixed LAN address | YNY | Blocks the rehearsal, not the build. |

## Global definition of done

A task is not complete unless, in addition to its own test:

1. It has a test that fails before the change and passes after.
2. No SQL outside `repo/`, no magic number outside `config/`, no error response outside the one error shape.
3. The audit trail row it should write is asserted by a test.
4. It works with the internet disconnected and the site LAN still up, except where the task is explicitly about the tool's network call.

## Workstreams and split

Two developers, one seam: the spec.

- **Hub developer:** A-series, B-series, D3–D4, E1–E2.
- **Client developer:** C5–C6, plus C1–C2 shared with the Hub side.
- **Either, once A1–A3 exist:** D1–D2, the template engine, which is the longest single piece.
- **F-series (optional) is unassigned.** It is not part of the December deliverable and nobody should start it without a promotion decision.

Build order matters only where a dependency is named. Within a series, tasks may run in parallel.

## A. Foundation

**A1. Repository and build**
Create the repo layout: `server/`, `client/`, `shared/`, `repo/`, `config/`, `migrations/`, `test/`. Node 22 LTS, TypeScript, Fastify, one build script, one test runner, one lint config.
*Done when:* a clean copy of the folder builds and passes tests with no network access. The build output runs with `node dist/server.js`.

**A2. Configuration constants**
All tunables in one module: lease 12 h, session 12 h, screenshot expiry 90 s, poll interval 2 s, image caps 3000 px / 8 MB / 20 MB / 10 MB, Hub-to-tool timeout 5 s, queue depth 100, tool host and port, paths for evidence, derived, data, backups.
*Done when:* a test asserts that no numeric literal above 60 appears in a route or repository file, and that changing the cap in config changes the enforced limit.

**A3. Schema and migrations**
Every table in the spec: `user`, `identity`, `session`, `service_principal`, `form_schema`, `form`, `steps`, `evidence`, `evidence_derivative`, `screenshot_request`, `daily_report`, `audit`, `edition`, `amendment`, `compile_job`. SQLite with WAL and a busy timeout set at open.
*Done when:* an empty database migrates to full schema; running migrations twice is a no-op; a second database created from scratch has an identical schema dump.

**A4. Repository module**
One module per aggregate: `steps.repo`, `evidence.repo`, `forms.repo`, `auth.repo`, `jobs.repo`. All SQL lives here. Claim, seal and amendment transitions run inside a transaction.
*Done when:* the grep test from A2 also fails on SQL keywords outside `repo/`, and a test proves two concurrent claims on one step produce one grant and one `409`.

**A5. Setup command**
Seeds two or three engineer accounts, generates the capture tool's service token, loads a form schema and its step list from a fixture, and creates the `form` rows with a pinned schema id.
*Done when:* running it on an empty database produces a SAT that the grid can read; it prints the tool token once and never again; it refuses to run a second time without `--force`; a wrong-format fixture is rejected with the offending row named.

**A6. Auth, sessions and capabilities**
`LocalPasswordProvider` behind the provider interface, `argon2` hashing, cookie session with a server-side row, `requireCapability(user, cap)` used by every route, a static role→capability map with one role.
*Done when:* no route handler reads a role directly; an expired session returns `401`; a restart invalidates all sessions; a removed identity stops logging in.

**A7. Offline install and service**
A self-contained folder: Node runtime, installed `node_modules`, built client, migrations, fixtures, `install.ps1`, service wrapper (NSSM or equivalent), log rotation, fixed port.
*Done when:* copying the folder to a clean Windows machine with no internet and running `install.ps1` gives a working `/me` over HTTP at the fixed address; the service restarts after a reboot without manual steps.

## B. Records and API

**B1. Grid read model**
`GET /sat/{satId}/grid`: every step with state, holder, lease expiry, revision, and per-slot evidence presence. State derived at read time.
*Done when:* a step claimed by another account shows holder and expiry; an expired lease reads `open` without any background job; a 100-step grid responds in under 200 ms on the site laptop.

**B2. Form read model**
`GET /forms/{formId}` returns form identity, pinned schema version, columns, slots, sections and its steps.
*Done when:* a second form, defined by a second schema row and its own steps, is served with no code change.

**B3. Claim, renew, release, expiry**
Three routes, a 12 h lease, `409` with holder details, `423` on a sealed step, lease expiry derived on read.
*Done when:* two browsers race and one loses with a readable message; renew extends from the server clock; release makes the step immediately claimable; every action writes an audit row.

**B4. Step upsert**
`PUT /steps/{stepId}` with `expected_revision`, values validated against the pinned schema, evidence hashes checked for existence.
*Done when:* `409 stale_revision` carries the current revision; `422 invalid_values` names the column ids; `422 unknown_evidence` names the missing hashes; a successful write increments the revision exactly once.

**B5. Submit**
`POST /steps/{stepId}/submit`, idempotent, `409 not_claimed`, flags computed from empty required columns and empty slots.
*Done when:* submitting twice returns the same body; a submitted step refuses further writes with a clear code; form-level `ready` becomes true only when every step of that form is submitted.

**B6. Seal**
`POST /forms/{formId}/seal`: write the edition with a canonical `content_sha`, record the pinned schema id, refuse `409 steps_open`, return the existing edition on a double tap.
*Done when:* `content_sha` is identical across two seals of identical data and different after any change; a sealed step returns `423`; a second seal returns the same edition id.

**B7. Amendments**
`POST /amendments` reopens named steps of one sealed form; a re-seal writes edition n+1 and marks n superseded; an amendment may be dropped only if no step was written during it.
*Done when:* only named steps reopen; re-seal mints exactly one new edition; dropping after a write is refused with a code and a reason; compile is refused while an amendment is open.

**B8. Error catalogue**
One error shape, one list of codes with their HTTP status, one helper that produces them.
*Done when:* every code named in the spec has a test asserting status and code; an unexpected exception returns the same shape with no stack trace; a test enumerates the catalogue and fails if a code is used that is not listed.

## C. Capture and evidence

**C1. Evidence upload**
`POST /evidence`, multipart, client hash verified, caps enforced before the body is read, `slot_id` recorded, one row per step per slot per revision.
*Done when:* 8 MB passes and 8 MB + 1 byte is `413`; identical bytes return `409 already_exists` and one file; a second image for the same slot in the same revision is `422 slot_occupied`; the file on disk is named by its hash.

**C2. Derivatives and orientation**
Generate the 1000 px document copy; apply EXIF orientation before capping; convert HEIC where the browser hands it over, refuse with `415` where it cannot.
*Done when:* a portrait phone photo lands upright in both the original and the derivative; the derivative is regenerable from the original; a HEIC that cannot be converted is refused with a readable message.

**C0. Stub capture tool**
A small HTTP script for tests and rehearsals: exposes `POST /capture`, reads a request id, posts a PNG from a local folder with the service token. Not shipped to the site.
*Done when:* C3 and C4 pass against the stub on a machine with no WinCC, and the stub is refused when it presents a wrong token.

**C3. Screenshot request, Hub queue, and status**
`POST /requests/screenshot` creates a row and enqueues a Hub-to-tool `POST /capture`; worker sends one at a time FIFO; `GET /requests/{id}` derives `pending`, `ready` or `expired`; a late `POST` is `409 expired`.
*Done when:* tool stopped still returns 202 then reads expired after 90 s; three concurrent requests go out in order and all report ready; a served request reports the evidence hash.

**C4. Service principal**
Token issue at setup, hash stored, header check on the tool route only.
*Done when:* a wrong token is `401`; the token never appears in logs or error bodies; disabling the principal stops the tool route and nothing else.

**C5. Client capture and picker**
Spool in IndexedDB, native camera hand-off, thumbnails, slot assignment, discard before upload, restore after reload.
*Done when:* five captures, two chosen, two uploads leave the device; a reload before save restores the spool; a discarded shot never appears in any request; the same flow works on a phone, with the tablet as the tested path.

**C6. Client grid and step screen**
Grid showing state, holder, expiry and slot progress; step screen with fields, capture, screenshot request with visible state, and submit.
*Done when:* a step held by someone else is visibly locked with the holder named; the screenshot shows waiting, then ready or "no screenshot after 90 seconds"; submit is unavailable unless the step is claimed by this account.

## D. Review and compile

**D1. Template engine**
Render one form from its pinned schema: columns in config order, one row per step, sections, no client template.
*Done when:* a golden-file test renders the fixture schema deterministically; a second schema row renders a second layout with no code change; a wide schema (12 columns) still produces a readable sheet.

**D2. Image placement**
Place one image per slot at its column, set row heights, choose the 1000 px derivative for the document, leave an empty slot visibly empty.
*Done when:* no image overlaps a cell boundary or a neighbouring row; a missing slot leaves a blank cell rather than a shifted row; the file opens in Excel and in LibreOffice without repair prompts.

**D3. Bundle builder**
Assemble the zip from `bundle.layout.json`: one xlsx per form, evidence by hash, `manifest.json`, archive hash recorded on the job.
*Done when:* the manifest validates against its JSON schema; every evidence hash in the manifest exists in the archive; `manual_file` evidence is shipped and not embedded; changing the layout file changes the tree without code changes.

**D4. Compile job**
`POST /compile` with `Idempotency-Key`, preconditions, progress, `GET /compile/{jobId}`, `GET /bundle/{satId}`.
*Done when:* `409 form_unsealed` and `409 form_amended` name the blocking form; the same idempotency key returns the same job; a failed job reports a readable error and leaves no partial bundle.

**D5. Daily summary**
`GET /summary/daily` returns text and stores a `daily_report` row; the client offers copy.
*Done when:* counts match the grid for the day; generating twice creates two rows and both are retrievable; the text names flagged steps rather than hiding them.

## E. Operations and proof

**E1. Backup and restore**
Backup command using SQLite's online backup plus an evidence copy, with a manifest (timestamp, row counts, file count, db hash); restore command into a target folder.
*Done when:* a restore into a scratch folder yields a grid identical in counts and hashes; a tampered evidence file is reported by the manifest check; both commands run from the installed folder with no network.

**E2. Audit coverage**
Every state-changing route writes one audit row: claim, renew, release, write, submit, seal, amendment, compile, backup.
*Done when:* a test drives one step end to end and asserts the ordered audit rows, with server time and actor.

**E3a. Go/no-go checkpoint**
Three outcomes, decided with the client facing owner present: (a) the real capture tool is ready and tested on a non-production target; (b) stub only — the screenshot leg is **unproven** and must be said in the demo and to management; (c) manual file attach is used for the rehearsal and the screenshot leg is deferred.
*Done when:* the outcome is written down with a date, and the demo script matches it.

**E3. Rehearsal**
Run the whole flow on sample forms against a WinCC test target, on a machine that is not the live control room PC: setup, accounts, capture, screenshot, submit, seal, compile, bundle, backup, restore.
*Done when:* one form goes from empty to a sealed edition and a bundle in one sitting; every defect found is logged with the task id it belongs to; the run is repeatable the next day after a service restart.

**E4. Site-day checklist**
Printed checklist: install, accounts, fixed address, Hub address, tool address and port, tool token, backup drive, rollback (restore previous backup), and who to call.
*Done when:* someone who did not build it can follow it without asking a question, and every step has a visible confirmation.

## F. Optional: document ingestion (not on the v1 path)

**Status: optional scope, not scheduled, not part of the December deliverable.** Nothing in A–E depends on it. If it is cancelled, nothing else changes. If it is built, it must not touch the Hub's write path.

**What it does.** Reads the client's SAT document, OCRs the scanned pages, reasons over the extracted text, and maps the result onto the fixed shapes we already own: the form schema (columns, types, slots) and the step list (tag, level, step number). Its only output is a candidate CSV plus a candidate schema JSON, both with provenance, both reviewed and committed by a person through the A5 path.

**What it must not do.** It must not write to the database, call the API, or create a form. Extraction proposes; a person commits. A hallucinated tag that reaches the record is a missing instrument on site day, and it will look correct.

**F0. Source routing.** If the client supplies an XLSX, read cells, not pixels: no OCR, no model, no risk. OCR is only for scanned images and PDFs. *Done when:* the tool refuses to OCR a spreadsheet and says why.

**F1. OCR and layout.** Page rasterising, deskew and rotation, per-word confidence, table and row detection. *Done when:* on a labelled sample, table rows are recovered in reading order with no merged or split rows, and every token carries a confidence and a bounding box.

**F2. Segmentation reasoning.** A model pass that labels regions: title, header row, column labels, instrument table, footnotes. Output is regions and their roles, not values. *Done when:* region roles are correct on the labelled sample, and an unlabelled region is reported rather than guessed.

**F3. Schema mapping.** Map column labels and cell patterns onto the master record: type (text, number, enum), options where a cell set implies them, required flags left **unset by default**, and slot candidates for image columns. *Done when:* every proposed field cites its source column and no field is committed without a human confirmation; a wrong proposal is visible in the diff, not silently applied.

**F4. Step-list extraction.** Tags, level (Cx2/Cx3), step numbers, section assignment, each with page and cell provenance. *Done when:* output is a CSV that the A5 import accepts unchanged, and a duplicate or missing tag is refused by that import rather than repaired silently.

**F5. Review surface.** One side-by-side view: document region, proposed value, confidence, and the ability to accept, edit or reject per row. *Done when:* a reviewer can clear a 30-row document without opening the original, and every rejection is recorded.

**F6. Quality gate.** A labelled set of real pages with hand-entered truth, scored per field: exact match, false add, false drop. *Done when:* the numbers exist for step tags and for schema columns, on paper, and the gate is a written threshold someone else agreed to. Without F6 this whole series is a demo, not a feature.

**Why it is optional.** The source may be a spreadsheet, in which case F is unnecessary. The quality bar needs a labelled sample nobody has produced. And the December path already carries the capture tool and the Wi-Fi assumption; this adds a failure mode to the one input that gates site day, in exchange for saving an afternoon of typing.

**If it is promoted,** it becomes a task with an owner and a date in this plan, and F6 lands before F3 is trusted. Promotion is a scope decision, not an engineering one.

## Named risks, and what the plan does about them

| Risk | What the plan does |
|---|---|
| Capture tool slips, so the screenshot leg is never proven on real equipment | C0 stub proves the Hub side; E3a records the outcome in writing; manual file attach exists as a rehearsal fallback, never as the December path |
| Wi-Fi client isolation blocks tablet-to-Hub traffic | No code answer. Confirmed in writing before site day, or the pilot cannot run |
| Schema drift between build and site day | D16: the schema is pinned, immutable, and recorded in every edition |
| Handover after two developers: nobody knows why a check exists | Every task's "done when" is a test, and the audit row is asserted |

## Traceability

| Decision | Where it lands |
|---|---|
| D4, D16 — form generated from the master record | A5, B2, D1, D2 |
| D6, D17 — accounts, seam, one role | A6, A5 |
| D8 — 12 h lease, no offline claim | B3, C6 |
| D10 — spool, never the record | C5 |
| D11, D12 — hashes, files not blobs | C1, C2, D3 |
| D13 — seal, amendment, edition | B6, B7, D4 |
| D14 — native camera, TLS optional | C5, A7 |
| D15 — four states, submit | B3, B5, C6 |
| D18 — repository seam | A4 |
| D19 — silent tool, three states | C3, C4 |
| D20 — no in-app acceptance | D3, E4 |

## What would change this plan

- **The stack changes to .NET.** A1–A4 and D1–D2 are rewritten; everything else is language-neutral.
- **Step-list loading needs a parser.** A small task becomes a module, and the risk the brief deliberately removed comes back.
- **The capture tool's contract changes** (it starts reporting errors, or it cannot post over HTTP). C3–C4 change; the three-state model may gain a fourth.
- **Document ingestion is promoted.** The F series gains an owner, a date and a quality threshold first; nothing else moves, because it feeds the setup import rather than the Hub.
- **Offline becomes a v1 requirement.** The plan grows a local store, sync and merge, and D9's six rules stop being cheap insurance and become the main design.

If any of these lands, re-plan the affected series. Do not patch tasks silently.
