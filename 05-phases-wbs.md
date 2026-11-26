# Phases and work breakdown

## Phases

| Phase | Outcome | Exit criteria | Includes |
|---|---|---|---|
| P1 | One step, one browser, one record: sign in, see the grid, claim, write, submit, and a claim race decided with a readable message | On the site laptop, two browsers claim one step: one gets 200, the other 409 naming the holder and expiry. A restart returns 401 on the next request. Every action leaves its audit row. | WP-1.1.1 – WP-1.1.6, WP-1.2.1 – WP-1.2.6, WP-1.2.9, WP-1.6.2, WP-1.6.3 |
| P2 | Capture on a tablet: a site photo through the device camera, a spool that survives a reload, evidence hashed and slot-bound, and a screenshot round trip against a stub | Five captures and two uploads on the tablet: 3000 px enforced, 8 MB plus one byte refused, identical bytes stored once, the spool restored after a reload, and a screenshot request reaching `expired` at 90 s with no job running | WP-1.3.1 – WP-1.3.7 |
| P3 | The standard format, the project's steps, a sealed form and an uploadable bundle | The standard format renders a workbook; an instrument-list spreadsheet imports as one step per row; the fixture form seals as edition 1; an edit returns 423; an amendment reopening one step produces exactly edition 2; the bundle opens in Excel and LibreOffice with no repair prompt and its manifest validates | WP-1.2.7, WP-1.2.8, WP-1.4.1 – WP-1.4.4, WP-1.5.4, WP-1.5.5, WP-1.5.8, WP-1.6.1 |
| P4 | Rehearsal with the real screenshot leg, backup and restore proven, and a checklist someone else can follow | One form goes from empty to a sealed edition and a bundle in one sitting against a non-production WinCC target, repeats the next morning after a restart, and a backup restores into a scratch folder with identical counts and hashes | WP-1.1.7, WP-1.1.8, WP-1.3.8, WP-1.5.1 – WP-1.5.3, WP-1.5.6, WP-1.6.4, WP-1.6.6 |
| P5 | Site day: the Hub installed on site, one real SAT captured, sealed, compiled, uploaded to ACC, and accepted by the signer | The signer accepts a bundle produced on site, with no builder present, and the rollback to the previous backup is demonstrated before the test starts | WP-1.5.7, WP-1.6.5 |
| P6 | Conditional: a scanned or PDF instrument list becomes a reviewed candidate that commits through the normal import | A labelled sample scores at or above a written threshold someone else agreed to, and a rejected row shows in the diff rather than reaching the record. Not started unless a scanned list actually arrives | WP-1.7.1 – WP-1.7.8 |

## WBS

```
1 SAT PoC v1
  1.1 Site platform and access
    WP-1.1.1 Repository, build and toolchain
    WP-1.1.2 Configuration constants module
    WP-1.1.3 Schema and migrations
    WP-1.1.4 Repository module and transactional transitions
    WP-1.1.5 Local accounts, sessions and capability check
    WP-1.1.6 Setup and seed command
    WP-1.1.7 Offline install package and Windows service
    WP-1.1.8 Backup and restore commands
  1.2 Step record and API
    WP-1.2.1 Error catalogue and response shape
    WP-1.2.2 Grid read model
    WP-1.2.3 Form read model and schema pinning
    WP-1.2.4 Claim, renew, release and derived expiry
    WP-1.2.5 Step write with revision and value validation
    WP-1.2.6 Submit and form-level ready
    WP-1.2.7 Seal and edition
    WP-1.2.8 Amendment and re-seal
    WP-1.2.9 Audit trail coverage
  1.3 Capture and evidence
    WP-1.3.1 Evidence upload, hash check and slot binding
    WP-1.3.2 Document derivative, orientation and format handling
    WP-1.3.3 Client capture, spool and picker
    WP-1.3.4 Client grid and step screen
    WP-1.3.5 Screenshot request and derived status
    WP-1.3.6 Service principal and tool endpoint
    WP-1.3.7 Stub capture tool for rehearsal
    WP-1.3.8 Capture tool change and handover
  1.4 Document and bundle
    WP-1.4.1 Template engine
    WP-1.4.2 Image placement and row geometry
    WP-1.4.3 Bundle builder, layout and manifest
    WP-1.4.4 Compile job and download
  1.5 Pilot operations and acceptance
    WP-1.5.1 Daily test summary
    WP-1.5.2 Rehearsal on a non-production target
    WP-1.5.3 Site-day checklist and runbook
    WP-1.5.4 Standard SAT format definition
    WP-1.5.5 Generality on a second project list
    WP-1.5.6 Photo and site-data position
    WP-1.5.7 Pilot qualification with the signing engineer
    WP-1.5.8 Instrument-list importer
  1.6 Site and platform readiness
    WP-1.6.1 Hub machine and fixed LAN address
    WP-1.6.2 Site network confirmation in writing
    WP-1.6.3 Site access, induction and dates
    WP-1.6.4 Backup drive and its custodian
    WP-1.6.5 Site install and pilot supervision
    WP-1.6.6 Capture tool go/no-go with the control room owner
  1.7 Document ingestion (conditional, P6)
    WP-1.7.1 Source routing: spreadsheet as cells, scans to OCR
    WP-1.7.2 OCR and layout extraction
    WP-1.7.3 Region segmentation pass
    WP-1.7.4 Schema mapping to the master record
    WP-1.7.5 Step-list candidate output
    WP-1.7.6 Review surface: accept, edit, reject
    WP-1.7.7 Labelled sample authoring
    WP-1.7.8 Score harness and written threshold
```

