# SAT PoC v1: architecture package

> Reference annex to the plan pack. The plan (`../00-index.md`) is the single source of truth: on scope, schedule and estimates the plan applies; on routes and payloads the interface spec in this folder applies.

One folder, thirteen artefacts: nine diagrams and four documents — the brief, the interface spec, the build plan and the deferred register.

Built 24 November 2026 from the architecture alignment interview. Revised the same day when the delivery model changed to web-first, and again on 26 November 2026 when the screenshot trigger moved from a file share to HTTP through a Hub queue.

Scope is v1, the December PoC: one site, one SAT, Cx2 and Cx3, two or three engineers, client unnamed.

Audience: the two developers splitting the build, and management reading the same pictures for the schedule verdict.

Status: draft for review. Not agreed with the developers.

## The honest claim

> v1 proves capture with no internet. It does not prove capture with no network. Site Wi-Fi is assumed for December because of the time constraint, and removing that assumption is v2.

That is a smaller claim than the earlier documents made, and it is the true one.

## Read in this order

**Structure**

1. [Who touches it](context.html): actors, external systems, and the boundary. Start here.
2. [Deployment on site](deployment.html): machines, stores, the no-internet line, and the manual upload into ACC.
3. [Components and the one seam](components.html): browser client, Hub API, and the store that v2 will swap.
4. [Step states](step-states.html): open, claimed, in progress, submitted, plus lease expiry.
5. [Sealing and editions](editions.html): a person seals one form, a compile never changes state.

**Sequence**

6. [Capture and submit one step](sequence.html): claim, photo upload, screenshot, record submit.
7. [The screenshot integration](screenshot.html): trigger out over HTTP through the Hub queue, image back over HTTP.
8. [Compile and collect the bundle](compile.html): a job, then a download by a person.
9. [Open the app the next morning](resume.html): session, grid, and what happened to yesterday's claim.

**Text**

10. [Plan pack](../00-index.md): scope, requirements, estimates and open questions. This plan is the single source of truth; hand it over with the diagrams.
11. [Interface spec](architecture-v1-interface-spec.md): routes, payloads, errors, schema, configuration shape, glossary, build order.
12. [Build plan](build-plan.md): the tasks, their dependencies, and the test that says each one is done. No effort, no dates.

Each diagram is a standalone HTML file with dark and light themes and a still or live toggle. The sequences are derived from the spec, so where they disagree, the spec wins.

## Decisions

