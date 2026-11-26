---
pack: sat-poc-v1
title: SAT PoC v1 — site capture to uploadable bundle
status: draft
version: 1.0
date: 2026-11-26
source_artifacts:
  - architecture/README.md (decisions D1-D20, estimate, deferred list)
  - architecture/architecture-v1-interface-spec.md (routes, payloads, schema, build order)
  - architecture/build-plan.md (task-level done-when checks)
  - architecture/decisions-and-open-items.md (twelve items deferred to v2)
  - Prior inputs (scope interview, developer estimate V2) retired; content folded into 01, 02 and 05
  - Standard SAT format (A11) and spreadsheet-first instrument list (A12), agreed 2026-11-26
---

# SAT PoC v1 — site capture to uploadable bundle

**Summary (30 seconds):** One site, one SAT, the Cx2 and Cx3 forms. An engineer on a tablet signs in to a Hub on the site LAN, claims a step, takes a site photo, asks the control room for a SCADA screenshot, types the readings, and submits. When every step of a form is submitted a person seals that form; a compile then packages the sealed forms, the workbooks and the evidence into one bundle a person uploads to ACC by hand. It proves the workflow with no internet and no AI in the loop.

## Trace summary

- Requirements: `66` · covered: `66` · uncovered: `0`
- Work packages: `51` · enablers: `3` · estimates: `65.5–105` dev-days (about `85`), of which `17–25` is conditional P6 work; the core commitment is `48.5–80` (about `64`), plus `10.5–13` engineer-days and `7–12` platform-days
- Capacity: 2.0 developer FTE gives about 40 dev-days in the 25 working days available. Even the core low band needs 6.1 weeks, so the date is not reachable at 2.0 FTE. See `05 § Capacity check` and `O15`
- Assumptions: `11` · Open questions: `13` · Delegated decisions: `8`
- Authority: this plan is the single source of truth. On scope, schedule and estimates this plan applies; on routes and payloads the interface spec's wire text applies.
