# DockFlow — M3 "Arrival Warnings" Sprint-Ready Ticket Breakdown v0.1

> **Companion to:** PRD §12 (M3), Module C (§8), Module H rules-based fallback (H-2), Technical Architecture v0.1, Data Model & API Spec v0.1, M0–M2 breakdowns
> **Date:** 2026-09-14 · **Milestone window:** weeks 11–12 (2 weeks) · **Team assumption:** 3 engineers
> **M3 exit criteria (PRD):** *Crew alerted 30 min before real arrivals at pilot site.*
> **Hard dependencies:** M2 complete — location pings flowing (consented), state machine live, SMS + in-app + email channels production-proven.

---

## 0. Reading guide & sizing

Same conventions as M0–M2. M3 totals **~28–34 engineer-days** — 2 weeks × 3 engineers. This milestone is short but unforgiving: its output is judged by dock crews in minutes, and a wrong alert cadence burns trust that took M1–M2 to build. Everything here is **rules-based ETA** (H-2) — the ML model stays in shadow until M4 — so correctness of *degradation labels* ("estimated — no live location") is as important as the estimates themselves.

---

## 1. Epic M3-A — Rules-Based ETA Engine (H-1/H-2, non-ML)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M3-A1 | ETA service skeleton (Python, Architecture §3) — consumes `location.ping_received` + `load.state_changed`; writes `eta_predictions`; refresh every 5 min per active trip **and** on each ping (H-1) | M2 done | 2d | Every active trip has a fresh (<6 min old) prediction; predictions carry `basis`, `confidence_min`, `model_version`; service restart loses nothing (state from events) |
| M3-A2 | **Live-location basis** — distance- and time-adjusted estimate from latest ping: remaining distance ÷ speed profile + stop allowance; confidence band from ping age/accuracy | M3-A1 | 2d | Stale ping (>15 min) widens band automatically; band never narrower than ±5 min |
| M3-A3 | **Rules basis (no live location)** — estimate from origin/destination distance, departure time (or scheduled window), historical trip durations on the lane when available (H-2) | M3-A1 | 2d | Zero-ping trips still get ETAs; `basis='rules'`; cold lanes (no history) fall back to distance-only with widest band |
| M3-A4 | Basis labeling end-to-end — every UI surface shows "live location" vs "estimated — no live location" (C-4); API exposes basis on trip/board payloads | M3-A2, M3-A3 | 1d | Label present on dock board, trips board, trip map, alerts; fixture test asserts no unlabeled ETA ships |

## 2. Epic M3-B — Geofence Arrival Detection (C-2)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M3-B1 | Geofence evaluator — ping inside facility radius (per-tenant, default 1 mi) → `geofence.entered` event → state machine `arrived` with `source: geofence`; dedupe (enter once per trip) | M3-A1 | 2d | Repeated pings inside fence → single transition; wrong-facility entry (carrier has multiple customers) → no transition, logged |
| M3-B2 | Arrival alert on geofence entry — dock crew + manager alerted with carrier, load, assigned dock (C-2); dock board reflects ≤10 s | M3-B1 | 1d | End-to-end: simulated ping → board update measured <10 s in staging |
| M3-B3 | Geofence admin — per-facility radius editor on map (A-4 surface); preview of fence on satellite view | M0 settings | 1.5d | Radius change takes effect on next ping; audit-logged |

## 3. Epic M3-C — Threshold Alerts (C-1)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M3-C1 | Threshold evaluator — consumes `eta.updated`; fires when predicted arrival crosses configured thresholds (default 60/30/10 min); per-tenant thresholds from settings; each alert names carrier, load, dock, predicted minute | M3-A1 | 2d | Crossing fires once per threshold per trip (no spam); ETA revision re-arms a threshold only if it uncrossed (e.g. 30→45→25 min) |
| M3-C2 | Alert routing — to roles subscribed for that dock/shift per notif prefs (C-1, I-1); in-app + email; optional manager SMS | M3-C1 | 1.5d | Preference matrix honored incl. new `arrival_warning` event type; quiet hours respected for email, never for in-app |
| M3-C3 | Dock board arrival strip — imminent arrivals (next 60 min) with live countdown and basis label | M3-C1, M1 dock board | 1d | Board updates countdown without refresh; stale ETA (>6 min) visually dimmed |

