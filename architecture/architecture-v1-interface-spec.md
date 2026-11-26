# SAT PoC v1 interface spec

> Reference annex to the plan pack. The plan (`../00-index.md`) is the single source of truth: on scope, schedule and estimates the plan applies; on routes and payloads this file applies.

Status: draft, revised 25 November 2026 after the design decision pass. Not agreed with the developers. Companion to the nine diagrams in this folder.

Decisions in this document are settled design, not options. What v1 deliberately does not build is in [decisions-and-open-items.md](decisions-and-open-items.md). Where a diagram and this document disagree, this document wins. Two diagrams were regenerated for this revision; see "Diagram status" at the end.

## Conventions

**Transport.** HTTP on the site LAN at a fixed address. No internet. TLS is optional hardening, not a requirement: the camera uses the device's own camera app rather than a secure-context browser API, so nothing in v1 needs HTTPS.

**Auth.** A local account provider behind a seam. A session cookie from `POST /auth/login`, `HttpOnly`, `SameSite=Lax`, sliding expiry of 12 hours. Server-side session record, so a Hub restart logs everyone out. Plan for that on a multi-day test. `OidcProvider` (Entra) is the v2 implementation of the same seam; no v1 code path depends on it.

**Identity.** Three identities matter: the engineer account, the device, and the service principal. `device_id` is a browser-generated UUID stored in `localStorage` and sent on every write. The capture tool is a service principal with its own token.

**Ids.** Client-generated UUIDv7 for steps and evidence. The client never waits for the server to name a thing, which is what makes a v2 replay possible.

**Time.** `captured_at` is client time. `received_at`, `claimed_at`, `claimed_until` and every audit row are server time. Where they disagree, the server wins. Write that down or the audit trail is ambiguous.

**Idempotency.** Every write is safe to repeat. Evidence is keyed by content hash. Steps are updated with a revision check. `POST /steps/{id}/submit` is idempotent on an already-submitted step.

**Errors.** One shape: `{"error": {"code": "...", "message": "...", "details": {}}}`.

**Caps.** One config constant: 3000 px long edge, 8 MB after client re-encode, reject above 20 MB before decode, tool PNG accepted up to 10 MB.

## Routes

Twenty-two routes.

| Method | Path | Purpose | Safe to repeat |
|---|---|---|---|
| POST | `/auth/login` | account + password to session | yes |
| POST | `/auth/logout` | end session | yes |
| GET | `/me` | current account, role, capabilities | yes |
| GET | `/sat/{satId}/grid` | step list with state, holder, lease expiry, slot progress | yes |
| GET | `/forms/{formId}` | form identity, schema version, columns, slots | yes |
| POST | `/steps/{stepId}/claim` | grant or refuse a 12 hour lease | yes, returns the same lease |
| POST | `/steps/{stepId}/renew` | extend while the step is open | yes |
| POST | `/steps/{stepId}/release` | give it back | yes |
| POST | `/steps/{stepId}/submit` | the engineer's act that finishes the step | yes, once submitted |
| POST | `/evidence` | upload one blob, bound to a slot | by hash |
| HEAD | `/evidence/{sha256}` | do you already have it | yes |
| GET | `/evidence/{sha256}` | fetch a blob | yes |
| PUT | `/steps/{stepId}` | upsert values and evidence at a revision, only while its form is draft or amended | by `(id, revision)` |
| POST | `/forms/{formId}/seal` | close one form: write its edition, freeze its steps | yes, returns the same edition |
| POST | `/amendments` | reopen named steps of a sealed form, reason required | by reason and set |
| POST | `/requests/screenshot` | ask the Hub to trigger one screenshot, returns a request id | no, may be slow |
| GET | `/requests/{requestId}` | `pending`, `ready` or `expired`, with the evidence hash when ready | yes |
| POST | `/evidence/screenshot` | the capture tool pushes the PNG for a request id | by request id |
| POST | `/compile` | start a bundle job; refuses if any form is draft or amended | by `Idempotency-Key` |
| GET | `/compile/{jobId}` | job status | yes |
| GET | `/bundle/{satId}` | download the finished bundle | yes |
| GET | `/summary/daily` | the daily test summary, text, also stored | yes |

