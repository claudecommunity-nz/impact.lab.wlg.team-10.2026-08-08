# Pitch — Wellington Emergency Information Triage

Only a 2-minute video and this repo are submitted; no slide deck required. The
"slides" below are internal working content, reused for the video script and
live-demo prep.

---

## Slide 1 — Title

**Wellington Emergency Information Triage**
*Turning public information into a working triage tool for emergency staff — before and during an emergency.*

Team 10

---

## Slide 2 — The problem

- WCC's emergency management team has run **10 activations in the past two years**
- During an activation, information arrives from many places at once — official warnings, sensor data, road incidents, community reports — in different formats, at different speeds, with different trust levels
- Staff don't have time to manually read, cross-reference, and prioritise every incoming item mid-emergency
- **Emergency decisions are only as good as the information flow behind them** — a triage and efficiency problem, not a data-access problem

**Track:** "Help emergency staff sort and prioritise incoming information" — every public report is checked against live official data and triaged by severity, turning a raw, unverified message into a prioritised, location-grouped item staff can act on.

**Also contributes to:** "Create a two-way information channel between communities and Council" and "Identify and verify emerging local impacts from public information" — every submission gets an immediate AI-generated follow-up question and proposed action, a two-way exchange at scale, not a one-way form drop.

**Solution:** report → clarifying follow-up + potential action → checked against live official sources for that location → one triaged, prioritised item for staff.

---

## Slide 3 — Lean Canvas at a glance

| Block | Content |
|---|---|
| **Problem** | Fragmented, high-volume, mixed-trust information during an activation; no fast way to sort it; no reliable two-way channel to trusted community groups |
| **Whose problem is it** | **WCC emergency management staff** during an activation. Secondary: **trusted community anchors** — marae, community centres, patrol groups — with no low-effort way to feed local information back to Council |
| **Existing alternatives** | Manually monitoring websites/feeds/radio/phone/email/social media; GeoNet; MetService App; Emergency Mobile Alert (broadcast-only, one-way); NZ Flood Pics |
| **Unique value proposition** | A living, prioritised triage feed from Wellington's own open data plus small custom-trained AI — staff spend time deciding, not searching. A foundation to build more capability on |
| **Solution** | Public report → AI clarify → live official-data check → AI triage → prioritised staff dashboard grouped by location; roadmap to offline-capable hardware nodes |
| **Channels** | Staff: web dashboard, no install. Community reporters: a simple web form today; SMS/radio hardware nodes in the resilience-layer roadmap |
| **Key metrics** | Time from submission → triaged; % of triage judgements staff confirm correct; official sources checked per submission; % of submissions receiving a clarifying follow-up (target: 100%) |
| **Unfair advantage** | A proven fine-tune-to-deploy pipeline, tested end-to-end on real infrastructure; a small-model architecture with no per-token cost or vendor lock-in |
| **Revenue streams** | WCC as founding customer (pilot); then licence to other NZ councils (the national-data layer and AI adapters already generalise — see Slide 10); longer-term, other cities/regions |
| **Cost structure** | One-time fine-tuning compute; free-tier hosting (Google Cloud Run + GitHub Pages); scales on demand; future cost is mainly the optional hardware resilience layer |

---

## Slide 4 — What we built

Every public report is instantly followed up on, checked against live official data, and turned into a triaged, prioritised item for staff — two small custom-trained AI models doing the work, on infrastructure that costs nothing at pilot scale.

- Public report form: submission → AI clarifying question → AI proposed action → checked against 5 live official sources → triaged automatically
- Two custom fine-tuned LLMs doing the clarify, action, and triage work
- Staff dashboard: prioritised feed grouped by location, map, detail view showing exactly which official data informed each judgement — reasoning shown, not hidden
- Middleware: polling, aggregating, normalising

---

## Slide 5 — Data flow

[Diagram: public report → clarify LLM question → answer → clarify LLM action → official-source poll (cached/throttled) → aggregate/normalise → classifier LLM → store → dashboard]

