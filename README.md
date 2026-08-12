# Impact Lab Wellington — Team 10: The Cherry Hackers

**Wellington City Council Emergency Management × Claude Code Community NZ**
Saturday 8 August 2026 · Wellington City Council

---

## Problem 04 — Help emergency staff sort and prioritise incoming information

> How might we help emergency staff rapidly sort incoming reports into information for awareness, information requiring verification, and information requiring action?

During an event, information arrives through phone calls, emails, forms, social media, news reports and partner agencies. The challenge is not just collecting it. Staff must identify where it relates to, whether it is new or duplicated, how reliable it may be, and whether it requires action.

A prototype could extract location, time, issue and potential urgency; group similar reports; preserve links to the original sources; and present a queue for human review.

**Desired outcome:** Staff spend less time sorting information and more time checking significant reports and coordinating action.

*The common theme is improving the flow and use of information between communities and Council before and during an event.*

---

## Our solution — Wellington Emergency Information Triage

A public report → an AI clarifier asks a follow-up question, then (after the
answer) suggests 1-2 actions → that triggers a check against 5 live official
NZ data sources for the same location → a second AI model triages the
report's severity in light of that official context → staff see one
prioritised, location-grouped item on a dashboard. Two small, fully
fine-tuned models (Phi-3.5-mini) do the clarify/action and triage work — no
third-party LLM API, no per-token cost, no external vendor dependency during
an actual emergency. This is a concept of a platform that aids organisations 
in their AI-powered transformations in a financially, socially, and
environmentally sustainable way.

*For an architecture diagram of this flow, see the demo video linked below.*

## See it live

- **Presentation deck** —
  https://docs.google.com/presentation/d/1hPS23Zs64ihz6spdi_oPm5G-IegdNTjKYh2JBO6PPtE/edit?usp=sharing
- **Demo video** — https://github.com/claudecommunity-nz/impact.lab.wlg.team-10.2026-08-08/raw/refs/heads/main/Video%20Summary.mp4
- **Proposed solution and rationale** — [`docs/PITCH.md`](docs/PITCH.md), the more detailed narrative and proposal
- Backend API — https://wellington-poller-ii3mghfupa-ts.a.run.app/events is
  the "common operating picture" feed other teams' prototypes can point at
  directly (`GET /events`, filterable by `suburb`, `hazard_type`,
  `source_type`)
- Frontend — deployed via GitHub Pages from this repo:
  - **Report a hazard** (public form) —
    https://claudecommunity-nz.github.io/impact.lab.wlg.team-10.2026-08-08/clarify=1
  - **Staff dashboard** (triage queue + map) —
    https://claudecommunity-nz.github.io/impact.lab.wlg.team-10.2026-08-08/dashboard/

## Getting started

**Run it locally:**

Backend (FastAPI + two in-process fine-tuned models):

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Frontend (SvelteKit, both the public form and the staff dashboard in one
project):

```bash
cd frontend
pnpm install
VITE_API_URL=http://localhost:8000 pnpm run dev
```

Then open `http://localhost:5173/` (report form) or
`http://localhost:5173/dashboard/` (staff dashboard).

**Project layout:**

```
backend/    FastAPI app, official-source pollers, fine-tuned model interfaces
frontend/   SvelteKit — public report form + staff dashboard, two routes, one project
docs/       Build plan, fine-tuning plan, frontend plan, solution pitch, known issues
```

**Docs:**

| File | What's in it |
|---|---|
| [`docs/PITCH.md`](docs/PITCH.md) | Pitch/demo narrative, including the live-demo script |
| [`docs/BUILD_PLAN.md`](docs/BUILD_PLAN.md) | System flow, API contract, event schema, deployment map, fine-tuning commands |
| [`docs/FINETUNE_PLAN.md`](docs/FINETUNE_PLAN.md) | What each model is trained on: instruction contract, dataset plan, taxonomy |
| [`docs/FRONTEND_PLAN.md`](docs/FRONTEND_PLAN.md) | The two frontends' design |
| [`docs/KNOWN_ISSUES.md`](docs/KNOWN_ISSUES.md) | Everything known to be wrong, missing or deliberately simplified, with status per item |
| [`docs/PROMPTS.md`](docs/PROMPTS.md) | Prompts used during the build |

## Data (provided by organisers)

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

## Ground rules

- These are **hazard-planning layers, not live emergency information**.
  In an emergency, call 111.
- **The data is not ours.** Each dataset belongs to its publisher — WCC, Greater
  Wellington, GNS Science, NIWA, Wellington Water, MBIE, NZTA, MetService.
  Licence terms vary per dataset; check the dataset's page before publishing
  anything derived from it, and credit the publisher.
- Be considerate with request rates. These are council servers, and at least one
  host throttles under concurrent load.
- **Keep personal details out of this repo.** It is public. No participant
  names, contact details or application material.
- Treat public social content as a *signal to investigate*, never as verified
  fact — surfacing something unverified as confirmed is the failure mode these
  problem statements are most wary of.

## Licence

Code here is MIT unless stated otherwise. The data is not covered by it.