Anything that does not fit this table should be questioned, because every addition has to survive the move to cloud.

**Two rules the route table now enforces.** Status is not a writable field: it is derived from claim, lease and submit. And a screenshot request has a read half, because without `GET /requests/{id}` the browser cannot tell waiting from broken.

## Payloads

### Claim

```json
POST /steps/st_7f3a/claim
{ "device_id": "dv_01H..." }

200 OK
{
  "step_id": "st_7f3a",
  "revision": 2,
  "claimed_by": "kai.xin",
  "claimed_until": "2026-12-01T21:14:00Z",
  "server_time": "2026-12-01T09:14:00Z"
}
```

`409` when someone else holds it, with the holder and the expiry. `423 sealed` when the step's form is sealed and no open amendment names that step, which is the answer that sends the caller to `POST /amendments` instead of editing.

### Evidence upload

```json
POST /evidence        (multipart: meta + bytes)
{ "client_hash": "b3d1e0…4a7", "kind": "site_photo", "slot_id": "photo_1",
  "step_id": "st_7f3a", "captured_at": "2026-12-01T09:12:41Z", "caption": "" }

201 Created
{ "sha256": "b3d1e0…4a7", "bytes": 1284311, "width": 3000, "height": 2000 }
```

`400 hash_mismatch`. `413 too_large` above 8 MB after encode (check the declared length before reading the body). `409 already_exists` is fine and means the blob is present. `422 slot_occupied` when that step already has evidence for that slot in that revision — one image per slot per revision, so a retake replaces the spooled file before it is ever sent.

The client re-encodes to the 3000 px cap before sending, so the wire never carries a camera original. Non-JPEG and non-PNG output is converted at this step; a format that cannot be converted is refused with `415 unsupported_media` rather than stored and failed later. The Hub derives the 1000 px document copy itself.

`kind: manual_file` is accepted and stored, never embedded in the document; see the bundle section.

### Screenshot, three calls over HTTP with a Hub queue

```json
POST /requests/screenshot
{ "step_id": "st_7f3a", "tag": "PT-1001", "device_id": "dv_01H..." }

202 Accepted
{ "request_id": "rq_01H...", "expires_at": "2026-12-01T09:15:30Z" }
```

The Hub creates the request row, enqueues a Hub-to-tool call carrying `request_id`, and returns. Queue is FIFO in memory, single sender, one call at a time. Send is `POST http://tool:8080/capture` with `request_id`, `tag`, `expires_at` and a bearer token, 5 s connect timeout. If the tool cannot be reached the request stays pending until expiry and the Hub log carries the error. The queue depth is 100; a request that arrives when it is full is refused with `503 queue_full` and no request row is created, so the engineer retries rather than waiting on a request that will never be sent.

The tool is silent: its only obligation is the POST below. The Hub therefore never claims to know why a shot failed.

```json
GET /requests/rq_01H...

200 OK
{ "request_id": "rq_01H...", "state": "pending",
  "expires_at": "2026-12-01T09:15:30Z", "server_time": "2026-12-01T09:14:12Z" }
```

States are derived from one row, not stored: `ready` when the evidence hash is set, `expired` when the deadline has passed with no hash, `pending` otherwise. There is no `failed` in v1 simple queue. A queued request that never gets an image expires like any other.

The browser polls every two seconds and stops at expiry. Retry is manual: the engineer asks again, which creates a new request id.

```json
POST /evidence/screenshot        (the capture tool calls the Hub)
{ "request_id": "rq_01H...", "tool": "wincc_capture", "tool_version": "2.1",
  "bytes": "<png>" }

201 Created
{ "sha256": "7f22ab…9c1", "bytes": 148003, "width": 1920, "height": 1080 }
```