Children sum to the parent in every branch: 1.1 has eight leaves, 1.2 has nine, 1.3 has eight, 1.4 has four, 1.5 has eight, 1.6 has six, 1.7 has eight. That is 51 work packages, and nothing in the plan sits outside them. Branch 1.7 is conditional: it is planned, banded and gated, and it is not built unless a scanned instrument list arrives.

## WBS dictionary

| WP | Title | Traces | Owner | Estimate | Depends on | Phase | Acceptance |
|---|---|---|---|---|---|---|---|
| WP-1.1.1 | Repository, build and toolchain | R62 | either developer | 1-2 | — | P1 | a clean copy builds, lints and passes tests with the network cable out, and the built server starts and answers an unknown route with the one error shape |
| WP-1.1.2 | Configuration constants module | R46 | hub developer | 0.5-1 | WP-1.1.1 | P1 | a test fails on any numeric literal above 60 in a route or repository file, and changing the 8 MB cap in config changes the enforced limit |
| WP-1.1.3 | Schema and migrations | R49 | hub developer | 2-3 | WP-1.1.1 | P1 | an empty database migrates to the full schema, a second run is a no-op, and a second database from scratch produces an identical dump |
| WP-1.1.4 | Repository module and transactional transitions | R46,R47 | hub developer | 2-3 | WP-1.1.1,WP-1.1.3 | P1 | two concurrent claims give one grant and one 409, and a grep test finds no SQL outside the repository module |
| WP-1.1.5 | Local accounts, sessions and capability check | R1,R2,R3 | hub developer | 2-4 | WP-1.1.3,WP-1.1.4 | P1 | a restart returns 401 on the next request, an expired session returns 401, and no handler reads a role directly |
| WP-1.1.6 | Setup and seed command | R52 | hub developer | 1-2 | WP-1.1.5 | P1 | on an empty database it produces a SAT the grid can read, prints the tool token once, and refuses a second run without the force flag |
| WP-1.1.7 | Offline install package and Windows service | R44,R62 | hub developer | 2-4 | WP-1.1.1 | P4 | a clean Windows machine with no internet installs from the folder, serves the app over HTTP at the fixed address, and restarts after a reboot with no manual step |
| WP-1.1.8 | Backup and restore commands | R43 | hub developer | 1-2 | WP-1.1.3,WP-1.1.4 | P4 | a restore into a scratch folder yields identical counts and hashes, and a tampered evidence file is named by the manifest check |
| WP-1.2.1 | Error catalogue and response shape | R45 | hub developer | 1 | WP-1.1.1 | P1 | every code in the spec has a test asserting its status, and a forced exception returns the shape with no stack trace |
| WP-1.2.2 | Grid read model | R4,R9,R58 | hub developer | 1-2 | WP-1.1.4,WP-1.2.1 | P1 | a 100-step grid responds under 200 ms on the site laptop, and an expired lease reads open with no job running |
| WP-1.2.3 | Form read model and schema pinning | R5,R6 | hub developer | 1-2 | WP-1.1.6,WP-1.2.1 | P1 | a second form from a second schema row is served with no code change, and the response names the pinned version |
| WP-1.2.4 | Claim, renew, release and derived expiry | R7,R8,R9,R59 | hub developer | 2-3 | WP-1.2.2 | P1 | two browsers race and one loses with the holder and expiry named, renew moves expiry by 12 hours from the server clock, and release frees the step at once |
| WP-1.2.5 | Step write with revision and value validation | R10,R11,R12,R13,R48 | hub developer | 2-3 | WP-1.2.3,WP-1.2.4 | P1 | each refusal carries what is needed to act, a write from a non-holder is 409 not_claimed, and three accepted writes leave the revision at 4 |
| WP-1.2.6 | Submit and form-level ready | R14,R15,R16 | hub developer | 1 | WP-1.2.5 | P1 | submitting twice returns the same body, submitting at an old revision returns 409 with the current revision, flags name empty required columns and slots, and ready flips only on the last submitted step |
| WP-1.2.7 | Seal and edition | R17,R18,R19,R20 | hub developer | 2-3 | WP-1.2.6 | P3 | two seals of identical data give one edition and the same content_sha, a seal with a step open names that step, and a sealed write returns 423 |
| WP-1.2.8 | Amendment and re-seal | R19,R21,R22 | hub developer | 2-3 | WP-1.2.7 | P3 | only named steps reopen, re-seal mints exactly one edition n+1, and a drop after a write is refused with a code and a reason |
| WP-1.2.9 | Audit trail coverage | R41 | hub developer | 1 | WP-1.2.4 | P1 | one step driven end to end yields the ordered audit rows with server time and actor |
| WP-1.3.1 | Evidence upload, hash check and slot binding | R25,R26,R27,R49,R60 | hub developer | 2-3 | WP-1.1.4,WP-1.2.5 | P2 | 8 MB passes and 8 MB plus one byte is 413 before the body is read, identical bytes give 409 with one file named by its hash, and a second image for a slot is 422 |
| WP-1.3.2 | Document derivative, orientation and format handling | R28 | hub developer | 1-2 | WP-1.3.1 | P2 | a portrait phone photo is upright in original and copy, the copy regenerates from the original, and an unconvertible file is 415 with a readable message |
| WP-1.3.3 | Client capture, spool and picker | R33,R34 | client developer | 3-4 | WP-1.2.4 | P2 | five captures and two uploads leave the device, a reload before save restores the spool, a discarded shot appears in no request, and the camera opens with no permission prompt |
| WP-1.3.4 | Client grid and step screen | R35 | client developer | 2-3 | WP-1.2.2,WP-1.3.3 | P2 | a step held by another account renders locked with the holder named, and submit is not offered unless this account holds the lease |
| WP-1.3.5 | Screenshot request and derived status, Hub FIFO queue | R29,R30 | hub developer | 1-2 | WP-1.2.4,WP-1.3.1 | P2 | tool stopped still returns 202 then reads expired after 90 s, three concurrent go out in order, a full queue is 503 with no row, and a late post is 409 expired |
| WP-1.3.6 | Service principal and tool endpoint | R31 | hub developer | 1 | WP-1.1.6 | P2 | a wrong token is 401, the token appears in no log or error body, and disabling the principal stops the tool route and nothing else |
| WP-1.3.7 | Stub capture tool for rehearsal | [ENABLER] for WP-1.3.5 | either developer | 0.5-1 | WP-1.3.5 | P2 | WP-1.3.5 and WP-1.3.6 pass against the stub on a machine with no WinCC, and the stub is refused on a wrong token |
| WP-1.3.8 | Capture tool change and handover | R32,R56 | either developer | 2-3 | WP-1.3.5,WP-1.3.6 | P4 | on a target that is not the live control room PC, one request stores one screenshot and still leaves a local file, and a second person reviews the change |
| WP-1.4.1 | Template engine | R6,R36 | either developer | 3-5 | WP-1.2.3,WP-1.5.4 | P3 | a golden-file test renders the fixture twice byte-identically, and a second schema row renders a second layout with no code change |
| WP-1.4.2 | Image placement and row geometry | R37,R38 | either developer | 1-2 | WP-1.3.2,WP-1.4.1 | P3 | no image crosses a cell or a row, a missing slot leaves a blank cell and no shift, and the workbook opens in Excel and LibreOffice with no repair prompt |
| WP-1.4.3 | Bundle builder, layout and manifest | R39,R40 | either developer | 1-2 | WP-1.2.7,WP-1.4.2 | P3 | bundle files are named `<step_no>-<slot_id>-<short sha>.<ext>`, the manifest validates, every hash in it exists in the archive, manual files are shipped and never embedded, and the layout file alone changes the tree |
| WP-1.4.4 | Compile job and download | R23,R24 | hub developer | 1-2 | WP-1.4.3 | P3 | both refusals name the blocking form, the same idempotency key returns the same job, and a failed job leaves no partial archive |
| WP-1.5.1 | Daily test summary | R42 | hub developer | 0.5-1 | WP-1.2.2 | P4 | counts match the grid for the day, generating twice stores two retrievable rows, and the text names flagged steps rather than hiding them |
| WP-1.5.2 | Rehearsal on a non-production target | R51 | either developer | 2-4 | WP-1.1.7,WP-1.1.8,WP-1.3.4,WP-1.3.8,WP-1.4.4 | P4 | one form goes from empty to a sealed edition and a bundle in one sitting, repeats the next day after a restart, and every defect is logged against a WP |
| WP-1.5.3 | Site-day checklist and runbook | R50 | either developer | 1-2 | WP-1.5.2 | P4 | a person who did not build it follows the checklist without asking a question, and every step has a visible confirmation |
| WP-1.5.4 | Standard SAT format definition | R52 | engineer | 3-4 | — | P3 | the format is written down as one master record with columns, types, slots, sections and layout, and it renders the agreed specimen workbook |
| WP-1.5.5 | Generality on a second project list | R53,R6 | engineer | 2 | WP-1.4.2 | P3 | a second fixture instrument list renders on the same format with no code change, and the acceptance is written down with a date |
| WP-1.5.6 | Photo and site-data position | R61 | engineer | 0.5-1 | WP-1.3.2 | P4 | a written position says what the dataset contains, who takes it away and what is not deleted, and it ships with the checklist |
| WP-1.5.7 | Pilot qualification with the signing engineer | R57 | engineer | 3 | WP-1.5.2,WP-1.5.3,WP-1.6.1 | P5 | the checklist run on site produces a sealed bundle a person uploads, with no builder present |
| WP-1.5.8 | Instrument-list importer | R63 | either developer | 2-3 | WP-1.1.6 | P3 | a 30-row spreadsheet imports as 30 steps; a duplicate tag or a missing level is refused by name and the table is unchanged |
| WP-1.6.1 | Hub machine and fixed LAN address | R55 | platform | 2-3 | — | P3 | the machine and the address are named in the checklist, and a tablet reaches the app at that address |
| WP-1.6.2 | Site network confirmation in writing | R54 | platform | 0.5-1 | — | P1 | a message or document from the site network owner states whether client isolation is on, and it is quoted in 06 |
| WP-1.6.3 | Site access, induction and dates | [ENABLER] for WP-1.6.5 | platform | 1-2 | — | P1 | the site date, the induction requirement and the nominated signing engineer are named in writing |
| WP-1.6.4 | Backup drive and its custodian | [ENABLER] for WP-1.1.8 | platform | 0.5-1 | WP-1.1.8 | P4 | a named drive exists, a named person runs it, and the restore has been exercised once on that drive |
| WP-1.6.5 | Site install and pilot supervision | R44,R57 | platform | 2-3 | WP-1.1.7,WP-1.5.3,WP-1.6.1,WP-1.6.3 | P5 | the Hub installs on site, the run produces a sealed bundle, and a rollback to the previous backup is demonstrated before the test starts |
| WP-1.6.6 | Capture tool go/no-go with the control room owner | R56 | solution architect | 1-2 | WP-1.6.2 | P4 | one of the three outcomes is written down with a date, and the demo script matches it |
| WP-1.7.1 | Source routing: spreadsheet as cells, scans to OCR | R64 | either developer | 1-2 | WP-1.5.8 | P6 | an XLSX goes straight to the importer and is never OCR'd; a scan or PDF is routed to the OCR path and reports why |
| WP-1.7.2 | OCR and layout extraction | R65 | either developer | 3-4 | WP-1.7.1 | P6 | on the labelled sample, table rows come back in reading order with no merged or split row, and every token carries a confidence and a bounding box |
| WP-1.7.3 | Region segmentation pass | R65 | either developer | 3-4 | WP-1.7.2 | P6 | region roles are correct on the labelled sample, and an unlabelled region is reported rather than guessed |
| WP-1.7.4 | Schema mapping to the master record | R64 | either developer | 3-4 | WP-1.7.3,WP-1.5.4 | P6 | every proposed field cites its source column, required flags stay unset by default, and a wrong proposal shows in the diff rather than being applied |
| WP-1.7.5 | Step-list candidate output | R64 | either developer | 2-3 | WP-1.7.4 | P6 | output is a CSV that WP-1.5.8 accepts unchanged, and a duplicate tag is refused by the import rather than repaired |
| WP-1.7.6 | Review surface: accept, edit, reject | R64 | client developer | 3-5 | WP-1.7.5 | P6 | a reviewer clears a 30-row document without opening the original, and every rejection is recorded |
| WP-1.7.7 | Labelled sample authoring | R66 | engineer | 2-3 | — | P6 | a set of real pages with hand-entered truth exists on paper, covering both step tags and schema columns |
| WP-1.7.8 | Score harness and written threshold | R66 | either developer | 2-3 | WP-1.7.5,WP-1.7.7 | P6 | exact match, false add and false drop are reported per field, and the threshold is written and agreed by someone other than its author |