| # | Decision | Why |
|---|---|---|
| D1 | Standalone product, outside Project SCALE. No SSM identifiers. | SCALE has design documents but no running platform. |
| D2 | The Hub keeps a clean API edge: record store, evidence blob store, and a read API in front of both. No versions, lineage, permissions or publication status. | Nearly free now, and it is the joint a future cloud SCALE reads through. |
| D3 | Images are embedded in the xlsx as image objects, and the same images are filed in the bundle. The bundle keeps an original capped at 3000 px; the document gets a 1000 px copy. | The signer should not open a folder to check a photo. The cap is what stops the workbook and the bundle becoming unusable. |
| D4 | The SAT form is an xlsx **generated from the master record** — the form schema holds columns, order, labels, types, sections, slots and layout. One row per step, one image per slot. `exceljs`. | Supersedes "fill the client's approved template". Config-driven columns make a new form a schema row, and remove template-fidelity work and the commercial docx module. |
| D5 | **Web-first.** The client is a web app served by the Hub. The tablet runs a browser, no install, no local database. | Simplest thing that works for December. Supersedes the earlier Electron decision. |
| D6 | Per-engineer accounts on the Hub, issued by YNY. The session is a normal authenticated HTTP session. The signer does not sign in the app. | The Hub is on the LAN. AD or SSO would not fit W09. |
| D7 | The browser asks the Hub for a screenshot. The Hub calls the tool over HTTP through a simple FIFO queue, and the capture tool pushes the PNG back over HTTP. | HTTP decision 2026-11-26 supersedes the prior file share design. Queue lives in the Hub, single sender, tool stays single shot. The image leg comes back over HTTP so the same evidence path serves both photos and screenshots. |
| D8 | The step claim is decided online by the Hub, with a 12 hour lease. No offline claim in v1. | With one copy of the step, there is nothing to reconcile. |
| D9 | **v2 must run without Wi-Fi.** Plan is a desktop wrapper with a local SQLite store and sync. | The user's stated roadmap. See the six rules below. |
| D10 | The browser keeps an IndexedDB **spool** for in-flight capture only. It is never the record. | Turns "Wi-Fi dropped, start again" into "Wi-Fi dropped, carry on". |
| D11 | Evidence is content addressed. The file name is the SHA-256 of the bytes. | Immutability and idempotent transfer fall out of the addressing, not out of policy. |
| D12 | Bytes are files, never BLOBs. On the Hub at least, and the client sends the capped copy. | The compile needs files, one corruption should not take the records with it, and SQLite's own guidance is files above about 100 KB. |
| D13 | **A form is immutable once sealed.** Sealing one form writes its edition with a `content_sha`; from then on a write returns `423`. Corrections are amendments that reopen named steps and produce the next edition of that form, never edits. A compile packages sealed forms, refuses while any form is draft, and never seals. Evidence is append-only. | A signer signs one artefact, and the artefact is one form, so the edition is a form. Sealing on a person's act rather than on a compile means a rehearsal compile cannot lock the test, and one form's typo cannot re-version the forms that were already right. |
| D14 | **Camera is the device's own camera app.** `<input type="file" capture>` on tablet and phone; one code path, no `getUserMedia`. Tablet is supported, phone is best effort. | No permission dance, and nothing in v1 needs a secure context, so TLS stops being load-bearing. |
| D15 | **Step states are four: open, claimed, in progress, submitted.** `submitted` is an explicit engineer act; "ready to seal" is derived at form level. | Status was in the state diagram but unreachable over the API. A write means saved, not finished. |
| D16 | **The master record is the form schema, versioned and immutable.** Per project. A form pins a schema version, every edition records it, and a schema change after the first seal means a new form. | Different projects differ; a sealed edition must always render the same bytes. |
| D17 | **Auth is a local provider behind a seam.** `user`, `identity`, `session`, one role, one capability check, OSS libraries only. Entra is the v2 provider, not a v2 rewrite. | Three accounts do not justify an IdP service on a site laptop; M365 login needs the cloud anyway. |
| D18 | **SQLite behind a repository module.** No SQL in route handlers. | Keeps the v2 Postgres move to a driver swap plus one migration. |
| D19 | **The capture tool is silent.** It posts the PNG and nothing else; request state is `pending`, `ready`, `expired`, derived by the Hub. | The Hub cannot observe a tool failure, so it must not pretend to. |
| D20 | **The app ends at a sealed bundle.** No e-signature and no in-app acceptance record. | The signer signs in ACC or on paper; v1 does not invent an audit record it cannot verify. |

## What web-first removed

No local database, no sync engine, no push queue, no merge, no unsynced count, no conflict state, no packaging or update problem on three tablets. Roughly 7 to 9 developer days.

## Six rules that keep v2 an addition rather than a rewrite

1. Client-generated ids and idempotent upserts keyed by `(client id, revision)`, never server-assigned identity.
2. Evidence content-addressed by hash, so a resent photo is deduplicated.
3. Device id and a per-device sequence on every write, so offline events can be ordered later.
4. Step state lives in the record, never only in a session.
5. The API stays the only write path, with a small surface: claim, screenshot, submit.
6. Evidence bytes stay out of the database.

Each is nearly free now. Skipping them is what makes v2 a rewrite.

## What the package deliberately leaves out

- No SCALE components, by D1.
- No AI or vision in the running PoC. Setup-time OCR and model extraction is optional scope, described in the build plan's F series and the [deferred register](decisions-and-open-items.md); it is not on the December path.
- No cloud, automated ACC upload, or e-signature.
- No live in-page camera, no client template to fill, no variable image counts per slot.
- No SSO, no IdP service, no RBAC beyond one role.
- No WhatsApp integration: the daily summary is text a person copies.
- No Postgres, no multi-tenant filtering, no second Hub process.
- No print-perfect document layout. Manual touch-ups before signing are accepted.
- The daily summary is generated as text on the Hub; sharing it through WhatsApp stays outside the app.
- No offline capture path, by D5 and D8.

## Estimate

| Item | Ballpark, 18 Nov | Now |
|---|---|---|
| Developer days | 57 | 62 |
| Engineer days | 12 | 12 |
| Platform days | 12 | 12 |
| Total person-days | 81 | 86 |
| Build cost | MYR 69,600 | MYR 73,600 |
| Reserve at 20 percent | MYR 13,920 | MYR 14,720 |
| Build budget | MYR 83,520 | MYR 88,320 |
| Hardware and licences | MYR 26,500 | MYR 26,500 |
| Total PoC | MYR 110,020 | MYR 114,820 |