The tool authenticates with a service principal token in `Authorization`, issued by the setup command. No cookie, no IP allowlist. `401 bad_service_token`. `404 unknown_request` when the id does not exist. `409 expired` when the deadline passed — the bytes are dropped and the engineer retries; the Hub does not accept a late image silently. `409 already_served` is fine.

The two legs are separate on purpose. The trigger goes out over HTTP through the Hub queue so ordering lives in one place and the tool stays single shot. The image comes back over HTTP because that is the leg where you want a confirmation, a timestamp and an error. After that the bytes take the same path as a site photo: hashed, stored, one `evidence` row.

Tool inbound, simple first:

```json
POST http://tool:8080/capture        (the Hub calls the tool)
Authorization: Bearer <hub_to_tool_token>
{ "request_id": "rq_01H...", "tag": "PT-1001", "expires_at": "2026-12-01T09:15:30Z" }

202 Accepted
{ "request_id": "rq_01H...", "accepted": true }
```

Tool checks the bearer token, captures the screen for the tag, then posts to `/evidence/screenshot` as above. Tool runs one shot at a time. Hub sends one at a time in `created_at` order. No Hub retry. A new tap makes a new request id.

### Step upsert

```json
PUT /steps/st_7f3a
{
  "revision": 2,
  "expected_revision": 1,
  "device_id": "dv_01H...",
  "client_seq": 41,
  "values": { "reading": "4.02", "unit": "bar", "result": "pass" },
  "evidence": ["b3d1e0…4a7", "7f22ab…9c1"]
}

200 OK  { "revision": 2, "stored_at": "2026-12-01T09:15:02Z" }
409 stale_revision  { "current": 3 }
409 not_claimed     { "claimed_by": "kai.xin", "claimed_until": "..." }
422 unknown_evidence { "missing": ["7f22ab…9c1"] }
```

`409 not_claimed` when the caller does not hold the step's lease, because a write from a non-holder would advance the revision under the holder's feet.

`422 unknown_evidence` is the important one: a step may not reference bytes the Hub has not confirmed. That is the failure that otherwise produces a good-looking record with a missing photo.

`values` is validated against the schema for the form's pinned version: required columns, enum options, number format. A failure is `422 invalid_values` with the offending column ids. Validation warnings do not block a save; they are what the seal reports as flags.

### Submit

```json
POST /steps/st_7f3a/submit
{ "device_id": "dv_01H...", "note": "" }

200 OK
{ "step_id": "st_7f3a", "status": "submitted",
  "revision": 2, "flags": ["slot photo_2 empty"] }
```

`409 not_claimed` when the caller does not hold the lease. `409 stale_revision` when the caller's revision is not current. Submitting twice returns the same result. Submit does not block on flags: a missing slot is recorded and travels into the manifest, because the alternative is an engineer who cannot finish a step and cannot say why.

### Seal a form

```json
POST /forms/cx2-instrument-v1/seal
{ "device_id": "dv_01H...", "note": "panel walked, photos filed" }

201 Created
{
  "form_id": "cx2-instrument-v1",
  "edition_id": "ed_01H...",
  "edition_number": 1,
  "schema_version": "2026-11-24.1",
  "content_sha": "9a41c2…0fe",
  "sealed_at": "2026-12-01T18:40:00Z",
  "flags": ["st_7f3a: no scada_screenshot", "st_802b: result empty"]
}
```

`409 already_sealed` returns the edition that already exists, so a double tap is safe. `409 steps_open` names steps that are not submitted, because a form cannot be sealed around a step nobody finished. Flags do not block the seal; they travel into the manifest, so a missing photo stays visible instead of becoming a surprise on site day.

`formId` names a form without a SAT id because one Hub serves one SAT in v1. If v2 hosts several, the route grows to `/sat/{satId}/forms/{formId}/seal`.

`content_sha` is a hash of that form's pinned schema version, its steps and their evidence at the moment of sealing, in a canonical order. It is what makes two editions of one form distinguishable later, and it is only honest while the schema version is immutable.

