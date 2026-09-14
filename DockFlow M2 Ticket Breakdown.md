# DockFlow — M2 "Carrier Side" Sprint-Ready Ticket Breakdown v0.1

> **Companion to:** PRD §12 (M2), Modules D, E, F (§8), Technical Architecture v0.1, Data Model & API Spec v0.1, M0/M1 breakdowns
> **Date:** 2026-09-14 · **Milestone window:** weeks 8–10 (3 weeks) · **Team assumption:** 3 engineers
> **M2 exit criteria (PRD):** *A load moves tender → dispatched with real SMS.*
> **Hard dependencies:** M0 + M1 complete; **10DLC/toll-free registration approved** (submitted in S0-3 — if this hasn't cleared by M2 week 1, escalate to fallback provider immediately; it is the milestone's critical path).

---

## 0. Reading guide & sizing

Same conventions as M0/M1. M2 totals **~45–55 engineer-days** — 3 weeks × 3 engineers with buffer. This is the milestone where the platform becomes two-sided: every ticket involving cross-org data goes through the load authorization gate, and the driver experience must work for a person who will never log in, never install an app, and may be on a 3G connection in a dead zone (PRD §11).

---

## 1. Epic M2-A — State Machine Completion (Module F)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-A1 | Activate full transition set — `scheduled → tendered → accepted → dispatched → en_route → arrived → at_dock → loading_unloading → completed`, plus `cancelled`/`reschedule_requested` from any pre-completed state (Spec §2.3 table) | M1-A2 | 2d | All transitions validated incl. actor + source rules (driver can only fire `en_route`/`arrived`/`at_dock`/`loading_unloading` on own trip); property-based suite extended to full graph |
| M2-A2 | Per-role field projection — warehouse never sees carrier-internal fields, carrier never sees warehouse directory internals (F-1); serializer allowlists per audience | M2-A1 | 1.5d | Fixture test: carrier response payload contains zero warehouse-internal keys (and vice versa); serializer coverage enforced in CI |
| M2-A3 | Shared load timeline UI — both portals render the same immutable timeline (who/what/when), role-filtered fields (F-1) | M2-A2 | 1.5d | Timeline renders identically on both sides minus role-filtered fields; events attributed |

## 2. Epic M2-B — Tenders (Module D, first half)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-B1 | Tender creation — warehouse `POST /loads/{id}/tender` to selected carriers with explicit expiry; requires `active` relationship; multiple carriers per load allowed (sequenced or parallel per tenant setting) | M1-B5, M2-A1 | 2d | Tender without active relationship → 422; expiry persisted; **no code path creates a tender except this endpoint (PRD G-2 proof)** — grep-level test in CI |
| M2-B2 | Tender inbox — carrier `GET /tenders?status=` + detail view with pickup/delivery, window, load details, facility, rate if provided (D-1) | M2-B1 | 2d | Inbox ordered by expiry urgency; expired tenders auto-marked; warehouse notified on expiry |
| M2-B3 | Accept / decline / counter — state transitions + notifications both sides; counter proposes alternative window, warehouse accepts/declines counter (D-1) | M2-B2 | 2d | Full negotiation round-trip works from both UIs; every step in timeline + notifications |
| M2-B4 | Carrier portal: relationship acceptance UI + enable `carrier_requests` flag — dispatcher appointment requests (A-2) now flow end-to-end | M1-B5, M2-B3 | 1.5d | Dispatcher requests slot → warehouse manager approves/reschedules/declines → carrier notified; flag enabled per tenant |

## 3. Epic M2-C — Fleet & Driver Management (D-4)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-C1 | `vehicles` + `drivers` migrations and CRUD API (Spec §2.2); phone validation (E.164) | M0 done | 1d | Driver without valid phone cannot be saved (D-4); duplicate unit numbers rejected |
| M2-C2 | Fleet & drivers UI — carrier admin manages trucks, trailers, drivers incl. equipment qualifications | M2-C1 | 2d | Admin completes fleet setup without API client; deactivation preserves history |

## 4. Epic M2-D — Dispatch & Magic-Link Tokens (D-2, E-1 plumbing)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-D1 | `trips` migration + dispatch endpoint — `POST /loads/{id}/dispatch` assigns driver+truck+trailer; equipment qualification check; creates trip, transitions load → `dispatched` | M2-C1, M2-A1 | 1.5d | Unqualified equipment → 422 with reason; dispatch idempotent (retry safe) |
| M2-D2 | **Magic-link token service** — single-trip tokens: mint on dispatch, hashed at rest, expiry = completion +24 h, audience `drv`, scope = one trip (Spec §5.3) | M2-D1 | 2d | Token opens own trip only; expired/cross-trip/forged → 401; token never logged (redaction test) |
| M2-D3 | Dispatch SMS pipeline — `trip.dispatched` event → notifications worker → SMS with load ref, facilities+addresses, window, truck/trailer, secure link (E-1) | M2-D2, M2-E1 | 1.5d | SMS dispatched ≤30 s from assignment (D-2); content matches E-1 checklist; failure retries + flags dispatcher after final failure (I-2) |