## Sequence

Dependencies only. No calendar dates.

- P1 must exit before P2 starts: the capture path writes through the step route and needs the lease.
- P2 must exit before P3's bundle work: the workbook places derivatives that only exist once upload works.
- P3 must exit before P4: the rehearsal exercises a real sealed form and a real bundle.
- P4 must exit before P5: the rehearsal is what makes the site day a test rather than a debugging session.
- WP-1.3.8 (the real capture tool) blocks WP-1.5.2 and therefore P5. It also carries `O4`, which is not in our control.
- WP-1.6.2 (`O3`, the Wi-Fi answer) blocks no P1 to P3 work package, gates WP-1.6.6, and blocks every tablet test on site. It is scheduled in P1 because it is a fact someone can check in an afternoon and it cannot be fixed later by writing code.
- WP-1.5.4 (the master record) blocks WP-1.4.1, which is the longest single piece. If the master record is not ready, WP-1.4.1 builds against a fixture schema and WP-1.5.5 becomes the only proof of generality.
- WP-1.6.1 (`O10`, the Hub machine) blocks WP-1.5.7 and WP-1.6.5, not the build.
- WP-1.1.8 (`O11`, the backup destination) can be built and tested into a scratch folder without the named drive.
- WP-1.5.8 blocks the site day. Without the instrument list in the record there are no steps to test, and it needs `O13` settled for any list whose tags conflict with the drawings.
- P6 is conditional and unordered against P1 to P5. If a scanned list arrives, WP-1.7.1 to WP-1.7.6 must produce a candidate that WP-1.5.8 accepts before the site day, and WP-1.7.7 and WP-1.7.8 must land before any candidate is trusted (R66). Nothing in P1 to P5 waits on P6.