## The step record

| Field | Meaning |
|---|---|
| `id` | client-generated, stable for the life of the step |
| `sat_id`, `form_id`, `level` | Cx2 or Cx3 |
| `step_no`, `tag` | position and instrument tag |
| `status` | open, claimed, in progress, submitted. Derived from lease and submit; never written by a client |
| `revision` | integer, increments on every accepted write |
| `values` | JSON keyed by column id from the pinned schema version |
| `claimed_by`, `claimed_until` | the lease |
| `device_id`, `client_seq` | who wrote it, and in what order from that device |
| `updated_at`, `updated_by` | last accepted write |
| `submitted_at`, `submitted_by` | the engineer's finishing act |

Values are JSON rather than columns on purpose. Columns per form would make a new schema a migration, and the whole generality claim is that a new form is a schema entry.

## Schema

```sql
CREATE TABLE user (
  id            TEXT PRIMARY KEY,
  username      TEXT NOT NULL UNIQUE,
  display_name  TEXT NOT NULL,
  role          TEXT NOT NULL DEFAULT 'engineer',   -- one role in v1
  password_hash TEXT NOT NULL,
  active        INTEGER NOT NULL DEFAULT 1,
  created_at    TEXT NOT NULL
);

CREATE TABLE identity (
  provider  TEXT NOT NULL,          -- 'local' in v1, 'entra' in v2
  subject   TEXT NOT NULL,          -- username, or OIDC sub
  user_id   TEXT NOT NULL REFERENCES user(id),
  PRIMARY KEY (provider, subject)
);

CREATE TABLE session (
  id         TEXT PRIMARY KEY,      -- opaque cookie value
  user_id    TEXT NOT NULL REFERENCES user(id),
  device_id  TEXT,
  created_at TEXT NOT NULL,
  expires_at TEXT NOT NULL
);

CREATE TABLE service_principal (
  id         TEXT PRIMARY KEY,      -- e.g. 'capture_tool'
  label      TEXT NOT NULL,
  token_hash TEXT NOT NULL,
  active     INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL
);

CREATE TABLE form_schema (          -- the master record
  id         TEXT PRIMARY KEY,      -- e.g. 'cx2-instrument-v1@2026-11-24.1'
  form_id    TEXT NOT NULL,
  version    TEXT NOT NULL,
  payload    TEXT NOT NULL,         -- columns, sections, slots, layout
  created_at TEXT NOT NULL,
  UNIQUE (form_id, version)
);

CREATE TABLE form (
  id             TEXT PRIMARY KEY,  -- e.g. 'cx2-instrument-v1'
  sat_id         TEXT NOT NULL,
  level          TEXT NOT NULL CHECK (level IN ('Cx2','Cx3')),
  title          TEXT NOT NULL,
  schema_id      TEXT NOT NULL REFERENCES form_schema(id),  -- pinned, immutable
  created_at     TEXT NOT NULL
);

CREATE TABLE steps (
  id               TEXT PRIMARY KEY,
  sat_id           TEXT NOT NULL,
  form_id          TEXT NOT NULL REFERENCES form(id),
  level            TEXT NOT NULL CHECK (level IN ('Cx2','Cx3')),
  step_no          INTEGER NOT NULL,
  tag              TEXT,
  status           TEXT NOT NULL CHECK (status IN
                     ('open','claimed','in_progress','submitted')),
  revision         INTEGER NOT NULL DEFAULT 1,
  values_json      TEXT NOT NULL DEFAULT '{}',
  claimed_by       TEXT,
  claimed_at       TEXT,
  claimed_until    TEXT,
  updated_at       TEXT,
  updated_by       TEXT,
  device_id        TEXT,
  client_seq       INTEGER,
  submitted_at     TEXT,
  submitted_by     TEXT,
  UNIQUE (sat_id, form_id, step_no)
);

CREATE TABLE evidence (
  id            TEXT PRIMARY KEY,
  step_id       TEXT NOT NULL REFERENCES steps(id),
  slot_id       TEXT NOT NULL,      -- slot in the pinned schema
  kind          TEXT NOT NULL,
  sha256        TEXT NOT NULL,
  bytes         INTEGER NOT NULL,
  mime          TEXT NOT NULL,
  width         INTEGER, height INTEGER,
  captured_at   TEXT NOT NULL,
  captured_by   TEXT NOT NULL,
  device_id     TEXT NOT NULL,
  source_app    TEXT,
  caption       TEXT,
  revision      INTEGER NOT NULL,   -- the step revision it is attached at
  received_at   TEXT NOT NULL,
  UNIQUE (step_id, slot_id, revision)
);

CREATE TABLE evidence_derivative (
  evidence_id TEXT NOT NULL REFERENCES evidence(id),
  purpose     TEXT NOT NULL,
  sha256      TEXT NOT NULL,
  bytes       INTEGER NOT NULL,
  width       INTEGER NOT NULL, height INTEGER NOT NULL,
  created_at  TEXT NOT NULL,
  PRIMARY KEY (evidence_id, purpose)
);

CREATE TABLE screenshot_request (
  id                  TEXT PRIMARY KEY,
  step_id             TEXT NOT NULL REFERENCES steps(id),
  tag                 TEXT,
  requested_by        TEXT NOT NULL,
  device_id           TEXT,
  created_at          TEXT NOT NULL,
  expires_at          TEXT NOT NULL,
  served_evidence_id  TEXT,         -- null until the tool posts the PNG
  served_at           TEXT
);

CREATE TABLE daily_report (
  id         TEXT PRIMARY KEY,
  day        TEXT NOT NULL,
  sat_id     TEXT NOT NULL,
  generated_by TEXT NOT NULL,
  generated_at TEXT NOT NULL,
  body       TEXT NOT NULL
);

CREATE TABLE audit (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  at          TEXT NOT NULL,          -- server time, always
  actor       TEXT,
  device_id   TEXT,
  entity      TEXT NOT NULL,          -- step | evidence | lease | bundle | form
  entity_id   TEXT NOT NULL,
  action      TEXT NOT NULL,
  detail_json TEXT
);

CREATE TABLE edition (
  id             TEXT PRIMARY KEY,
  sat_id         TEXT NOT NULL,
  form_id        TEXT NOT NULL,
  number         INTEGER NOT NULL,
  schema_id      TEXT NOT NULL,      -- what layout produced this document
  content_sha    TEXT NOT NULL,      -- hash of this form's steps and evidence at seal
  sealed_at      TEXT NOT NULL,
  sealed_by      TEXT NOT NULL,
  superseded     INTEGER NOT NULL DEFAULT 0,
  UNIQUE (form_id, number)
);

CREATE TABLE amendment (
  id         TEXT PRIMARY KEY,
  form_id    TEXT NOT NULL,
  edition_id TEXT NOT NULL REFERENCES edition(id),
  reason     TEXT NOT NULL,
  step_ids   TEXT NOT NULL,          -- JSON array
  opened_at  TEXT NOT NULL,
  opened_by  TEXT NOT NULL,
  closed_at  TEXT
);

CREATE TABLE compile_job (
  id          TEXT PRIMARY KEY,
  sat_id      TEXT NOT NULL,
  requested_by TEXT NOT NULL,
  idem_key    TEXT UNIQUE,
  state       TEXT NOT NULL,          -- queued | running | done | failed
  bundle_path TEXT,
  bundle_sha  TEXT,                   -- hash of the finished archive
  started_at  TEXT, finished_at TEXT,
  error       TEXT
);
```

