# DockFlow — M4 "AI Features + Vision Foundation" Sprint-Ready Ticket Breakdown v0.1

> **Companion to:** PRD §12 (M4), Modules G & H (§8), Vision Addendum K.4–K.5, Technical Architecture v0.1 (§12 mapping), Data Model & API Spec v0.1, M0–M3 breakdowns
> **Date:** 2026-09-14 · **Milestone window:** weeks 13–16 (4 weeks) · **Team assumption:** 3 engineers
> **M4 exit criteria (PRD):** *Recommendations live with ≥ 1 pilot; ETA shadow accuracy measured.*
> **Architecture scope note:** per Technical Architecture §12, M4 also delivers the **vision ingestion endpoint, reconciler, anomaly rules engine, and camera simulator** — the entire vision pipeline built against synthetic events, so V0 hardware lands on working software.

---

## 0. Reading guide & sizing

Same conventions as M0–M3. M4 totals **~52–62 engineer-days** — 4 weeks × 3 engineers. Two tracks run in parallel: **AI** (matching + ETA graduation) and **vision foundation** (pipeline without hardware). Both are trust products: matching only works if managers believe the explanations, and the vision pipeline must classify correctly before a single camera is purchased.

---

## 1. Epic M4-A — Matching Data Foundation (feeds G-1)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M4-A1 | Carrier performance aggregation — per (warehouse_org, carrier_org, lane): load count, on-time %, avg dwell, equipment types, recency, rate history if present; rebuilt from `load_events` (dwell = measured where vision/reported timestamps exist) | M3 done | 2d | Aggregates reproducible from event log; per-tenant isolation (G-5: no cross-tenant leakage, RLS + tests) |
| M4-A2 | **CSV historical import for cold start (G-4)** — `POST /imports/loads_historical`: column mapping UI, validation report, dry-run mode; imports synthesize historical aggregates without fake live loads | M4-A1 | 2.5d | FDE onboarding checklist executable: import → aggregates visible → recommendations possible; bad rows reported line-by-line, never silently dropped |
| M4-A3 | Preferred-carrier lists — manually curated per warehouse (G-4b); used as ranking input + cold-start floor | M4-A1 | 1d | Curated carriers surface first in directory; listed as factor when ranked |

## 2. Epic M4-B — Matching Service, Explainability & Approval Gate (G-1, G-2, G-3, G-5)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M4-B1 | Ranking service (Python AI service) — scoring: lane history, on-time % at this facility, dwell at these docks, equipment fit, recency, rate if known; weights versioned per model_version | M4-A1 | 2.5d | Score decomposes into factor contributions summing to total; deterministic given same inputs (tested) |
| M4-B2 | **Explainability renderer** — every recommendation shows top reasons in plain language: "On-time 94% across 18 loads on this lane; avg dwell 52 min at your docks" (G-3) | M4-B1 | 1.5d | Every score decomposes to human-readable factors; no bare numbers anywhere in UI |
| M4-B3 | Recommendations UI — ranked list on outbound load creation; manager selects one/more, adjusts, or ignores; only then `POST /loads/{id}/tender` (M2-B1) fires | M4-B2 | 2d | Zero auto-tendering paths (G-2 CI test still green); ignoring recommendations is a first-class flow, not a dead end |
| M4-B4 | Honest cold start — no history for lane → UI shows directory + preferred list with "not enough history" state; **never fabricated rankings (G-4)** | M4-B3 | 1d | Fixture test: empty-history warehouse sees zero fake scores |
| M4-B5 | Decision logging + learning loop — approved/declined recorded on `carrier_recommendations` (G-5); decision feed available for weight tuning; per-tenant only | M4-B3 | 1d | Every decision attributed + timestamped; audit exportable (J-4) |

## 3. Epic M4-C — ETA Model: Shadow → Live (H-1, H-4)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M4-C1 | ML ETA model v1 — features: live pings (speed, remaining distance), lane history, time-of-day, historical dwell at destination; trained/evaluated on M3 accuracy data (`shadow=true` rows alongside rules engine) | M3-F4 | 3d | Offline eval report vs rules baseline; model artifact versioned; falls back to rules on any feature gap (H-2) |
| M4-C2 | Shadow-mode operation — both engines write predictions (rules `shadow=false`, ML `shadow=true`); accuracy dashboard per facility: ±15 min @ 1h-out hit rate (PRD §3.2 target ≥70%) | M4-C1 | 2d | Dashboard answers "would ML have beaten rules this week?" per site; exportable |
| M4-C3 | Graduation flag — per-tenant `eta_model_live` flag flips alert/queue inputs from rules to ML **only after** accuracy target met at that site (H-4); rollback = flag off | M4-C2 | 1d | Flag flip live-switches basis without deploy; basis label updates to reflect model version; rollback tested |
| M4-C4 | Confidence calibration — predicted bands vs observed error; auto-widen bands when model is overconfident on a lane | M4-C2 | 1.5d | Band-accuracy report shows predicted ±X contains actual ≥ target rate |

