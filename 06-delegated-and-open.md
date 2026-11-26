# Delegated decisions and open questions

## Delegated

Decisions this pack deliberately does not make. Each one changes from project to project, costs little to reverse, and does not cross a contract boundary.

| ID | Decision | Decides | Why delegated | By phase | Impact if wrong |
|---|---|---|---|---|---|
| D1 | Module and file layout inside the four top-level folders | receiving architect | internal, invisible from outside, and their code to organise | P1 | contained in one phase; a move later is mechanical |
| D2 | Test strategy, fixture shape and coverage bar | receiving developers | their quality bar and their runner | P1 | a thin suite shows up as defects in P4, not as a wrong design |
| D3 | HTTP framework internals within Fastify: plugins, schema validation, serialiser choices | either developer | their maintenance burden; the wire contract is pinned in `03` | P1 | none outside their code, because routes and payloads are pinned |
| D4 | Sheet geometry constants: row heights, column widths, header treatment inside `layout` | either developer | a layout value, cheap to change, and the signer sees the result | P3 | one config or constant edit, then re-render |
| D5 | Log format, metric names and internal error text | hub developer | observability conventions belong to whoever operates it | P1 | log parsing breaks; no record or contract moves |
| D6 | Client component structure, CSS and the layout of the grid and step screen | client developer | their product surface; `04` fixes controls and states, not pixels | P2 | rework inside the client only |
| D7 | CI wiring, release packaging steps and how the install folder is produced | platform | their pipeline and their deployment habit | P1 | a slower build, no change to what ships |
| D8 | User-facing copy in the app, excluding anything the generated document must state | client developer | their product owner's call, unless a word is legally load-bearing | P2 | text edits only |

What is deliberately not delegated, because reversing it would cost more than about five days or would cross a boundary: the API contract, the data model and retention position, the auth model, the screenshot two-leg boundary, the seal and amendment semantics, the image caps, the manifest contents, the stack, and the licence choices. Those are pinned in `03`.

## Open

| ID | Question | Owner | Needed by | Blocks |
|---|---|---|---|---|
| O1 | Who accepts each phase, and who is the client's nominated signing engineer | requester | 2026-12-01 | handoff; the pack cannot be sealed |
| O2 | Who approves a scope change after the seal | requester | 2026-12-01 | handoff; the pack cannot be sealed |
| O3 | Is Wi-Fi client isolation switched on, yes or no, in writing from the site network owner. The answer is pasted into this row when it arrives | site network owner | 2026-12-05 | WP-1.6.2, WP-1.5.2, P5. No code fallback exists |
| O4 | Delivery date for the capture tool change, and a test target that is not the live control room PC | Jien Ming | 2026-12-08 | WP-1.3.8, WP-1.6.6, P4 exit, P5 |
| O5 | Confirmation of the ACC bundle tree and file naming, from whoever uploads | ACC uploader | 2026-12-10 | WP-1.4.3 quality, not its build |
| O7 | Is platform capacity raised to about 0.5 FTE | delivery lead | 2026-11-28 | every phase; at 0.3 FTE platform owns over 8 weeks of work in a 5.0 week window |
| O8 | Is the 2 developer-day testing proposal accepted, adding work to WP-1.5.2 | requester | 2026-12-01 | WP-1.5.2 estimate only |
| O10 | Which machine is the Hub, and what fixed LAN address do the tablets use | YNY | 2026-12-12 | WP-1.6.1, WP-1.5.7, WP-1.6.5 |
| O11 | Which backup drive, who runs it, and where it is stored | YNY | 2026-12-12 | WP-1.6.4; WP-1.1.8 can still be tested into a scratch folder |
| O12 | Retention and destruction of the site dataset: who takes it away, what is destroyed, who signs that off | client | 2026-12-20 | WP-1.5.6 only. v1 keeps everything until answered (R61) |
| O13 | Which document wins when the instrument list and the drawings disagree on a tag, its range, or its level | engineer | 2026-12-05 | WP-1.5.8, and the step list the site day runs on |
| O14 | Only if P6 is triggered: who owns the quality threshold and the labelled sample | requester | 2026-12-15 | WP-1.7.7, WP-1.7.8 |
| O15 | Is developer capacity raised to about 3.2 FTE, or is scope cut to about 48 dev-days. The core estimate is 64 dev-days against 25 working days at 2.0 FTE, which is 8.0 weeks of work in a 5.0 week window | delivery lead | 2026-11-28 | every phase; the date is not reachable at 2.0 FTE in any band |

Three of these are worth stating twice, because they are the ones a reader will want to solve by writing code:

- **O3 is a fact, not a task.** If the tablets cannot reach the Hub, the pilot does not run and there is no design change that saves it. It takes an afternoon to answer and it should be answered first.
- **O15 is arithmetic, not pessimism.** 64 core dev-days against 25 working days at 2.0 FTE is 8.0 weeks of work in a 5.0 week window, and the 48.5-day low band still needs 6.1 weeks. No plan, tool or agent closes that gap. It is answered by raising capacity to about 3.2 FTE or by cutting scope to about 48 dev-days, and it should be answered before `P1` starts rather than discovered in `P4`.

## What is not verified

No technical owner has reviewed this pack, so no claim here is stronger than its cited source. Where the sources disagree the interface spec wins. Where the sources are silent, the gap is an `O#` above or a delegated decision, never an invented answer.

Three gaps, stated plainly:

- **The instrument list has never been seen.** No real list is in this folder, so its column names, level coding and duplicate-tag habits are unknown. That is `O13`, on the critical path at `WP-1.5.8`.
- **No diagram has been reviewed by eye.** That is true of the nine in the architecture package and the seventeen in `presentation/`; their gates are automated. Nothing in this pack is derived from a diagram.
- **The estimate has not been validated by the two developers.** The handoff acceptance below is where that gets checked.

## Change control

After the seal, this pack is read-only. No document is edited, not by the skill, not by the requester, not to fix a typo.

A change is a new `07-amendment-NN.md` that cites the `R#` and `WP-` it alters, states the reason and the estimate delta, and names the approval authority from `01 § Authority`. Re-sealing with an amendment bumps the version to 1.1 and re-runs the seal over the numbered documents including the amendment. If the validator reports a stale seal, the correct action is an amendment, never a re-seal of the edit.

This pack is not sealed. That is deliberate: `O1` and `O2` are unnamed people, and a seal over a document with no change authority turns the seal into a suggestion.

## Handoff acceptance

The receiving team confirms, in writing:

- every `R#` is understood and traceable to at least one `WP-`;
- every `O#` blocking a phase they are starting has an answer or an accepted risk;
- the estimate band is their band, not ours — disagreement is recorded here, not silently absorbed;
- The stack is TypeScript; `WP-1.1.1` starts on that basis.

Until those four lines exist, this pack is a proposal. The architecture package says the same about itself: "draft for review, not agreed with the developers."