**The pipeline starts with a public report, not a background feed:**
1. A member of the public submits a report
2. The **clarifier** immediately asks a follow-up question — every submission, not just ambiguous ones
3. Once answered, the clarifier runs again and suggests 1-2 next steps, shown directly on the form
4. That exchange **triggers a check against live official data** for the location — also what keeps free-tier hosting warm only when needed
5. Aggregation pulls the officially-relevant context for that location
6. The **classifier** takes the clarified report plus that context and produces one triage judgement — genuinely new information, not a restatement of the official sources
7. Staff see one prioritised, location-grouped item, with the official context that informed it fully visible

**Real official sources checked against (live, not simulated):**
- MetService CAP feed — severe weather warnings (same feed Civil Defence uses)
- GeoNet — earthquake and volcanic alert data
- NZTA — real-time road incidents and closures
- NEMA Emergency Mobile Alert CAP polygons — broadcast areas of declared civil defence emergencies
- GWRC — live river level, flow, and rainfall telemetry

**Simulated:** public reports — no live public channel exists yet for this hackathon build, so submissions are simulated to demonstrate the clarify/triage pipeline (see roadmap).

**Why triage the report, not the official sources directly:** each official source already publishes its own severity signal — reclassifying that would just be a slower copy of free information. The hard, valuable problem is judging an *unverified public claim* against what's officially known — that's the only thing the classifier does.

---

## Slide 6 — Channels

- **Emergency staff (primary):** open a URL, no install. Sits alongside existing tools; long-term target is pushing alerts into whatever system WREMO staff already use, not a new screen to watch
- **Community reporters, today:** a simple public web form, low friction, works on any phone
- **Community reporters, future:** low-bandwidth hardware nodes (radio/SMS) at trusted anchor sites for when the web form isn't reachable
- **Council decision-makers, future:** a backup low-bandwidth device surfacing only the highest-severity items when the primary dashboard is degraded

---

## Slide 7 — The two custom LLMs

**Why custom fine-tuned models, not a generic LLM API call:**

1. **Small and efficient** — 3-4B parameters on low-cost CPU hosting; no per-token cost, no rate limits, no vendor uptime dependency during an emergency
2. **Purpose-built** — each fine-tuned for one narrow job; adding capability means fine-tuning another small model on the same pipeline, not re-architecting
3. **Auditable** — narrow, testable, traceable — matters when output feeds a real emergency decision
4. **Works with degraded connectivity** — self-hosted, no live third-party dependency
5. **Socially sustainable** — small enough for staff to fine-tune themselves, building real in-house AI skills rather than depending on a black box
6. **Efficient frontend too** — both apps are Svelte, not React: compiles away the framework instead of shipping a virtual-DOM runtime, so smaller downloads and less client-side compute on every device, including low-end phones

