# Impact Lab Wellington — Team 10

Context for Claude Code working in this repo.

## The event

A one-day build with Wellington City Council Emergency Management, Saturday
8 August 2026, at Wellington City Council. Ten teams, five problem
statements, two teams per statement. Each team shipped one working
prototype and demoed it for four minutes.

See `docs/PITCH.md` for the solution pitch and demo narrative.

## This team's problem — 04: Help emergency staff sort and prioritise incoming information

> How might we help emergency staff rapidly sort incoming reports into information for awareness, information requiring verification, and information requiring action?

During an event, information arrives through phone calls, emails, forms, social media, news reports and partner agencies. The challenge is not just collecting it. Staff must identify where it relates to, whether it is new or duplicated, how reliable it may be, and whether it requires action.

A prototype could extract location, time, issue and potential urgency; group similar reports; preserve links to the original sources; and present a queue for human review.

**Desired outcome:** Staff spend less time sorting information and more time checking significant reports and coordinating action.

All five statements sit inside one frame: the common theme is improving the flow and use of information between communities and Council before and during an event.

## What success looks like

Each prototype is a module in a shared **common operating picture**: a live map
of emergency signals. Prefer outputs that compose — GeoJSON, a feed, an
endpoint — over a self-contained UI that nothing else can read.

Judging is on a four-minute demo. Something running and pointed at real
Wellington data will land better than architecture that isn't finished.

## Our solution — Wellington Emergency Information Triage

Public reports a hazard → an AI clarifier asks a follow-up question, then
suggests 1-2 actions → that triggers a check against 5 live official NZ data
sources for the same location → a second AI model triages the report's
severity in light of that official context → staff see one prioritised,
location-grouped item on a dashboard. `GET /events` is the "common operating
picture" endpoint other teams' prototypes can point at directly. See
`docs/BUILD_PLAN.md` ("System summary", "Public-submission-triggered flow")
for the full flow, and `docs/FINETUNE_PLAN.md` for what each model is
actually trained on.

**Two fine-tuned models, not one, and not adapters swapped on a shared
base** — two fully-fused GGUF models loaded simultaneously in one Cloud Run
service:
- **Clarifier**: one model, two inference calls (task-conditioned via system
  prompt, not two models) — Call 1 asks a question, Call 2 (after the
  answer) suggests actions. See `docs/FINETUNE_PLAN.md` "Model 1 —
  Clarifier".
- **Triage Classifier**: outputs *only* `severity` + `rationale`.
  `hazard_type` is always deterministic (set by code from the matched
  official source), never model output. See `docs/FINETUNE_PLAN.md` "Model
  2 — Triage Classifier" for why.

## Tech stack

- **Backend**: Python, FastAPI, `httpx` for source fetching, Pydantic
  schemas, `llama-cpp-python` for in-process model inference. No database —
  in-memory store, ephemeral by design for this prototype.
- **Frontend**: Svelte 5 (runes) + Tailwind + SvelteKit (`adapter-static`,
  prerendered — not a plain SPA, see `docs/FRONTEND_PLAN.md` for why). One
  project, two routes: the public report form and the staff dashboard.
- **Fine-tuning**: MLX (`mlx_lm.lora`) on Apple Silicon, base model
  `microsoft/Phi-3.5-mini-instruct`, fused and quantized (GGUF, Q4_K_M) for
  portable CPU serving.
- **Deployment**: backend on Google Cloud Run; frontend on **GitHub Pages**,
  deployed from this repo.

## Key commands

```bash
cd backend && uvicorn app.main:app --reload --port 8000       # run backend locally
cd frontend && VITE_API_URL=http://localhost:8000 pnpm run dev # run frontend locally
.venv/bin/python3 backend/scripts/generate_triage_dataset.py   # regenerate Triage Classifier dataset
.venv/bin/python3 backend/scripts/generate_clarifier_dataset.py # regenerate Clarifier dataset
```

Fine-tuning commands (run manually, one per terminal, not automated) are in
`docs/BUILD_PLAN.md` "Fine-tuning commands" — see the `fine-tune-adapter`
skill for the venv caveat.

## Data

The public GIS datasets Wellington City Council Emergency Management shared are
catalogued, checked and made queryable here:

- **Catalogue + SDK** — https://github.com/claudecommunity-nz/wcc-emergency-gis-data
- **Browse the datasets** — https://claudecommunity-nz.github.io/wcc-emergency-gis-data/

74 datasets: flood, landslide, earthquake, tsunami, coastal inundation and
climate layers, plus emergency hubs, post-quake road reopening order, water
tanks, deprivation by area, and live river-level and rainfall telemetry.
`wcc_gis.py` is a single file with no dependencies — copy it and
`catalogue.json` into your project.