Bytes never enter these tables. Files live in `evidence/<sha256>` and `derived/<sha256>`. The `bytes` columns store file size, not file content.

The step list is project data, not schema: it arrives as rows in `steps`, loaded at setup by the fixed-column spreadsheet import (R63). Which document wins when the list and the drawings disagree is `O13`.

Form **state** is still derived, and that is on purpose:

| Derived state | Test |
|---|---|
| `draft` | no unsuperseded edition for the form |
| `ready` | no edition yet, and every step of the form is `submitted` |
| `sealed` | one unsuperseded edition, and no open amendment names the step |
| `amended` | an open amendment row names the step |
| `superseded` | the edition has a later edition of the same form |

There is no step-level `complete`. "All fields done, ready to seal" is a statement about the form, and duplicating it per step invites two answers to one question.

## Configuration shape

The master record is the form schema. It is per project, versioned, and immutable once any form pins it.

```json
{
  "id": "cx2-instrument-v1@2026-11-24.1",
  "form_id": "cx2-instrument-v1",
  "version": "2026-11-24.1",
  "level": "Cx2",
  "sections": [
    { "id": "inst", "title": "Instrument checks", "layout": "table" }
  ],
  "columns": [
    { "id": "reading", "label": "Reading", "type": "number", "required": true, "order": 3 },
    { "id": "unit",    "label": "Unit",    "type": "enum", "options": ["bar","kPa"], "order": 4 },
    { "id": "result",  "label": "Result",  "type": "enum", "options": ["pass","fail","n/a"], "required": true, "order": 5 }
  ],
  "slots": [
    { "id": "photo_1", "kind": "site_photo",       "order": 6 },
    { "id": "screen_1","kind": "scada_screenshot", "order": 7 }
  ],
  "layout": { "row_start": 8, "row_per_step": 3, "sheet": "SAT" }
}
```

