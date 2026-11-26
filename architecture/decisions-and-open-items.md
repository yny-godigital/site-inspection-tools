---
title: "SAT PoC v1: Decisions deferred to v2"
version: "0.1"
date: "2026-11-25"
---

# Decisions deferred to v2

> Reference annex to the plan pack. The plan (`../00-index.md`) is the single source of truth; the deferred table in `01` carries what v1 does not build.

[Architecture package](README.md)

Status: Working register. Records what v1 deliberately does not build, so the work can be picked up later. Nothing here is scheduled or estimated.

v1 is the December PoC: one site, one SAT, site Wi-Fi assumed. Each decision below was made to keep v1 inside the December window, or to keep it small enough to finish. v1 decisions (D1 to D20) live in the [architecture package](README.md); the interface contract is the [v1 interface spec](architecture-v1-interface-spec.md).

## Deferred to v2

### 1. Run without Wi-Fi

- v2 must run with no Wi-Fi at all. v1 proves capture with no internet, not with no network; site Wi-Fi is assumed for December because of the time constraint, and removing that assumption is v2.
- Planned shape: a desktop wrapper with a local SQLite store and sync.
- Source: architecture README, decision D9, recorded 24 November 2026.

### 2. Local store and sync engine

- v1 has no local database, no sync engine, no push queue and no merge. The browser keeps an IndexedDB spool for in-flight capture only, and it is never the record (D10). The step claim is decided online by the Hub, so there is no offline claim in v1 (D8).
- v2 adds the durable local store plus sync. Source: brief to developers, section 3, by decision D9.
- Six rules already bind v1 work now, so v2 stays an addition rather than a rewrite (architecture README):
  1. Client-generated ids and idempotent upserts keyed by (client id, revision), never server-assigned identity.
  2. Evidence content-addressed by hash, so a resent photo is deduplicated.
  3. Device id and a per-device sequence on every write, so offline events can be ordered later.
  4. Step state lives in the record, never only in a session.
  5. The API stays the only write path, with a small surface: claim, screenshot, submit.
  6. Evidence bytes stay out of the database.

### 3. Automated ACC upload

- v1 produces an upload-ready bundle per SAT (generated workbook, evidence under a fixed naming convention, manifest) and a person uploads it to ACC by hand. Acceptance stays "a form in ACC", achieved by a person clicking upload.
- Automated upload to ACC is v2. It removes service-account access negotiation, per-project field mapping and rejection handling from the December path.
- Sources: scope interview Q15, 18 November 2026; architecture README, "What the package deliberately leaves out".

### 4. Cloud deployment of the Hub

- v1 runs the Hub on a site laptop on the site LAN. v2 runs it as a cloud container. Nothing is rewritten when it moves.
- Source: scope interview, client-plus-server architecture answer, 18 November 2026.

### 5. M365 SSO

- v1 has one local account provider and no identity provider service. Accounts are seeded on the Hub and passwords are re-seeded if lost.
- v2 adds Entra ID over OIDC as a second provider behind the same seam: `identity(provider, subject, user_id)` already allows a person to hold both a local and an Entra identity, and the session is minted the same way. Routes and the capability check do not change.
- A self-hosted IdP (Keycloak, Authentik, Zitadel, Authelia) is only worth running when a second application needs the same login, or when a site must have SSO with no internet. Neither is true in v1.
- Source: architecture decision D17, 25 November 2026.

### 6. Postgres

- v1 uses SQLite behind a repository module, single process, WAL mode. There is no second Hub process and no horizontal scaling.
- v2 in the cloud needs Postgres: driver swap, a DDL migration (`AUTOINCREMENT`, `TEXT` timestamps, no `JSONB` in v1), a data move, and a migration run with the app stopped.
- Source: architecture decision D18, 25 November 2026.

### 7. RBAC and multi-project hosting

- v1 has one role (`engineer`), one SAT and one project per Hub. The `role` column and the `sat_id` columns exist so v2 adds data, not schema.
- v2 adds roles, tenant and project isolation, and the `/sat/{satId}/forms/...` route shape.
- Source: developer estimate note "no RBAC in v1"; scope interview Q9.

### 8. Documents and image presentation

- v1 generates one xlsx per form from a schema, one image per slot per revision, and accepts manual touch-ups before signing. `manual_file` evidence ships in the bundle and is never embedded.
- v2 candidates, all deliberately out: variable image counts per slot, print-perfect pagination, client branding, docx output, and embedding non-image files.
- Source: scope interview Q6, Q7 and Q19; architecture decision D16.

### 9. Live in-page camera

- v1 uses the device's own camera app through a file input with `capture`.
- v2 could add `getUserMedia` for an in-page preview, which requires HTTPS with a certificate the devices trust. It buys framing help, not new evidence.
- Source: architecture decision D14, 25 November 2026.

### 10. Acceptance record and e-signature

- v1 ends at a sealed bundle. Acceptance happens in ACC or on paper, outside the app, and v1 keeps no acceptance record.
- v2 candidates: a signature capture, a signed-copy return path, and an acceptance status in the audit trail. Each needs a legal answer, not just code.
- Source: architecture decision D20; scope interview Q8.

### 11. Retention and destruction

- v1 keeps every blob, every edition and every superseded copy. Nothing is deleted automatically.
- Open for v2 and for the client: how long a site dataset is kept, what leaves the site, what is destroyed, and who signs that off. The v1 position is the "keep all" line below.
- Source: architecture README "Still open", 25 November 2026.

### 12. WhatsApp automation

- v1 generates a daily summary as text, stores it, and a person copies it into their own WhatsApp. No Business API, no attachment, no recipient management.
- v2 candidates: automated delivery through a gateway, PDF attachment, recipient lists, and an approval position on whether the client's WhatsApp is an approved channel for project data.
- Source: scope interview Q17; architecture decision on the summary, 25 November 2026.

## Open, not yet assigned to a version

- **OCR + AI extraction of the SAT document** (optional scope, unscheduled). Scanned pages are OCR'd, a model reasons over the layout and text, and the result is mapped onto our fixed shapes: the form schema and the step list. Output is a candidate CSV plus schema JSON with provenance, committed by a person through the normal setup import. No database writes, no API calls, no exception to the one write path. Tasks F0–F6 in the [build plan](build-plan.md).
  - Why optional: if the client supplies a spreadsheet, cells can be read directly and none of this is needed. The quality bar also needs a labelled sample with hand-entered truth that does not exist yet, and the December path already carries the capture tool and the Wi-Fi assumption.
  - Open before promotion: who owns the quality threshold, which documents count as the labelled set, and whether this may appear in the demo story at all — Q11 and Q12 already ruled that the PoC demonstrates workflow, not AI.
- Real-time runtime screen capture. Open item from the 12 November 2026 alignment meeting with Liaw Jien Ming.
- AI-judged SAT and step selection, judging from the description instead of the user picking from a list. Under discussion; it risks errors and team collision. Off in the v1 cost model, where it costs nothing.
