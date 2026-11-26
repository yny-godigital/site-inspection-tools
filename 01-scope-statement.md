# Scope statement

## Problem

An engineer walks a plant and records a site acceptance test on paper and in a folder of phone photos. The instrument readings are typed up later, and the SCADA screens are photographed off a control room monitor or re-taken on a return visit. Correlating a photo to the step it belongs to is done by hand and by memory, and it is the part that fails: a form arrives in the acceptance portal with the right readings and the wrong screenshot, or with a step nobody can evidence. The site has no internet, so nothing cloud-hosted is available on the day the evidence exists. Testing and compiling therefore run in series across separate trips, and a multi-day test accumulates a folder nobody can audit afterwards.

## Outcome

One real SAT, at one site, captured on the day it is tested, compiled on the same day, and accepted by the signing engineer through the client's acceptance portal. After delivery:

- the form follows the standard SAT format, generated from the project's instrument list;
- every step of the Cx2 and Cx3 forms carries its readings, a site photo per slot, and a SCADA screenshot where the step needs one;
- the record cannot reference evidence that does not exist, because a write naming an unknown image hash is refused;
- a form is closed by a person sealing it, and a change after that point is an amendment that produces a second edition, not an edit;
- a person uploads one bundle to the acceptance portal, and that bundle is the artefact the signer signs;
- the whole of the above runs with the internet disconnected and the site LAN up.

## In scope

- A Hub on a site machine: web service, record store, evidence store, and an HTTP API of 22 routes.
- A browser client served by the Hub: sign in, the step grid, the step screen, camera hand-off, the capture spool, submit.
- A form schema (the master record) that generates the workbook: columns, order, labels, types, sections, image slots and layout geometry.
- A template engine that renders one form from its pinned schema version, deterministically, with one image per slot.
- A bundle builder: one workbook per form, evidence filed by hash under a fixed tree, a manifest that names edition, revision and flags.
- Claim, lease, submit, seal, amendment and edition, with the audit rows they produce.
- SCADA screenshot capture by two HTTP legs: Hub calls the tool, the image comes back over HTTP, with a simple FIFO queue in the Hub.
- The change to the existing capture tool so it accepts a request id and posts the PNG back.
- The site-day checklist, the rehearsal on a non-production target, and the go/no-go record for the capture tool.
- Backup and restore from the installed folder, with no internet.
- Defining our standard SAT format as one master record, and the generality test on a second, held-out instrument list.

## Out of scope

| Not doing | Reason |
|---|---|
| Capture with no network at all (v2 item 1) | Decision D9: v1 runs with no internet, not with no LAN. Site Wi-Fi is assumed for December; the durable local store, sync and merge are the v2 shape. |
| OCR and AI extraction of form documents | The form is defined by the standard SAT format (A11). Scanned instrument lists are conditional P6 scope (WP-1.7.1 – WP-1.7.8), not on the December path. |
| Automated upload to ACC (v2 item 3) | Scope interview Q15: a person uploads. Removes service-account negotiation, per-project field mapping and rejection handling from December. |
| E-signature, signature capture, or an in-app acceptance record | Decision D20: the signer signs in ACC or on paper; v1 does not invent an audit record it cannot verify. Needs a legal answer, not code. |
| SSO, Entra ID, any identity provider service | Decision D17: three accounts on a site laptop. Entra is the v2 provider behind the same seam. |
| Roles, RBAC, multi-tenant or multi-project hosting | Scope interview Q9 and the developer note: one site, one SAT, one role. The `role` and `sat_id` columns exist so v2 adds data, not schema. |
| Postgres, a second Hub process, horizontal scaling | Decision D18: SQLite behind a repository module, single process. |
| Cloud deployment of the Hub | Scope interview Q14: the Hub stays on the site LAN for the test. v2 moves it; nothing is rewritten when it does. |
| WhatsApp delivery, the Business API, recipient management | Decision per scope interview Q17: v1 generates text a person copies. No gateway, no attachment pipeline, no approval position. |
| Live in-page camera (`getUserMedia`), in-app video preview | Decision D14: the device's own camera app through a file input. An in-page preview needs TLS with a device-trusted certificate and buys framing help, not new evidence. |
| Client-supplied templates, template fidelity work, docx output | Out of scope. The workbook is generated from the master record (D4). |
| Variable image counts per slot, a second angle in one slot | Interface spec: one image per slot per revision. A second angle is a second slot. |
| Print-perfect pagination, client branding, laid-out document design | Scope interview Q7 and Q19: manual touch-ups before signing are accepted, and generality is bounded to fields, step structure, slots and sections. |
| Any Project SCALE component, or an SSM identifier | Decision D1: a standalone product outside SCALE. SCALE has design documents but no running platform. |
| Client name in any artefact | Leave the client unnamed. Describe Cx2, Cx3 and ACC. |
| Raising the testing effort by 2 developer-days (the W11 proposal) | Proposed, not agreed. It is O8; it is not in the estimates below until it is agreed. |
| TLS termination on the Hub | Decision D14 and the interface spec: desirable hardening, off the critical path. HTTP on a fixed LAN address for December. |
| An agent-written change with no named human reviewer | `build-plan.md`, global definition of done. Every work package below names an owner role; review is a human duty, not a delegated one. |

## Deferred