Rules that follow:

- **Column order is config.** There is no anchor to a client template, because there is no client template. The document is generated from this schema and the pinned step list.
- **One image per slot per revision.** A second angle is `photo_2`, a second slot, not a second image in `photo_1`.
- **A slot binds a kind, not a filename.** The compile pairs slot with evidence by `slot_id`.
- **A step with no evidence renders an empty slot** rather than failing.
- **Schema version is pinned by the form at creation.** Editing it produces the next version; it never edits a version already referenced. After the first seal, a schema change means a new `form_id`, because re-rendering a sealed form under new columns would make `content_sha` lie.

The generality test is unchanged: render a second form from a second schema row plus its steps, with no code change.

## Forms, sealing and amendments

A form is editable until it is sealed. Sealing is a person's act on one form, not a side effect of compiling. `POST /forms/{formId}/seal` writes one `edition` row for that form, carrying `content_sha` and the pinned `schema_id`, and from that point every step of that form returns `423 sealed`.

```
draft ──seal──▶ sealed (edition n) ──amendment──▶ named steps open ──re-seal──▶ sealed (edition n+1)
                      │
                      └── later edition ──▶ superseded (kept, never deleted)
```

A correction after sealing is an **amendment**, not an edit:

```json
POST /amendments
{ "form_id": "cx2-instrument-v1", "edition_id": "ed_01H...",
  "reason": "reading 12 typed as 4.20", "step_ids": ["st_7f3a"] }

201 Created  { "amendment_id": "am_01H...", "steps_unsealed": ["st_7f3a"] }
```

Only the named steps reopen, and only inside that form. Editing them increments their revision and appends evidence, never replaces it. The next seal writes edition n+1 of the same form and marks edition n superseded. No other form moves. Both `content_sha` values stay recorded and the compile records `bundle_sha`, so "which copy went to ACC" is answerable at the form and at the bundle.

Compile packages sealed forms only. It never seals anything itself, and it refuses on two preconditions:

```
409 form_unsealed   any form is still draft
409 form_amended    any form has an open amendment
{ "error": { "code": "form_amended",
             "details": { "form_id": "...", "amendment_id": "...", "opened_by": "..." } } }
```

The second refusal is deliberate. A form is sealed and then amended only because the sealed content is known to be wrong, so compiling at that moment would ship something already known wrong. The amendment is short-lived by design, so the refusal costs minutes, not days.