```python
import wcc_gis

wcc_gis.ids("tsunami")                                    # find datasets
wcc_gis.features("tsunami-evacuation-zones", at=(-41.2790, 174.7804))
wcc_gis.geojson("footpaths", bbox=wcc_gis.WELLINGTON)     # straight into MapLibre
wcc_gis.hilltop_data("Hutt River at Taita Gorge", "Flow")[-1]
```

Three traps worth knowing before you lose an hour to them:

- Everything is published in **NZTM2000, not lat/lng**. Request raw and your
  pins land off the coast of Africa. Always ask for `outSR=4326`.
- **A quarter of the layers are rasters** that advertise a query capability,
  then refuse to answer. Ask them for a PNG instead.
- **One query is silently capped** (`footpaths` has 8,130 features; a request
  returns 2,000). Page properly, or check `exceededTransferLimit`.

**⚠️ Worth a second look before final submission:** our actual official-source
integration (`backend/app/sources/`) queries GeoNet, MetService, NZTA, NEMA,
and GWRC's live APIs directly via `httpx`, not through `wcc_gis.py` — this
was built before checking whether the catalogue covers the same live-feed
use case. It might not (real-time earthquake/road/weather feeds vs. this
catalogue's GIS layers may be genuinely different data), but this hasn't
been explicitly confirmed either way — check before claiming we're not
using the provided catalogue, rather than assuming.

## Architecture conventions

- **Deterministic vs. model-inferred, kept strictly separate.** `hazard_type`
  is always code, never a model. Only judgement calls that genuinely need
  inference (is this report serious, what question should we ask) go to a
  model. See `backend/app/context.py`'s docstring.
- **Canonical render functions.** The exact same function that renders
  context into text for training data must be used at live-serving time
  (`context.render_context_text()`, `classifier.build_user_message()`,
  `clarifier.build_act_user_message()`) — never hand-type an approximation
  in two places.
- **Every new official data source gets verified against live data before
  its rules are trusted** — not assumed from documentation. See
  `.claude/skills/add-data-source`.
- **CSV-driven training datasets, not hand-authored JSONL.** Non-engineers
  can add rows directly; a generation script builds the actual JSONL via the
  canonical functions above and round-trip validates every row before
  accepting it. See `.claude/skills/fine-tune-adapter`.

## Important paths

- `docs/BUILD_PLAN.md` — the master build plan: system flow, API contract,
  event schema, hour-by-hour schedule, deployment map, fine-tuning commands
- `docs/FINETUNE_PLAN.md` — what each model is trained on: instruction
  contract, dataset plan, taxonomy, official-context format
- `docs/FRONTEND_PLAN.md` — the two frontends' design
- `docs/PITCH.md` — the pitch/demo narrative, including the live-demo script
- `docs/KNOWN_ISSUES.md` — everything known to be wrong, missing or
  deliberately simplified, with status per item
- `backend/app/sources/` — one module per official data source
- `backend/app/context.py` — the aggregator (deterministic context-building)
- `backend/app/classifier.py` / `backend/app/clarifier.py` — the two
  models' interfaces
- `backend/data/*_examples.csv` + `backend/scripts/generate_*_dataset.py` —
  training data source + generation

## Constraints that matter here

- **Hazard-planning data, not live emergency information.** Nothing built
  today should be presented as an operational emergency source. In an
  emergency, 111. This applies to the GIS catalogue's layers *and* to
  anything this system infers or aggregates from public reports.
- **Show reliability, don't hide it.** If the prototype infers or
  aggregates, say so in the interface — the staff dashboard's official
  context and plain-language rationale exist specifically for this. Never
  present an unverified public report as confirmed fact.
- **This repo is public and must stay free of personal information** — no
  participant names, contact details, or anything from the application
  process. Note this is a different, stricter bar than the in-app `contact`
  field submitters can optionally provide (that's user data in the running
  system, not repo content).
- **Attribution.** Data belongs to its publishers and licences vary per
  dataset — WCC, Greater Wellington, GNS Science, NIWA, Wellington Water,
  MBIE, NZTA, MetService, and (for our own direct integrations) GeoNet and
  NEMA. Check before publishing anything derived.
- Be considerate with request rates. These are council servers, and at
  least one host throttles under concurrent load.
- Treat public social content as a *signal to investigate*, never as
  verified fact.

## Skills

- `.claude/skills/add-data-source` — adding/modifying an official-source
  poller
- `.claude/skills/fine-tune-adapter` — preparing data for, training, or
  exporting either model

## Conventions

- Keep the README's problem statement in sync if the scope shifts during the day.
- Commit early and often — the repo is the submission.