## 5. Epic M2-E — SMS Production Integration (Module I, SMS half)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-E1 | Primary SMS provider live — 10DLC-verified sender wired into notifications worker behind provider interface (M0-G2); delivery webhooks → `notifications.status` | S0-3 cleared | 2d | Real SMS delivered to real phone from staging; delivery status round-trips; sender identity = tenant branding (PRD §4) |
| M2-E2 | **Fallback provider + delivery monitor** — rolling delivery-rate calculation; <95% trips failover (PRD §11); failover event alerts ops + tenant admin | M2-E1 | 2d | Chaos test: primary disabled → messages flow via fallback without code change; dashboard shows per-provider delivery rate |
| M2-E3 | Per-tenant SMS budget alerting (PRD §13 risk) — monthly counter per org, threshold alert | M2-E1 | 1d | Threshold crossing notifies admin + ops; counter visible in settings |

## 6. Epic M2-F — Driver Mobile Experience (Module E)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-F1 | **Mobile trip view** — `/t/{token}`: load ref, pickup/delivery facilities + addresses, window, assigned dock, truck/trailer, facility instructions (gate codes, check-in procedure) (E-1) | M2-D2 | 2.5d | Renders on iOS Safari + Android Chrome; functional on 3G (throttled test); large-type, high-contrast, no login prompt ever |
| M2-F2 | Big-thumb status buttons — **En route → Arrived → At dock → Loaded/Unloaded → Departed**; each tap → state machine → both portals update realtime (E-2) | M2-F1, M2-A1 | 2d | Buttons sized/spaced for gloved thumbs; offline tap queues and syncs; timestamps feed dwell analytics |
| M2-F3 | **Location consent + sharing** — explicit per-trip consent screen; browser geolocation pings while trip active; persistent "sharing location" indicator; stops at Departed; stop-anytime control (E-3, PRD §11) | M2-F2 | 2.5d | No ping stored without consent row; consent revocation halts collection immediately; pings land in `location_pings` (90-day retention job verified) |
| M2-F4 | Warehouse trip map — simple map showing consented truck position + latest ETA placeholder (real ETA in M3/M4) | M2-F3 | 1.5d | Warehouse sees position only while consent active and trip live; map hidden otherwise |
| M2-F5 | Change-notification SMS — reschedule, dock change, cancellation on active trip → driver SMS (E-4) | M2-E1, M2-B3 | 1d | Every material change produces SMS; content includes what changed, not just "update" |

## 7. Epic M2-G — Active Trips Board & Notifications Catalog

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-G1 | Dispatcher trips board — `GET /trips?active=` + UI: live status per trip, risk flag placeholder (ETA-based flagging arrives M3; M2 flags = scheduled window already passed) (D-3) | M2-D1 | 2d | Board updates via realtime channel `trips:{org}`; passed-window trips visually flagged |
| M2-G2 | Full notification event catalog (I-2) — appointment requested/approved/rescheduled/declined, tender received/accepted/declined/countered, dispatched (SMS), dock reassignment; prefs honored; all deep-linked | M2-E1, M1-F2 | 2d | Catalog complete per I-2; every event logged with delivery status |

## 8. Epic M2-H — Milestone Hardening & Exit

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M2-H1 | End-to-end demo data — second carrier org, drivers with real test numbers, full fleet; `make seed-demo` extended | all above | 1d | Two-sided demo reproducible from scratch |
| M2-H2 | **Milestone script: "tender → dispatched with real SMS"** — warehouse tenders → dispatcher accepts → assigns driver → real phone receives SMS → link opens trip → driver taps through statuses → both portals live-update | M2-H1 | 1d | Founder runs it on staging with a real phone; ≤30 s SMS verified with a stopwatch |
| M2-H3 | Security pass — magic-link token abuse testing (replay, cross-trip, brute force), SMS content injection check, rate limiting on driver endpoints | M2-D2, M2-F2 | 1.5d | Rate limits enforced; token abuse attempts rejected + logged; dependency scan clean |
| M2-H4 | Privacy verification — ping retention purge job, consent enforcement, driver data minimization review against PRD §11 | M2-F3 | 1d | Automated retention test passes; consent matrix documented |

**M2 exit checklist (maps to PRD exit criteria):**

- [ ] A load travels the full lifecycle tender → dispatched → completed across two real orgs
- [ ] A real phone receives the dispatch SMS within 30 s; the link works with no login (H-2)
- [ ] Driver status taps update both portals in realtime (F-2)
- [ ] Location sharing is consent-gated, indicated, revocable, and stops at Departed (F-3)
- [ ] SMS failover demonstrated by disabling the primary provider (E-2)
- [ ] Zero cross-org data leaks — per-audience serializer tests green (A-2)

---

## 9. Explicitly NOT in M2

Geofence arrival detection and threshold alerts (M3 — location data collected in M2 feeds it), ETA prediction (M4 — F-4/G-1 carry placeholders with honest "scheduled time" labels), AI matching (M4), vision module (M4/V0), in-app messaging (P1), POD/BOL upload (P1).

**Biggest M2 risks:** (1) **10DLC not cleared** — the S0-3 escalation path exists precisely for this; a milestone that ends without real SMS fails its exit criteria, so treat registration status as a weekly check from Sprint 0; (2) **browser geolocation quirks** (iOS Safari backgrounding, permission persistence) — F-3 needs real-device testing across the PRD browser matrix, not just emulation; (3) **driver UX on bad connections** — throttled 3G testing is in the ACs because drivers are the users least able to file a bug report and most able to abandon the product.

---

*End of M2 breakdown v0.1. Next: M3 (Arrival warnings — geofences, threshold alerts, escalation, rules-based ETA) ticket breakdown.*