**Model 1 — Clarify and action:** two steps, every submission (not just ambiguous ones) — a follow-up question immediately, then 1-2 suggested next steps once answered, shown directly on the form. *(Limit: suggestions aren't yet checked against official data — a dedicated high-stakes-suggestion model informed by official corroboration is on the roadmap.)*

**Model 2 — Triage:** takes the clarified report plus relevant official data plus any related nearby report, outputs a severity level and one-line rationale — nothing else. Hazard type is deterministic, not model output. Never used to re-classify official sources — they already publish their own severity; the AI is reserved for judging an unverified public claim against official and community reality, reasoning shown in plain language.

*(Demo: submit a report live, watch the question appear, answer it, watch the action appear, then the triaged result checked against real official data.)*

---

## Slide 8 — What makes this credible

- **Usefulness to WCC:** every report gets prioritised against real official data; the two-way channel runs on every submission, not just ambiguous ones
- **Working with supplied data:** built against 5 real live feeds
- **Working demo:** live clarify + live triage against real data, not a screenshot
- **Honesty about limits:** public reports are simulated (no live public channel exists yet) — and why
- **Credible path to real use:** roadmap steps are small, concrete, tied to specific identified gaps
- **Modular:** data sources, frontends, and AI capability can each be extended independently

---

## Slide 9 — Roadmap

**2 weeks:** harden the prototype — a labeled evaluation set for the classifier, duplicate/cluster detection, move off in-memory storage, a test → demo → production release cadence.

**1 month:** add a **confidence signal alongside severity** — the classifier currently outputs only severity + rationale; adding confidence (how vague the report is, how strong the corroboration) lets the two combine into one **priority score**. A high-severity, high-confidence report surfaces first; a high-severity, low-confidence one gets flagged for human review rather than auto-elevated or silently trusted. Same model, same pipeline, one more output field — not a new architecture.

**2-3 months:** pilot with WCC — a real public-submission channel (social media as a second input), a manual staff trigger for pulling official data on demand, retrain on real pilot data, staff login. Also turns the report form into an **installable PWA with push notifications**, letting staff notify a **single submitter** (closing the loop on their report) or a **group of users** in an affected area — a two-way, targeted complement to NEMA's one-way Emergency Mobile Alert.

**4 months:** split out a **dedicated third model for high-stakes actions** ("call 111", "evacuate") — the hackathon build combines these with low-stakes suggestions before any official corroboration exists; a pilot needs that judgement moved to *after* the classifier's context-informed severity check, validated by WCC's domain expert.

**5-6 months:** production readiness — move off free-tier hosting, staff-briefing summarisation, plain-language public alert adapters, integrate into WREMO's existing operational systems.

**8 months:** scale — 1-2 more multi-language models (te reo Māori, Samoan, and others), expand beyond Wellington City.

**10+ months — resilience layer (beyond-internet hardware):** the system above assumes connectivity; real emergencies often don't have it.
- **Community input nodes** — LoRa/mesh or SMS-gateway units at trusted community anchors, so reports can be submitted when internet/cell is down
- **Council backup dashboard device** — a low-bandwidth receiver surfacing only critical/high-severity items when the web dashboard's connectivity path is down
- A **future track, not a hackathon-day build** — needs hardware procurement, field logistics, and network partnerships a one-day team can't validate. Shown in the demo as a **simulated UI concept only**.

Each step tends to add one capability to the same proven pipeline — not a rebuild.

---

## Slide 10 — Scalability: beyond Wellington

**Yes, in tiers**

- **National — near drop-in:** MetService CAP, GeoNet, NZTA TREIS, and NEMA's alert feed are NZ-wide already; any council plugs into the same four sources today
- **Regional — config, not redesign:** river/rainfall gauges and local hazard layers differ per region, but most NZ councils publish through the same ArcGIS Hub pattern GWRC/WCC use — adding a council mostly means pointing at their endpoint and remapping fields
- **The two LLMs — retrain, not rebuild:** hazard classification and clarification are generic tasks; scaling to another region means retraining on that region's suburb names and activation history — hours, not a new architecture
- **International — architecture transfers, data doesn't:** CAP is an international standard, so warnings ingestion could plug into another country's feed; GeoNet/NZTA/the regional-gauge pattern are NZ-specific and would need local equivalents

---

## Slide 11 — Ask / close

- What we'd want from WCC: access to real (even historical/anonymised) report data, and time with a domain expert to validate our classification categories against their activation taxonomy
- Repo link + demo video link
- Thank you

---

## Demo video storyboard (2 minutes)

Almost entirely live product footage, not slides — two brief diagram inserts (architecture, data flow) are the exceptions.

**Points to call out in narration** (point 5 is an "if it fits" addition):

1. **AI-first development** — reusable Claude skills, Claude Projects, custom MCP servers. (0:15–0:25)
2. **Security and data sovereignty** — small, self-hosted fine-tuned models, not a third-party API; sensitive data never leaves our environment. (1:35–1:50)
3. **Financially and environmentally sustainable** — no per-token cost, lower compute/energy than a large third-party model — including the frontend, Svelte instead of React. (1:35–1:50, Svelte detail cuttable if the beat runs long)
4. **Release cadence roadmap** — next step is a test → demo → production pipeline. (1:50–2:00)
5. **Socially sustainable** — small enough for staff to fine-tune in-house, building transferable skills rather than depending on a black box. (1:50–2:00, cut first if time is short)

| Time | Screen | Voiceover |
|---|---|---|
| 0:00–0:15 | Live dashboard, idle | One-sentence problem statement: 10 activations in 2 years, fragmented information, no fast triage |
| 0:15–0:25 | Slide — architecture diagram | "Here's how it works, and how we built it" — AI-first workflow — **(1)** |
| 0:25–0:55 | Live: submit a real report | Name 2-3 of the 5 official sources it's checked against; emphasise this is live, not canned |
| 0:55–1:25 | Live: question appears, answer it, action appears, triaged result appears | The differentiator — every submission gets an immediate two-way exchange *and* an actionable response, both tracks in one moment |
| 1:25–1:35 | Slide — data-flow diagram | Quick technical grounding — reinforces "working demo" |
| 1:35–1:50 | Live: full feed, map, detail panel with official context + rationale | Honesty about limits, then data sovereignty **(2)** and sustainability **(3)** |
| 1:50–2:00 | Live dashboard or title card | Release-cadence roadmap **(4)**, socially sustainable if time allows **(5)**, close on the ask |

**Production notes:**
- Diagram inserts: 5-10 seconds each, no more — simplify the diagram rather than extending screen time
- Everything else stays on the live dashboard; if something needs a slide to make sense, surface it in the live UI instead
- Prefill the answer text ahead of recording — don't type it live, the 0:55–1:25 beat already has four live moments in 30 seconds
- 1:35–1:50 carries three ideas in 15 seconds — needs a tightly rehearsed line; cut the Svelte/point-5 detail first if it runs long

**Slides needed:** architecture, data flow, roadmap + scalability, team members.

---

## Live demo (4 minutes, 16:30)

A separate, in-person 4-minute demo, distinct from the submitted video. Both systems are real and deployed (backend on Google Cloud Run, frontend on GitHub Pages) — this is a concrete script against live URLs, not a placeholder.

**Live URLs (confirm both are warm first):**
- Public report form — deployed root, `?clarify=1` for the two-step Clarifier flow
- Staff dashboard — deployed `/dashboard/` route

**Ground rule:** judges can interrupt with questions any time — the table below is the default path if nobody does, not a script to recite regardless.

| Time | What | Who | Talking point |
|---|---|---|---|
| 0:00–0:20 | Open on the dashboard, idle | Presenter | One-sentence problem statement |
| 0:20–1:00 | Submit a real report on the public form | Presenter or volunteer | Name 2 of the 5 official sources; this is live, not canned |
| 1:00–1:40 | Question appears; answer it (prefilled, not typed live); action appears | Same person | Every submission gets this two-way exchange — both tracks in one moment |
| 1:40–2:20 | Switch to dashboard: new report on map/feed, open detail panel | Presenter | Official context and plain-language rationale — not a black box |
| 2:20–2:40 | Use the severity filter (with live counts) to isolate High | Presenter | A real staff control — how a team lead narrows the feed fast |
| 2:40–3:10 | One sentence on sovereignty/security and efficiency | Presenter | Our own models on our infrastructure — cheaper and more sustainable, down to the frontend framework choice |
| 3:10–3:40 | Roadmap: confidence + priority as the concrete "what's next" | Presenter | *"The classifier already outputs severity and rationale — next is confidence, so a well-evidenced high-severity call and a vague, uncorroborated one don't get treated the same."* |
| 3:40–4:00 | Close | Presenter | The ask; thank you |

**If asked:**
- *"What breaks this?"* — venue wifi; the live path depends on real external calls. If it degrades, switch to the recorded video rather than fighting a live failure.
- *"Why not a bigger model / API?"* — no per-token cost, no rate limits, no vendor dependency during an emergency, runs fully within WCC-controlled infrastructure.
- *"How does this scale past Wellington?"* — national data sources are near drop-in, the two LLMs are retrain-not-rebuild.

**Pre-demo checklist:**
- [ ] Warm the backend a few minutes beforehand — Cloud Run's cold start shouldn't hit the first request judges watch
- [ ] Both frontend URLs pre-loaded in open tabs
- [ ] The Phase 2 answer text ready to paste, not typed live
- [ ] The video cued up and ready as the wifi-failure fallback — decide now who plays it if needed
- [ ] Dashboard already has at least one triaged report visible before 0:00
