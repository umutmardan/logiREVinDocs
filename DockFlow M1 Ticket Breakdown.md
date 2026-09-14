# DockFlow — M1 "Scheduling" Sprint-Ready Ticket Breakdown v0.1

> **Companion to:** PRD §12 (M1), Modules A & B (§8), Technical Architecture v0.1, Data Model & API Spec v0.1, M0 Ticket Breakdown
> **Date:** 2026-09-14 · **Milestone window:** weeks 4–7 (4 weeks) · **Team assumption:** 3 engineers
> **M1 exit criteria (PRD):** *Warehouse can run a full day of appointments with no carrier side.*
> **Hard dependency:** M0 complete (tenancy, auth, RLS, outbox, portal shells, flags).

---

## 0. Reading guide & sizing

Same conventions as the M0 breakdown: ideal engineer-day estimates, AC = acceptance criteria, *(stretch)* = first to cut under pressure. M1 totals **~55–70 engineer-days** — a full 4 weeks for 3 engineers with modest buffer.

**M1 scoping note (important):** PRD A-2 (dispatcher-requested appointments) requires a working carrier-side flow, which lands in M2. M1 delivers the warehouse side *plus* the API surface for A-2 behind a feature flag, so M2 only wires the carrier UI. Similarly, B-3's picker queue sequences by **scheduled time** in M1; the ETA-based re-sequencing arrives with Module H and is a drop-in sort-key change (ticket M1-E2 designs for this).

---

## 1. Epic M1-A — Load Domain & State Machine Foundation

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-A1 | `loads` + `load_events` migrations (Spec §2.3) incl. partitioned event table, `load_ref` generator (`LD-YYYY-NNNNN` per org) | M0 done | 1.5d | Migrations green; ref codes unique per warehouse org; event partition rollover tested |
| M1-A2 | **State machine module** — transition table (Spec §2.3) as versioned config; transition validator; side-effect dispatcher; M1 subset active: `draft → pending_approval → scheduled`, `→ cancelled`, `→ reschedule_requested` | M1-A1 | 2.5d | Illegal transitions rejected 422 with reason; every legal transition writes `load_events` row via outbox in same tx; side effects (notify) fire post-commit |
| M1-A3 | Load authorization gate v1 (Architecture §5.2) — `canAccess(org, load)`; applied to all load endpoints; cross-org → 404 | M1-A1 | 1.5d | Warehouse A cannot read warehouse B's loads; carrier org without relationship → 404; matrix suite extended |
| M1-A4 | Dock conflict validator — overlapping windows on same dock rejected; slot duration defaults from tenant settings; equipment constraints enforced (reefer load → reefer-capable dock, PRD A-4) | M1-A1 | 2d | Double-book attempt → 422 with conflicting appointment detail; constraint violation names the violated constraint |

## 2. Epic M1-B — Appointments & Dock Calendar (Module A)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-B1 | Appointments API — `POST/GET/PATCH /appointments` + approve/decline/reschedule endpoints (Spec §5.1); create form fields per A-1 (carrier from directory, PO refs, pallets, weight, commodity, equipment, window, instructions) | M1-A2, M1-A3 | 3d | Create → appears immediately as `draft`/`pending_approval` (A-1); validation errors field-level; idempotency key prevents double-submit |
| M1-B2 | Calendar feed API — `GET /appointments?from&to&dock_id&state&direction&carrier_id` with cursor pagination; optimized query (index per Spec §2.3) | M1-B1 | 1d | Day + week ranges return <300 ms at 10k appointments seed data |
| M1-B3 | **Dock calendar UI** — day/week views, dock-lane layout, filters (dock, status, carrier, direction), color by state (A-3) | M1-B2 | 3–4d | Manager reads a full day at a glance; filter combinations work; empty state guides first appointment |
| M1-B4 | Drag-and-drop rescheduling — drag appointment to new time/dock → optimistic UI → conflict validation → both-sides notification stub (in-app) | M1-B3, M1-A4 | 2.5d | Conflict → drag rejected with inline reason (A-3); success writes `load_events` with `source: user` |
| M1-B5 | Carrier-requested appointment API (A-2, warehouse half) — `POST /appointment-requests` (carrier audience, flag-gated), manager approve/reschedule/decline; reschedule proposes alternative slot rather than flat reject | M1-A2 | 2d | Behind `carrier_requests` flag (off until M2); manager decision notifies requester via in-app event |
| M1-B6 | Docks & facility config UI — dock CRUD (label, equipment constraints, out-of-service toggle), operating hours, slot-duration defaults (A-4) | M0 settings screens | 1.5d | Constraints set here are enforced by A4 validator; out-of-service dock unschedulable |

## 3. Epic M1-C — Dock Assignment Engine (Module B, first half)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-C1 | **Dock suggestion engine** — score candidate docks: availability window, equipment fit, current status; return top suggestion + one-line explanation ("Dock 4 free 14:00–16:00, fits reefer", B-1) | M1-A4 | 2d | Suggestion always explains itself; no suggestion when nothing fits (honest empty, never fabricated) |
| M1-C2 | Assign/override API — `POST /loads/{id}/assign-dock`; manager/worker override with reason captured to event payload | M1-C1 | 1d | Override allowed per RBAC (worker+, PRD §9.1); override reason in `load_events` |
| M1-C3 | Auto-suggest on confirm — state transition to `scheduled` triggers suggestion; if manager confirms in-line, assignment lands without a second screen | M1-C1, M1-B4 | 1d | One-click confirm path works from calendar detail |