The developer figure is 57 from the ballpark, plus 3 for the capture tool integration inside W04, plus 2 if the testing proposal is accepted. So 60 to 62 depending on that proposal, which is not yet agreed.

Schedule: 62 developer days at 2.0 FTE is 7.75 weeks against a 6.2 week window. To fit it needs about 2.5 developer FTE. Platform at 0.3 FTE still needs 10 weeks and still binds today; once platform is raised to about 0.5 FTE, the developer becomes the binding constraint.

**A warning about the number.** It has landed near 57 three times today for unrelated reasons: web-first gave back what embedded images added, then the tool work and testing added a little. That is not stability, it is compensation. Nobody should commit to 62 on the strength of it matching the last figure. Re-base the work packages once, against the final scope and a real step list, rather than patching the number every time a decision lands.

### Effect of agentic delivery

There is AI budget to delegate coding to agents. It changes which line binds, and it does not change the date.

**Compresses.** Bounded, testable, well-specified code: the client screens, the API routes, the capture flow, the template engine, the bundle builder, and the capture tool change itself. These have a clear definition of done and can be checked by a test.

**Does not compress.** W01, which is blocked on the step list and a real layout rather than on effort. W11, which is on-site judgement. W10 platform work, which is a site laptop and a network owner. And the step list, the ACC answer and the Wi-Fi confirmation, which are other people's decisions.

**The arithmetic, deliberately generous.** Suppose 20 of the 62 developer days are agent-doable with human review, and agents compress them by 40 percent. That saves 8 days: 54 instead of 62. At 2.0 FTE that is 6.75 weeks against a 6.2 week window. Still over. Platform at 0.3 FTE is still 10 weeks and still the binding constraint.

**The honest conclusion.** AI budget buys code, not the date. The date moves when platform capacity rises to about 0.5 FTE, or when scope narrows. It also introduces a new bottleneck: every agent-authored change needs a named human who owns and reviews it. Two developers reviewing output is a different constraint from two developers writing it, and it is the one nobody estimates.

## Still open

1. **Step-list loading.** Decided: a fixed-column spreadsheet import (R63, WP-1.5.8). Still open: which document wins when the list and the drawings disagree (`O13`).
2. **ACC bundle tree and metadata.** Provisional and behind `bundle.layout.json`; the uploader confirms it. The manifest is not provisional.
3. **Wi-Fi client isolation, in writing.** Wi-Fi is load bearing, so this is a harder blocker than before. No code fallback exists.
4. **Capture tool change.** Accept a request id, POST the PNG with a service token, keep writing locally. Owner and repo agreed; missing are a delivery date and a test target that is not the live control room PC.
5. **Hub hardware and site address.** Which laptop or mini PC, and the fixed LAN address the tablets use.
6. **Backup destination.** Which USB drive, who runs it, where it lives. The procedure and the tested restore are now designed.
7. **Sample SAT form.** Downgraded from blocker to swap-in: the schema is ours and the document is generated. It validates a real layout and supplies the step list.
8. **Retention and destruction.** v1 keeps everything. Who takes the site dataset away, and what is destroyed, is the client's later decision.

Closed since the last revision: TLS is no longer required for the camera, the e-signature question is answered (none in v1), and the daily summary is text only, so no device needs WhatsApp installed.

## Verification

Automated gates passed on all nine diagrams: schema validation, delivery, strict checks, and a real-browser viewport check. No unresolved crossings remain. None of the nine has been reviewed by eye.

**Known debt after the 25 November revision.** The two diagrams that disagreed with the spec — `step-states.html` (five states, a `verified` transition) and `sequence.html` (upload on capture, no submit leg) — were **regenerated on 25 November 2026** from their updated sources. All gates pass: validate, deliver, check and a real-browser viewport check. Neither has been reviewed by eye since.

## Working files

`.working/` holds the nine diagram sources and their gate receipts. Nothing there is a deliverable. Regenerate a diagram from the pack folder:

```bash
node /home/ccw/.agents/skills/archify/bin/archify.mjs finalize architecture \
  architecture/.working/components.architecture.json architecture/components.html --quality showcase
mv architecture/*.json architecture/.working/
```

The type differs by diagram: `architecture` for context, deployment and components; `sequence` for the four call flows; `lifecycle` for step states. The `mv` line matters, because a run always writes its receipts next to the HTML.