A SAT is ready for the client when every form is sealed, no amendment is open, and one compile has run.

Seven rules follow, and they are the reason this is simpler than the reopen-after-submit flow it replaces:

1. **Evidence is append-only.** No row is deleted and no file is overwritten, because the file name is the hash.
2. **The current image for a slot is the one with the highest step revision at or below the form's sealed edition.** One row per slot per revision makes this unambiguous without a timestamp tie-break.
3. **A sealed edition never changes.** Any change produces a new edition of that form, so the record of what a signer saw cannot drift.
4. **A compile never changes state.** It reads and writes bytes. The only thing that freezes a form is a person sealing it, so a rehearsal compile cannot lock the test.
5. **An amendment may be dropped only if no step was written during it.** Once a write lands the audit trail has changed, and the only way forward is a new edition. Otherwise "we cancelled the amendment" becomes a way to hide an edit.
6. **An open amendment blocks compiling.** A sealed form with an open amendment is not compile-ready, because the seal is known to be wrong. `409 form_amended` names the form.
7. **Write scope and supersede scope are different.** An amendment may touch one step or all of them; the re-seal issues one new edition of the whole form either way. Five wrong readings are one amendment and one edition, not five.

Two things this buys, stated plainly, because both were defects in the compile-seals-everything rule it replaces:

- An early compile to check the layout, the row heights or the image size is free. It cannot seal the SAT.
- A typo in one form does not re-version the forms that were already right.

## Bundle layout

The tree below is **provisional** until the person who uploads to ACC confirms it. It is produced by one function whose input is `bundle.layout.json`, so the confirmation is a config change, not a code change.

```
SAT-<satId>/
  01-Cx2/
    <formId>-form.xlsx
    evidence/012-photo_1-b3d1e0.jpg
    evidence/012-screen_1-7f22ab.png
  02-Cx3/
    ...
  manifest.json
```

`manifest.json` is **not** provisional. It carries `satId`, `generatedAt`, one entry per form with its `formId`, `schemaId`, `editionId`, `editionNumber` and `content_sha`, one entry per step with `revision`, `submittedBy` and `evidence[]`, and a `flags[]` array naming incomplete steps. The document copy and the original are both listed, with the original marked authoritative.

`manual_file` evidence is listed in the manifest and copied into `evidence/`, and is never embedded in the document.

The bundle is delivered as one zip. A folder tree is possible from the same source if the uploader prefers it, which is one line in `bundle.layout.json`.

## Glossary

| Term | Meaning |
|---|---|
| **SAT** | Site acceptance test. One SAT is the unit of the bundle. |
| **Form** | One xlsx rendered from a schema version and a step set. The unit of sealing, of the edition and of the manifest entry. |
| **Master record** | The form schema: columns, order, labels, types, sections, slots, layout geometry. Per project, versioned, immutable once pinned. Not the instruments. |
| **Step list** | Project data: instruments, tags, levels, step numbers. Loaded into `steps` at setup. |
| **Seal** | A person's act that closes one form and writes its edition. Not a compile. |
| **Edition** | One sealed version of one form, identified by `content_sha`, produced under one pinned schema version. Indivisible: a one-step correction still produces a new edition of the whole form. |
| **Amendment** | An action, not a state. It reopens a named **set** of steps (one or many, always a subset of one form) with a required reason. The write scope is that subset; the effect is not, because re-sealing issues a new edition of the whole form. It ends at re-seal, or is dropped only if no step was written during it. |
| **Superseded** | A state, not an action, and always whole-form: an older edition replaced by a newer edition of the same form. Kept on disk, never edited, never deleted, and still the record of what was signed on its date. |
| **Cx2, Cx3** | The client's commissioning levels. Both live inside one SAT bundle. |
| **ACC** | The client's acceptance portal. A person uploads to it in v1. |
| **Hub** | The site server. Web service, record store, evidence store. |
| **Browser client** | The browser app, served by the Hub. |
| **Record store** | SQLite, behind a repository module. Steps, evidence metadata, audit, jobs. |
| **Evidence store** | Files named by hash. Original and document copy. |
| **Spool** | Browser IndexedDB holding in-flight captures and one in-flight form. Never the record. |
| **Lease** | A 12 hour grant that one account holds a step. |
| **Slot** | A place in the form where evidence of one kind goes. One image per slot per revision. |
| **Kind** | Site photo, SCADA screenshot, manual file. |
| **Revision** | Integer on a step; every accepted write increments it. |
| **Derivative** | A regenerable copy of evidence, such as the 1000 px document image. |
| **Bundle** | The upload-ready folder for one SAT. |
| **Manifest** | The index inside a bundle. |
| **Capture trigger** | Hub calls `POST /capture` on the tool. Tool pushes images back over HTTP; the Hub queue holds order. |
| **Service principal** | A non-human identity with a token. The capture tool is one. |

