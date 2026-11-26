# Presentation set: 17 sequence diagrams

Standalone HTML for presenting. Not the deliverable and not sealed: the authoritative flows are the
tables and Mermaid blocks in [`../04-design.md`](../04-design.md). Three of the seventeen are the work
before site commissioning (S15, S16, S17). Where the two disagree, the document
wins and this folder is regenerated.

Open [`index.html`](index.html) to browse all seventeen. Each page has dark and light themes and a still
or live toggle.

| File | Flow | Routes covered |
|---|---|---|
| `s01-sign-in.html` | Sign in and carry a capability | `POST /auth/login`, `GET /me`, `POST /auth/logout` |
| `s02-morning-grid.html` | Open the app the next morning | `GET /sat/{satId}/grid`, claim |
| `s03-form-read.html` | Read a form and review its flags | `GET /forms/{formId}` |
| `s04-lease.html` | Lease lifecycle | claim, renew, release |
| `s05-evidence.html` | Capture to stored evidence | `POST /evidence`, `HEAD /evidence/{sha}`, `GET /evidence/{sha}` |
| `s06-step-write.html` | Write a step and survive the refusals | `PUT /steps/{stepId}` |
| `s07-submit.html` | Submit, flags, form-level ready | `POST /steps/{stepId}/submit` |
| `s08-seal.html` | Seal one form | `POST /forms/{formId}/seal` |
| `s09-amendment.html` | Amend, correct, re-seal, supersede | `POST /amendments` |
| `s10-screenshot.html` | The screenshot integration | the three screenshot routes |
| `s11-compile.html` | Compile and collect the bundle | `POST /compile`, job status, download |
| `s12-summary-backup.html` | Daily summary, backup and restore | `GET /summary/daily`, two commands |
| `s13-setup.html` | Install, accounts and the tool token | the setup command |
| `s14-site-day.html` | Site day | a person, the whole flow |
| `s15-format.html` | Define our standard SAT format | the setup import, schema pinning |
| `s16-instrument-list.html` | Instrument list to step list | the setup import, one step per row |
| `s17-ingestion.html` | Scanned list to candidate, then a person commits | conditional, gated |

## What is in this folder

- `*.html` — the seventeen diagrams, each one self-contained file.
- `index.html` — a landing page with all seventeen.
- `sources/*.sequence.json` — the archify input for each diagram. Edit here and regenerate.
- `.receipts/<slug>/` — the gate receipts: validate, deliver, strict check, and a real-browser viewport check.
- `.working/gen_sequences.py` — the generator that writes `sources/` and runs archify for every flow.

## Regenerating

Run from the pack folder:

```bash
python3 presentation/.working/gen_sequences.py
```

One flow only:

```bash
python3 presentation/.working/gen_sequences.py s09-amendment
```

All seventeen passed on archify 3.3.1: validate, deliver, strict check and a real-browser viewport
check, at `--quality showcase`. No warnings, and no `inspect-sequence-width` suggestion was raised.
None has been reviewed by eye — the checks above are automated.

## How these are built

They follow the 3.3.1 authoring defaults for a fresh Sequence, which is why the generator is small and
the JSON carries no geometry beyond message order:

- **No `meta.viewBox`, no `meta.column_fit`.** The renderer measures the timeline and spreads the
  participant columns, so a long flow grows downwards and a wide one widens up to the readable limit.
  The measured canvases here range from 920×501 to 1019×840.
- **No `meta.animation`.** Live motion is on by default; the Viewer's Live/Still button controls it.
- **28px base message spacing**, and `44 / 55 / 66 / 77`px after a one, two, three or four line note.
  The line estimate narrows as participant columns get tighter, which is why the generator scales
  characters-per-line by participant count.
- **No self-messages.** The renderer only draws between distinct participants, so an internal step
  becomes a note on the surrounding real message, or a card.
- **`meta.parentHref: "index.html"`**, so every diagram carries a Back control to the landing page.
- **`meta.locale: "en"`**, so the Viewer UI and the authored text agree.

Three reader-only fields are used where they add something the timeline cannot show:

- `payload` on the key writes (`POST /auth/login`, `POST /evidence`, `PUT /steps`, `POST /amendments`,
  `seal`, and the `form_amended` refusal). It appears in the Semantic Passport, never in the SVG.
- `explanation` on the record store, the evidence store, the spool and the capture tool.
- `cards`, two per diagram, carrying the branch detail that the canvas has no room for.

## Editing rules that still bite

- **Notes need vertical room.** A note extends below its message, and how far it wraps depends on the
  final lane width. If you add a note, expect the next message to move down.
- **Order is meaning.** Segment bounds and activation bounds are computed from message positions in
  `gen_sequences.py`; if you hand-edit `sources/*.json`, move them together.