## 4. Epic M4-D — Vision Foundation: Pipeline Without Hardware (K.4, K.5.2)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M4-D1 | Vision schema migrations — `edge_boxes`, `cameras`, `zones`, `vision_events` (partitioned), `anomalies`, `claims_bundles` (Spec §3) | M0 done | 1.5d | Migrations green; RLS via facility→org chain verified |
| M4-D2 | **Camera simulator** — service emitting realistic `VisionEvent` streams from a site layout definition (zones, choke points, scripted scenarios: normal day, unlogged move, forgotten pallet, door mismatch) | M4-D1 | 2.5d | Scenarios drive full pipeline end-to-end; simulator is the M4 demo harness and the permanent regression tool |
| M4-D3 | Edge ingestion API — `POST /edge/v1/vision-events`: mTLS identity, batch ingest, idempotent by edge UUID, backfill-safe (Spec §5.4) | M4-D1 | 2d | Duplicate delivery → single stored event; out-of-order backfill lands correctly; load-tested at 10× expected rate (Architecture §11.4) |
| M4-D4 | **Reconciler** — joins `vision_events` against active loads/tasks/appointments → `logged` / `unlogged` / `divergent` (Architecture §4.3); CSV-imported WMS tasks supported as reconciliation context | M4-D3 | 2.5d | Simulator scenarios classify with ≥ target precision on the "unlogged move" scenario; divergent load flagged with both viewpoints visible |
| M4-D5 | Anomaly rules engine — K-A1 unlogged movement, K-A2 forgotten pallet (per-zone `is_storage` + tenant thresholds), K-A3 departed-but-staged, K-A4 door-count mismatch; per-tenant config from `settings.vision` | M4-D4 | 2d | Each rule covered by simulator scenario + unit tests; thresholds hot-configurable per tenant |
| M4-D6 | Anomaly triage UI — list (`open` first), ack/assign/resolve with note; clip placeholder (clip pipeline completes with V0 hardware); notification routing per vision config | M4-D5 | 2d | Manager triages a simulated anomaly end-to-end; audit trail on resolution |
| M4-D7 | Edge config channel — `GET /edge/v1/config` + zone/rule editor in portal (zone polygons drawn over camera-view placeholder) | M4-D3 | 2d | Zone edit in portal → config payload reflects it; versioned config (box acks version applied) |

## 5. Epic M4-E — Milestone Hardening & Exit

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M4-E1 | AI decision audit — recommendations, decisions, shadow comparisons, flag changes all in audit log + exportable (J-4: "who approved this carrier and why" in one click) | M4-B5, M4-C3 | 1d | One-click answer reproducible on demo data |
| M4-E2 | **Milestone script A: "recommend → approve → tender"** — import pilot CSV history → create outbound load → explained recommendation → manager approves → tender sent | M4-B3 | 0.5d | Founder runs unaided on staging |
| M4-E3 | **Milestone script B: "camera-free vision demo"** — simulator runs the forgotten-pallet scenario → anomaly raised → triaged → resolved; shown to pilot prospect as the V0 preview | M4-D6 | 0.5d | Demo runs from `make seed-demo` + one simulator command |
| M4-E4 | Model governance note — model versions, training data scope (per-tenant only), and shadow reports recorded in repo docs; FDE runbook updated with graduation checklist | M4-C3 | 1d | Runbook lets an FDE graduate a site's ETA model without an engineer |

**M4 exit checklist (maps to PRD exit criteria):**

- [ ] Recommendations live with ≥ 1 pilot (or pilot-signed staging demo), every score explained (B-2, B-3)
- [ ] No auto-tendering path exists — CI grep test green (B-3)
- [ ] Cold start is honest — no fabricated rankings, CSV import works (A-2, B-4)
- [ ] ETA shadow accuracy measured per facility against the ≥70% target (C-2)
- [ ] Full vision pipeline proven against the simulator — ingest → reconcile → anomaly → triage (D-3…D-6)
- [ ] V0 hardware BOM finalized from Architecture §11.3 decision, ready to order for pilot install

---

## 6. Explicitly NOT in M4

Physical edge box software (V0 — the cloud side is now ready for it), clip pipeline from real cameras (V0), claims bundle sharing polish (V3 productized; basic generation exists via schema), plate capture (P1), zone-level inventory counts (V2).

**Biggest M4 risks:** (1) **Matching data sparsity** — pilot history may be thin even after CSV import; B-4's honest-empty design is the mitigation, and the FDE onboarding checklist (A-2) must be treated as blocking for the pilot promise; (2) **shadow accuracy may miss the 70% target** — the milestone exits on *measured*, not *passed*; the rules engine stays live until a site earns graduation (C-3), which is exactly what H-4 was designed for; (3) **reconciler classification quality** — the "unlogged move" false-positive rate determines whether warehouse managers trust or mute anomaly alerts; D-4's simulator precision target must be met before V0 hardware is ordered.

---

*End of M4 breakdown v0.1. Remaining roadmap: M5 (pilot hardening — CSV polish, KPI analytics, FDE runbook) and V0 (physical vision kit deployment) — both can be ticketed once pilot dates are real.*