## Build order

This is the shape of the build. The task-level version, with dependencies and a test per task, is [build-plan.md](build-plan.md).

1. Accounts, sessions and the capability check; the repository module.
2. Master record and the step list, pinned to a schema version.
3. Schema and the API skeleton, with the three write routes stubbed.
4. Claim, lease, submit and seal, including lease expiry and the form-level `ready` state.
5. Capture, the picker, evidence upload with slot binding and the hash check.
6. Template engine, generated from the schema, with the second-schema test.
7. Screenshot request, Hub FIFO queue, tool `POST /capture`, status polling, tool token.
8. Bundle builder, manifest and `bundle.layout.json`.
9. Grid, daily summary, backup and restore.
10. Rehearsal on a sample form and a WinCC test target.
11. Site day.

Steps 3 to 5 are the split point: one developer on the Hub, one on the client, with this document as the contract.

## Diagram status

This revision changed more than the diagrams were built from. Two disagreed and were regenerated from `.working/*.json` on 25 November 2026, with all gates passing (validate, deliver, check, browser check):

- `step-states.html` now shows four states, with `submitted` reached by an explicit act and no `complete` or `verified` transition.
- The old `sequence.html` summary now shows pick-before-upload, the screenshot status poll, and a separate submit leg, but it was removed on 26 November 2026. The presentation S set (S04 plus S05 plus S06 plus S07, and S10) carries that flow now.

Neither has been reviewed by eye since regeneration. The other seven diagrams were not affected by this revision and are unchanged.

## Open items

The deferred-to-v2 register lives in [decisions-and-open-items.md](decisions-and-open-items.md). What remains unresolved for v1 is listed here, each with an owner and a consequence:

- **Instrument-list conflicts (`O13`).** The list arrives as a fixed-column spreadsheet (R63). Which document wins when its tags, ranges or levels disagree with the drawings is still open.
- **ACC bundle tree and metadata.** The tree above is provisional; the uploader confirms it. Blocks nothing, because the manifest is ours and the tree is one config file.
- **Wi-Fi client isolation, in writing.** If tablets cannot reach the Hub, v1 does not work. No design fallback exists; this is a site fact, not a code question.
- **Capture tool change.** It must accept a `request_id` over HTTP, POST the PNG with a service token, and keep writing locally. Owner and repository are agreed; what is missing is a delivery date and a test target that is not the live control room PC. On the critical path.
- **Optional document ingestion (F0–F6 in the build plan).** Unbuilt and unscheduled: OCR on scanned pages, a model pass to reason over layout, then a mapping onto the schema and step-list shapes, emitted as a candidate CSV with provenance and committed by a person through the setup import. It never writes to the Hub. See [decisions-and-open-items.md](decisions-and-open-items.md).
- **Sample SAT form document.** No longer a blocking dependency: the format is defined by the schema and the document is generated. A real layout and step list remain as swap-in validation for the master record.
- **Hub hardware and site address.** Which laptop or mini PC, and the fixed LAN address the tablets use.
- **Backup destination.** Which USB drive, who runs it, and where it is stored when not in use.