## 4. Epic M1-D — Realtime Gateway & Live Dock Board (Module B, second half)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-D1 | **Realtime gateway service** (Architecture §3) — WS/SSE server, Redis pub/sub subscription to domain events, channel authz (`dock-board:{facility_id}` requires warehouse membership) | M0-B3 | 2.5d | Subscribe auth tested cross-tenant; SSE fallback works with WS blocked; reconnect resumes cleanly |
| M1-D2 | Dock board API + projection — `GET /dock-board` snapshot (dock status, current truck/load, next arrival per dock) maintained as read projection from events | M1-D1 | 1.5d | Snapshot <300 ms; projection rebuilds from event log (rebuild script included) |
| M1-D3 | **Live dock board UI** — wall-mountable large-type read-only view (B-2): every dock's status, truck, load contents, next arrival; auto-push via gateway | M1-D2 | 3d | Updates appear <10 s from event (target ~1 s); works in a bare TV browser without login prompt loops (dedicated read-only display token); readable at 5 m |
| M1-D4 | Degradation mode — gateway down → board polls snapshot every 15 s with visible "stale" badge; never blank (Architecture §7.1) | M1-D3 | 1d | Kill Redis in staging → board degrades, labels staleness, recovers automatically |
| — | *(stretch)* Board display themes — dark mode for dim warehouses, shift-filter presets | M1-D3 | 1d | Toggle persists per device |

## 5. Epic M1-E — Outbound Staging Queue (B-3, warehouse-internal)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-E1 | Staging queue API — `GET /staging-queue`: today's outbound appointments with load details, staging notes field, dock assignment | M1-B1 | 1d | Picker role can read (RBAC); notes editable by picker+ |
| M1-E2 | Queue UI sequenced by **predicted arrival** — M1: sort key = scheduled window start, implemented behind an `eta_sort` seam so Module H swaps in predicted ETA without UI changes (B-3) | M1-E1 | 1.5d | Sequencing function isolated + unit tested; UI shows basis label ("scheduled time") ready to become "predicted arrival" |
| M1-E3 | Staging notes — per-load free-text notes visible on queue + dock board detail; edits timestamped and attributed | M1-E1 | 1d | Notes survive reschedule; edit history in event payload |

## 6. Epic M1-F — Notifications for M1 Events (in-app + email)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-F1 | In-app notification center — bell + list in warehouse portal; unread state; deep links to load/dock view (I-3) | M0-G1 | 2d | Every M1 event below lands in-app within seconds via gateway |
| M1-F2 | M1 event catalog wired — appointment created / approved / declined / rescheduled, dock assigned/changed; per-user prefs honored (I-1); email mirrors in-app for managers | M1-F1, M1-B4 | 2d | Preference matrix respected; every notification carries deep link; delivery logged in `notifications` |

## 7. Epic M1-G — Milestone Hardening & Exit

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M1-G1 | Seed + demo environment — realistic seed: 1 warehouse, 12 docks, 3 carriers linked, 2 weeks of appointments, varied states/equipment | all above | 1.5d | `make seed-demo` reproduces; used for every sprint demo |
| M1-G2 | **"Run a full day" milestone script** — scripted walkthrough of exit criteria: configure docks → schedule a day → assign docks → run the board → reschedule mid-day → close out | M1-G1 | 1d | Founder runs the script unaided on staging; zero dead ends |
| M1-G3 | Load test — calendar + board under 5k concurrent viewers on one facility (PRD §11 ceiling), 100k appointments seeded | M1-D3 | 1.5d | Board latency <10 s p99 at ceiling; calendar queries hold <300 ms |
| M1-G4 | Authorization & state-machine suite extension — all M1 endpoints × roles; all M1-active transitions legal/illegal; vision-source transition fixtures rejected pre-flag | M1-A2, M1-A3 | 1.5d | CI gate green; intentional-fault fixture still fails the suite |

**M1 exit checklist (maps to PRD exit criteria):**

- [ ] A warehouse admin configures docks, hours, and constraints entirely in the UI (B-6)
- [ ] A manager schedules, approves, reschedules a full day's appointments — including drag-and-drop with conflict protection (B-3, B-4)
- [ ] Dock suggestions explain themselves; overrides are captured with reasons (C-1, C-2)
- [ ] The dock board runs live on a cheap tablet/TV, updates <10 s, degrades gracefully (D-3, D-4)
- [ ] Pickers work from the staging queue (E-2)
- [ ] All of the above with **zero carrier-side dependency** — the milestone's defining constraint

---

## 8. Explicitly NOT in M1

Carrier portal tender flows (M2), SMS/driver anything (M2), geofences & arrival alerts (M3), ETA prediction (M4 seam prepared in E-2), matching (M4), vision module (M4/V0). The `carrier_requests` flag (B-5) staying **off** until M2 is the scope fence.

**Biggest M1 risks:** (1) **drag-and-drop calendar UX complexity** — the highest-variance UI ticket; mitigate by time-boxing B-3/B-4 and falling back to a "move" dialog if the drag interaction slips; (2) **realtime gateway is new infrastructure** — D1 starts week 1 of the milestone, not week 3; (3) **conflict-validation edge cases** (overnight windows, out-of-service toggles mid-day) — A4 gets dedicated property-based tests, not just examples.

---

*End of M1 breakdown v0.1. Next: M2 (Carrier side — tenders, dispatch, SMS magic links, driver mobile view) ticket breakdown.*