| Deferred | Revisit when |
|---|---|
| Capture with no Wi-Fi: desktop wrapper, local SQLite, sync, merge | The client asks for a multi-day test on a site with no usable LAN, or v2 starts. |
| Durable local store and sync engine | Same trigger as above. The six rules that keep v2 an addition (client ids, content-addressed evidence, device sequence, state in the record, one write path, bytes out of the database) bind v1 work now. |
| Automated ACC upload | ACC offers a stable API and a service account, and someone owns rejection handling. |
| Cloud Hub, Postgres, container deployment | A second site, or a client who wants the record off the plant. |
| Entra ID over OIDC | A second application needs the same login, or the client requires directory accounts. |
| RBAC and multi-project hosting | A second SAT on the same Hub, or a second role appears in a real workflow. |
| Variable images per slot, docx output, print-perfect layout | A signer refuses a document that needs touch-ups, or the client supplies a mandated template. |
| In-page camera preview | The site has TLS with device-trusted certificates and framing is reported as a real error source. |
| Signature capture and an acceptance record in the app | A legal position exists that says the in-app record is sufficient. |
| Retention and destruction of the site dataset | The client names who takes the dataset away and what is destroyed. v1 keeps everything (R61). |
| WhatsApp automation | The client confirms their WhatsApp is an approved channel for project data. |
| OCR and AI extraction (WP-1.7.1 – WP-1.7.8) | A scanned or PDF instrument list actually arrives, a labelled sample exists, and someone owns the quality threshold. Until then P6 is planned and not built. |
| Raising testing effort by 2 developer-days | The requester accepts the proposal (O8). It adds work to WP-1.5.2, not a new work package. |

## Constraints

| Kind | Value | Hard or soft |
|---|---|---|
| Deadline | 2026-12-31, the witnessed site day | hard |
| Calendar as at 2026-11-25 | 25 working days to the deadline, about 5.0 weeks | hard fact |
| Budget | About MYR 114,820 total, of which MYR 26,500 hardware and licences. It traces to the source package's 62 dev-days; the estimate here is 64 core, band 48.5-80 | soft |
| Team — developer | 2.0 FTE, TypeScript | hard |
| Schedule capacity | 2.0 developer FTE supplies about 40 dev-days in the 5.0 week window, at 4 productive days per FTE-week. The core estimate is 64, and even its 48.5 low band needs 6.1 weeks. Fitting 64 into 5.0 weeks needs 3.2 FTE. This is O15 | hard fact |
| Team — engineer | 12 engineer-days, band 11-13, at about 0.75 FTE, on the critical path at the end | hard |
| Team — platform | 10 platform-days, band 7-12, at 0.3 FTE | hard |
| Site access | The client's site, induction, and a nominated signing engineer. Not in our control | hard |
| Network | Site Wi-Fi must exist, reach the control room, and permit tablet-to-Hub traffic. Written confirmation is O3 | hard |
| Compliance | No formal security or data-protection review identified (A8). The site dataset stays on the site machine and nothing is deleted in v1 | soft |
| Vendor / licence | `exceljs` (MIT), `argon2`, `fastify`, Node 22 LTS. No commercial licence and no cloud service on the critical path | hard |
| Hardware | Site machine plus tablets, about MYR 26,500, owner not yet named (O10) | hard |
| Input | The instrument or IO list is the source of tags, levels and step numbers, and normally arrives as a spreadsheet (A12). Which document wins when the list and the drawings disagree is O13 | hard |

## Assumptions

| ID | Assumption | If wrong | Lands in |
|---|---|---|---|
| A1 | Site Wi-Fi exists, reaches the control room, and permits tablet-to-Hub traffic | The pilot cannot run on site. There is no code fallback; the record has to be carried by hand | WP-1.6.2, §Constraints |
| A3 | The capture tool change lands with a non-production test target before the rehearsal | The screenshot leg stays unproven. The stub proves the Hub side only, and the go/no-go record has to say so | WP-1.3.8, WP-1.6.6 |
| A4 | Platform capacity rises to about 0.5 FTE | Platform needs over 8 weeks at 0.3 FTE against a 5.0 week window, and the date fails no matter how many developers are added | §Constraints, O7 |
| A5 | The standard format allows at most one image per slot per step | The schema needs variable slots, which is a schema change and, after the first seal, a new form | WP-1.5.4 |
| A6 | One Hub serves one SAT, so no route carries a SAT id in its form paths | Every form, seal and amendment route changes shape, and the client work follows | WP-1.2.3, WP-1.2.7 |
| A7 | The tree the ACC uploader needs is close to the provisional one in `bundle.layout.json` | A config edit, not code, provided the manifest stays ours | WP-1.4.3, O5 |
| A8 | No formal security or data-protection review applies to this delivery | A 2-4 week review lands on the critical path with about 5 weeks left, and it can return conditions that change the evidence store | §Constraints, WP-1.5.6 |
| A9 | There is no outside anchor for these estimate bands, so they stay deliberately wide | A comparable real duration would narrow them, and the width below is over-insurance rather than honesty | 05 § Estimate summary, WP-1.5.2 |
| A10 | The two developers accept the boundary in this pack | The pack is a proposal, not an agreement. The architecture package already says "draft for review, not agreed with the developers" | All phases |
| A11 | The SAT format is a single standard, defined and versioned once | The master record becomes per-project, the generality proof gets heavier, and WP-1.5.4 grows by the per-client design time | WP-1.5.4, WP-1.5.5, §In scope |
| A12 | The instrument or IO list arrives as a spreadsheet in the normal case, and a scan or PDF is the exception | The spreadsheet importer is the wrong main path, P6 stops being conditional and becomes the critical path, and the December date is not reachable | WP-1.5.8, P6, §Constraints |

## Authority

| Role | Person | Power |
|---|---|---|
| Accepts each phase | Not named — O1 | signs off phases |
| Accepts the pilot outcome | The client's nominated signing engineer, not yet nominated — O1 | accepts the site day, in ACC or on paper |
| Approves a scope change after the seal | Not named — O2 | sole authority once the pack is sealed |