## Estimate summary

| Phase | Work packages | Dev-days | Engineer-days | Platform-days |
|---|---|---|---|---|
| P1 | 15 | 17.5-28 | 0 | 1.5-3 |
| P2 | 7 | 10.5-16 | 0 | 0 |
| P3 | 10 | 12-20 | 5-6 | 2-3 |
| P4 | 9 | 8.5-16 | 0.5-1 | 1.5-3 |
| P5 | 2 | 0 | 3 | 2-3 |
| P6 | 8 | 17-25 | 2-3 | 0 |
| Total | 51 | 65.5-105 | 10.5-13 | 7-12 |
| Core only, P1-P5 | 43 | 48.5-80 | 8.5-10 | 7-12 |

Every cell is the sum of the work packages in that phase, at the band each package carries in the dictionary. No midpoint column: a rounded midpoint per row does not add up, and a total that disagrees with its own rows is worse than a band.

One cell carries the 2 developer-day testing proposal if `O8` is accepted: P4 dev-days becomes 10.5-18. It is not in the table above.

## Capacity check

At 4 productive days per FTE-week, the source package's own rate:

| Case | Dev-days | Weeks at 2.0 FTE | FTE needed in 5.0 weeks |
|---|---|---|---|
| Core, low band | 48.5 | 6.1 | 2.4 |
| Core, estimate | 64 | 8.0 | 3.2 |
| Core, high band | 80 | 10.0 | 4.0 |
| With P6 | 65.5-105 | 8.2-13.1 | 3.3-5.3 |

No case fits 2.0 FTE in the 25 working days the deadline allows, and the low band is the closest at 6.1 weeks. The developer is the binding constraint before platform is, which is the reverse of the source package, where 62 dev-days at 2.0 FTE needed 2.5 FTE against a 6.2 week window. The pack has 5.0 weeks and 64 core dev-days. That is `O15`, and it is a scope or capacity decision, not an estimate the two developers can narrow by working harder.