## 4. Epic M3-D — Late Escalation & Picker Queue Activation (C-3, H-3)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M3-D1 | Late-risk detector — predicted ETA slips past window end + grace period (per-tenant) → manager escalation + calendar conflict marker | M3-A1 | 1.5d | Escalation fires once per trip; calendar shows conflict badge on affected appointment |
| M3-D2 | One-tap reschedule from escalation — manager picks proposed alternative slot; carrier auto-notified (C-3) | M3-D1 | 1.5d | Reschedule reuses M1 drag/drop conflict validation; notification diff states old→new window |
| M3-D3 | **Activate `eta_sort` on staging queue** (M1-E2 seam) — picker queue re-sequences by predicted arrival; slipping ETA reorders queue automatically (H-3) | M3-A1 | 1d | Queue reorder animates + logs; basis label switches to "predicted arrival"; flag-gated per tenant |
| M3-D4 | Trips board risk flags — replace M2 placeholder ("window passed") with ETA-based at-risk flag (D-3 final form) | M3-A1 | 0.5d | Flag appears when predicted ETA > window end; clears on recovery |

## 5. Epic M3-E — Manual Fallback & Trust Calibration (C-4)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M3-E1 | Manual status fallback — dispatcher (or driver) sets En route / Arrived manually when location never shared; alerts fire on scheduled-time-minus-thresholds, clearly marked "estimated — no live location" | M3-C1 | 1d | Zero-consent trips still produce the full alert cadence; every surface carries the estimated label (A-4 verified) |
| M3-E2 | Consent-rate visibility — per-fleet opt-in rate surfaced to carrier admin + ops dashboard (PRD §13 risk: measure, don't assume) | M2-F3 | 1d | Opt-in % per carrier org; low-consent fleets flagged for dispatcher nudge |

## 6. Epic M3-F — Milestone Hardening & Exit

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M3-F1 | Arrival simulator — scripted fake trips with realistic ping streams (highway, urban, stalled, no-signal gap) driving the full alert pipeline in staging | all above | 1.5d | Simulated trip produces correct 60/30/10 cadence ±2 min against script ground truth |
| M3-F2 | **Milestone script: "30-minute warning"** — simulated inbound trip → dock crew receives 30-min alert naming carrier/load/dock → geofence arrival → board flips — the PRD exit criteria, rehearsed before the pilot site sees it | M3-F1 | 0.5d | Founder runs unaided; alert content matches C-1 spec verbatim |
| M3-F3 | Alert-storm guard — N simultaneous slipping trips escalate as a digest, not N pings; per-user rate cap on arrival SMS | M3-D1 | 1d | 5 simultaneous late trips → 1 digest; cap configurable per tenant |
| M3-F4 | Accuracy instrumentation — every prediction vs. actual arrival logged (the dataset M4's shadow mode will be measured against, H-4) | M3-A1 | 1d | Accuracy report queryable per facility/basis; exportable CSV |

**M3 exit checklist (maps to PRD exit criteria):**

- [ ] 60/30/10 alerts fire correctly against simulated ground-truth trips (F-1)
- [ ] Geofence entry auto-marks arrival and flips the dock board in <10 s (B-2)
- [ ] A no-location trip still alerts end-to-end, honestly labeled (E-1)
- [ ] Late escalation → one-tap reschedule → carrier notified (D-2)
- [ ] Picker queue sequences by predicted arrival (D-3)
- [ ] Accuracy instrumentation recording — M4's shadow mode has data waiting (F-4)

---

## 7. Explicitly NOT in M3

ML ETA model (M4 — this milestone builds the rules engine it must beat and the instrumentation it will be judged by), carrier matching (M4), vision module (M4/V0), multi-stop trips (P1), HOS-aware prediction (roadmap, PRD glossary).

**Biggest M3 risks:** (1) **Alert fatigue** — the fastest way to lose dock crews; F-3 digest guard + once-per-threshold semantics exist for this, and thresholds must be tuned per pilot site before go-live (FDE task, not a default); (2) **rules-ETA quality variance by lane** — A3's honest confidence bands and basis labels are the mitigation; never let a wide estimate masquerade as a tight one; (3) **geofence false positives** on facilities near highways — B-1 requires dwell-confirmation logic (inside fence ≥ N seconds) before transitioning, tuned per site.

---

*End of M3 breakdown v0.1. Next: M4 (AI features — carrier matching with approval gate + explainability, ETA model in shadow→live, plus the vision ingestion foundation and camera simulator) ticket breakdown.*
