# DockFlow — MVP Product Requirements Document

> **Working title** — "DockFlow" is a placeholder. Rename freely.

| | |
|---|---|
| **Version** | 0.1 (Draft) |
| **Date** | 2026-08-18 |
| **Status** | Draft for review |
| **Product type** | Two-sided logistics coordination platform (Warehouse ↔ Carrier) |
| **Business model** | Hybrid: multi-tenant SaaS core + Forward Deployed Engineer (FDE) customization |
| **Document owner** | Founder (warehouse/logistics domain lead) |

---

## 1. Executive Summary

DockFlow is a logistics coordination platform that connects **warehouses** and **trucking companies** through two independent web portals with separate authentication:

- **Warehouse Portal** — used by warehouse admins, managers, order pickers, and dock workers to schedule appointments, assign loading docks, and prepare for arrivals.
- **Carrier Portal** — used by carrier admins and dispatchers to accept load tenders and assign drivers/trucks/trailers, and by **drivers** through a mobile-friendly web experience driven by SMS notifications.

The MVP delivers the operational spine: **appointment scheduling, dock assignment, and driver arrival warnings** — plus two AI capabilities at launch:

1. **Smart carrier matching** — when a warehouse creates a load, the system recommends trucking companies the warehouse has worked with in the past (ranked by lane, on-time %, dwell history, equipment fit). **Nothing is tendered without warehouse manager approval.**
2. **Driver ETA prediction** — continuously updated arrival estimates that power the arrival-warning system, so dock crews are ready when a truck hits the geofence.

The product is built as a **multi-tenant SaaS core** designed from day one for **FDE customization**: feature flags, per-tenant configuration, and integration APIs let deployed engineers adapt the product per client without forking the codebase.

---

## 2. Problem Statement

**Warehouses** still run dock scheduling on phone calls, emails, and spreadsheets. The result:

- Trucks arrive unannounced or bunched together → dock congestion, idle crews, detention fees.
- No advance warning of arrivals → pickers and loaders can't stage freight in time.
- Choosing a carrier for an outbound load is tribal knowledge — whoever the manager remembers.

**Trucking companies** on the other side have no visibility into dock availability, burn driver hours waiting, and coordinate dispatch through the same calls and emails.

**There is no shared source of truth between the two sides.** Each side runs its own tools (or none), and the handshake between them — the appointment, the arrival, the dock — is invisible until it goes wrong.

**Why us / why now:** the founding team combines hands-on warehouse management experience (dock ops, labor, carrier relationships) with a network of engineers deployable as FDEs to early clients — productize what the first clients need, customize the last mile on-site.

---

## 3. Goals & Success Metrics

### 3.1 Business goals (12 months post-MVP)

| # | Goal | Target |
|---|------|--------|
| BG-1 | Pilot customers live on the platform | 3–5 warehouses, 10+ carriers |
| BG-2 | Weekly active coordination (appointments created via platform, not phone) | ≥ 70% of pilot-site volume |
| BG-3 | FDE revenue attached to SaaS contracts | ≥ 1 paid customization engagement |
| BG-4 | Referenceable case study with quantified dwell/detention reduction | 1+ |

### 3.2 Product KPIs (measured per pilot site)

| KPI | Baseline (typical, manual) | MVP Target |
|---|---|---|
| Average driver dwell time at dock | 90–120 min | −20% |
| Appointments with ≥ 30 min advance arrival warning | ~0% | ≥ 80% |
| Dock utilization conflicts (double-booked dock-hour) | frequent | < 2% of slots |
| Tender acceptance rate (matched carriers) | n/a | ≥ 60% |
| Manager approval rate of AI carrier recommendations | n/a | ≥ 50% (quality signal) |
| ETA prediction accuracy (± 15 min, 1 hour out) | n/a | ≥ 70% |
| SMS delivery success rate | n/a | ≥ 98% |

---

## 4. Business & Deployment Model: Hybrid SaaS + FDE

The company sells a **multi-tenant SaaS core** and attaches **Forward Deployed Engineers** to clients who need customization (integrations with a client's WMS/TMS, custom workflows, on-site rollout).

This shapes the product architecture — these are **product requirements**, not just engineering preferences:

| Requirement | Rationale |
|---|---|
| **Config over code** — per-tenant settings for geofence radius, alert thresholds, approval rules, dock layouts | FDEs tune a client without branching code |
| **Feature flags per tenant** | FDE-built features roll out dark, enable per client, graduate to core if reusable |
| **Public REST API + webhooks** (appointment created, status changed, arrival predicted) | FDEs integrate client WMS/TMS/ERP without core changes |
| **CSV import/export everywhere** (historical loads, carriers, docks, users) | Client onboarding and AI cold-start (see §8, Module G) |
| **Per-tenant branding** (logo, colors, email/SMS sender name) | Enterprise clients expect it; cheap to build early |

**SaaS core principle:** an FDE customization that three clients want becomes a core feature; a customization one client wants stays a flagged extension. The PRD's scope discipline protects this.

---

## 5. Personas & Roles

### 5.1 Warehouse Portal roles

| Role | Description | Top jobs |
|---|---|---|
| **Warehouse Admin** | Runs the account; manages users, docks, carrier directory, settings | Onboard org, configure docks/geofences, manage permissions |
| **Warehouse Manager** | Owns the schedule and carrier relationships | Approve appointments & carrier matches, resolve conflicts, monitor dwell |
| **Order Picker** | Prepares outbound loads / puts away inbound | See what's arriving/departing and when, so freight is staged in time |
| **Warehouse Worker** (dock/yard) | Works the dock | Live dock board: who's at which dock, load contents, next arrival |

### 5.2 Carrier Portal roles

| Role | Description | Top jobs |
|---|---|---|
| **Carrier Admin** | Runs the carrier account | Manage users, fleet (trucks/trailers), drivers, warehouse relationships |
| **Dispatcher** | Assigns freight to drivers | Accept/decline tenders, assign driver + truck + trailer, track active trips |
| **Driver** | Drives; **mobile web only, no native app** | Receive assignment via SMS, view trip + dock details, share ETA/location, check in on arrival |

### 5.3 Driver access model (key decision)

Drivers are **not** portal users with passwords. Authentication is **SMS magic-link**:

1. Dispatcher assigns a driver → driver receives SMS with assignment (truck, trailer, pickup/delivery, time window) and a secure link.
2. Tapping the link opens the mobile trip view with a short-lived, single-trip token.
3. No app install, no password, no account management. Token expires when the trip completes.

This removes the #1 adoption barrier for drivers and keeps the dual-auth model clean: drivers never touch the Carrier Portal login.

---

## 6. Scope

### 6.1 In scope — MVP (P0)

| Module | Summary |
|---|---|
| A. Appointment scheduling | Create/approve/reschedule inbound & outbound dock appointments |
| B. Dock assignment & live dock board | Manual + system-suggested dock assignment; real-time board |
| C. Driver arrival warning | ETA-based alerts (60/30/10 min) + geofence arrival detection |
| D. Carrier tenders & dispatch | Tender inbox (accept/decline/counter), assign driver/truck/trailer |
| E. Driver mobile experience | SMS assignment + magic link, trip view, status updates, location sharing |
| F. Shared load lifecycle | One appointment/load object both sides see, with full status timeline |
| G. AI carrier matching | Ranked recommendations from warehouse's carrier history, manager-approved |
| H. AI ETA prediction | Continuous ETA feeding Module C; rules-based fallback |
| I. Notifications | SMS (drivers), in-app + email (portal users), preferences |
| J. Admin & tenancy | Org onboarding, RBAC, feature flags, audit log, CSV import/export |

### 6.2 Fast-follow (P1) — explicitly *not* MVP, but designed for

- In-app messaging between warehouse and carrier on a load thread
- POD/BOL document upload & storage (manual upload, **no OCR yet**)
- Dwell-time & carrier scorecard analytics dashboard
- Detention-time tracking and reports
- Multi-stop trips

### 6.3 Out of scope (MVP)

- Native mobile apps (mobile web covers drivers)
- Full WMS functionality (inventory, pick-path optimization, put-away logic)
- Billing, invoicing, carrier payment, factoring
- ELD/telematics hardware integrations (location comes from the driver's phone)
- BOL/POD OCR and AI document extraction
- Natural-language analytics copilot
- Open marketplace bidding to unknown carriers (matching is limited to **carriers the warehouse has worked with before** — this is a trust feature, not a marketplace)
- SSO/SAML enterprise auth (roadmap)

---

## 7. Product Overview

```
┌──────────────────────────┐          ┌──────────────────────────┐
│     WAREHOUSE PORTAL     │          │      CARRIER PORTAL      │
│  warehouse.dockflow.app  │          │   carrier.dockflow.app   │
│                          │          │                          │
│  Admin · Manager ·       │          │  Admin · Dispatcher      │
│  Picker · Worker         │          │                          │
│                          │          │  Drivers → mobile web,   │
│  Auth stack A            │          │  SMS magic link (no pw)  │
│  (user pool A)           │          │  Auth stack B            │
└────────────┬─────────────┘          └────────────┬─────────────┘
             │                                     │
             └───────────────┬─────────────────────┘
                             ▼
              ┌────────────────────────────┐
              │   SHARED COORDINATION CORE  │
              │                             │
              │  Load / Appointment object  │  ← the only thing both
              │  Status state machine       │    orgs share; all cross-org
              │  Notifications service      │    access is server-side
              │  AI: matching + ETA engine  │    authorized per-load
              │  Multi-tenant data store    │
              └────────────────────────────┘
```

**Dual-authentication principle:** the two portals have fully separate auth configurations — separate user pools, session policies, and (later) separate SSO providers. A warehouse user can never authenticate into the Carrier Portal and vice versa. The two orgs meet only through the shared Load/Appointment object, where every cross-org read/write is authorized server-side against that specific load.

---

## 8. Functional Requirements

Priority legend: **P0** = MVP, **P1** = fast-follow. Each story lists key acceptance criteria (AC).

---

### Module A — Appointment Scheduling (Warehouse Portal) · P0

**A-1.** As a **warehouse manager**, I can create an appointment (inbound or outbound) with: carrier (from directory), PO/reference numbers, load details (pallets, weight, commodity, equipment type: dry van / reefer / flatbed), requested time window, and special instructions.
- AC: appointment appears on the dock calendar immediately as *Draft* or *Pending Approval*.

**A-2.** As a **dispatcher**, I can request an appointment slot at a warehouse I have a relationship with; the warehouse manager approves, reschedules, or declines.
- AC: manager sees the request with full load context; decision triggers notification to the carrier; reschedule proposes an alternative slot rather than rejecting flat.

**A-3.** As a **manager**, I see a dock calendar (day/week views) of all appointments, filterable by dock, status, carrier, direction (inbound/outbound), with drag-and-drop rescheduling.
- AC: drag-and-drop change validates dock conflicts before saving; both sides notified on change.

**A-4.** As a **warehouse admin**, I can configure my facility: docks (ID, type, equipment constraints), operating hours, appointment slot duration defaults, geofence radius, and alert thresholds.
- AC: dock constraints (e.g., reefer-only dock) are enforced by the scheduling and assignment logic.

---

### Module B — Dock Assignment & Live Dock Board (Warehouse Portal) · P0

**B-1.** When an appointment is confirmed, the system **suggests a dock** based on equipment constraints, dock availability, and load type; the manager or worker confirms or overrides.
- AC: suggestion explains itself in one line ("Dock 4 free 14:00–16:00, fits reefer").

**B-2.** As a **dock worker**, I see a **live dock board**: every dock's current status (free / occupied / reserved), the truck at it, the load contents, and the next scheduled arrival per dock.
- AC: board is wall-mountable (large-type read-only view), auto-refreshes, works on a cheap tablet or TV browser.

**B-3.** As an **order picker**, I see today's outbound queue with staging notes, so freight is staged before the truck arrives.
- AC: queue is sequenced by predicted arrival (fed by Module H), not just scheduled time.

---

### Module C — Driver Arrival Warning (Warehouse Portal) · P0

The flagship warehouse feature. Two trigger mechanisms, both configurable per tenant:

**C-1. ETA-threshold alerts.** As a **dock worker / manager**, I receive alerts when a driver is predicted to arrive within configured thresholds (default 60 / 30 / 10 minutes).
- AC: thresholds configurable per tenant; alerts go to the roles subscribed for that dock/shift; each alert names the carrier, load, assigned dock, and predicted arrival minute.

**C-2. Geofence arrival detection.** The system creates a geofence around each warehouse (default 1 mi radius, configurable). When the driver's device reports a position inside the geofence, the system marks **Arrived** and alerts the dock crew.
- AC: geofence entry auto-updates load status to *Arrived*; dock board reflects it within 10 seconds; no manual driver action required.

**C-3. Late-arrival escalation.** If predicted ETA slips past the appointment window by more than the configured grace period, the manager is alerted and the dock calendar shows the conflict.
- AC: manager can one-tap reschedule; carrier notified automatically.

**C-4. Manual fallback.** If a driver never shares location, dispatcher or driver can manually set status *En route / Arrived*, and alerts fire on scheduled time minus thresholds.
- AC: alerts clearly marked "estimated — no live location" so crews calibrate trust.

---

### Module D — Tenders & Dispatch (Carrier Portal) · P0

**D-1.** As a **dispatcher**, I receive load tenders from warehouses in a tender inbox and can **accept, decline, or counter** (propose a different time window).
- AC: tender shows pickup/delivery, time window, load details, facility, and (if provided) rate; every decision notifies the warehouse; tender has an explicit expiry.

**D-2.** On acceptance, as a **dispatcher**, I assign a **driver, truck, and trailer** to the load.
- AC: assignment triggers the SMS to the driver (Module E) within 30 seconds; truck/trailer IDs are included in the SMS and shown to the warehouse (gate/security knows what to expect).

**D-3.** As a **dispatcher**, I see all active trips on a board with live status and predicted ETA per trip.
- AC: trips at risk of missing their window are visually flagged.

**D-4.** As a **carrier admin**, I can manage my fleet (trucks, trailers) and drivers (name, phone — required for SMS, equipment qualifications).
- AC: a driver cannot be assigned without a valid phone number.

---

### Module E — Driver Mobile Experience · P0

**E-1.** As a **driver**, when dispatched I receive an **SMS**: load reference, pickup & delivery facilities with addresses, time window, assigned dock (once set), my truck & trailer assignment, and a secure link.
- AC: SMS sent within 30 s of assignment; link opens mobile trip view with no login; link is single-trip and expires at completion +24 h.

**E-2.** As a **driver**, the mobile trip view shows: route-relevant details, facility instructions (gate codes, check-in procedure), and big-thumb status buttons: **En route → Arrived → At dock → Loaded/Unloaded → Departed**.
- AC: each tap updates both portals in real time; statuses are timestamped (feeds dwell analytics later).

**E-3.** As a **driver**, I can opt in to **share my location for this trip only** (browser geolocation, periodic pings while the trip is active).
- AC: explicit consent screen per trip; location stops at *Departed*; a persistent "sharing location" indicator is visible; warehouse sees the truck on a simple map. (Consent-first: this is a legal/trust requirement, see §11 NFRs.)

**E-4.** If plans change, the driver receives an **SMS update** (reschedule, dock change) — drivers never need to check the app unprompted.
- AC: every material change to an active trip produces an SMS.

---

### Module F — Shared Load Lifecycle (Cross-portal core) · P0

One **Load/Appointment object** is the shared source of truth. State machine:

```
Draft → Pending Approval → Scheduled → Tendered → Accepted →
Dispatched → En Route → Arrived → At Dock → Loading/Unloading →
Completed        (any state → Cancelled / Reschedule requested)
```

**F-1.** Both portals render the same load timeline (who did what, when), each side seeing the fields relevant to its role.
- AC: every state transition is timestamped, attributed, and immutable (audit log); the warehouse never sees carrier-internal data (e.g., driver pay) and the carrier never sees warehouse-internal data (e.g., other carriers' names in the directory).

**F-2.** All cross-org access is authorized **per load**: a carrier org can read/write only loads it is party to.
- AC: enforced server-side; covered by authorization tests.

---

### Module G — AI: Smart Carrier Matching · P0

**G-1.** As a **warehouse manager**, when I create an outbound load, I get a **ranked list of recommended carriers** drawn from carriers *my warehouse has worked with before*.
- AC: ranking factors at minimum: same/similar lane history, equipment fit, on-time % at my facility, historical dwell time at my docks, recency of last job, and (if known) historical rate.

**G-2. Manager approval gate — hard requirement.** No tender is ever sent automatically. The manager sees the ranked list, can select one or more carriers, adjust, or ignore the recommendations entirely, and only then do tenders go out.
- AC: zero auto-tendering code paths exist; this is a trust feature and a liability shield — AI recommends, humans decide.

**G-3. Explainability.** Every recommendation shows its top reasons: "On-time 94% across 18 loads on this lane; avg dwell 52 min at your docks."
- AC: every score decomposes into human-readable factors; no black-box numbers.

**G-4. Cold start.** A new warehouse has no history, so matching is bootstrapped by: (a) CSV import of historical loads/carriers during onboarding, (b) a manually curated preferred-carrier list, (c) falling back to "no recommendation" — never fabricated suggestions.
- AC: FDE onboarding checklist includes historical-data import; until ≥ 1 past carrier exists for a lane, the UI shows the directory, not fake rankings.

**G-5. Learning from decisions.** Approved/declined recommendations are logged to improve ranking over time.
- AC: manager decisions feed the model; per-tenant data never leaks into another tenant's rankings.

---

### Module H — AI: Driver ETA Prediction · P0

**H-1.** The system produces a continuously updated ETA for every active trip, refreshed at least every 5 minutes (or on each location ping).
- AC: ETA carries a confidence band; UI shows "arriving ~14:20 (±10 min)".

**H-2. Inputs (graceful degradation):** live location pings (when shared) → distance- and traffic-adjusted estimate; without live location → rules-based estimate from origin/destination distance, departure time, and historical trip durations on that lane. Historical dwell at the destination dock informs total turnaround predictions.
- AC: every ETA displays its basis ("live location" vs "estimated"); no crash path when data is missing.

**H-3.** ETAs feed Module C's arrival-warning thresholds and Module B's picker queue sequencing.
- AC: a slipping ETA re-sequences the picker queue and triggers C-3 escalation automatically.

**H-4. Shadow-mode launch.** For the first 2–4 weeks at a pilot site, the model runs in shadow: predictions logged, alerts still rules-based, accuracy measured against actuals; when accuracy ≥ target (§3.2), alerts switch to model-driven.
- AC: per-tenant switch is a feature flag (FDE controls rollout per site).

---

### Module I — Notifications · P0

**I-1.** Channels by audience: **SMS for drivers** (assignment, changes, dock instructions); **in-app + email** for portal users; optional SMS for warehouse managers for escalations.
- AC: notification preferences per user per event type; drivers always get SMS (no opt-down — it's their only channel).

**I-2.** Event catalog (MVP): appointment requested/approved/rescheduled/declined, tender received/accepted/declined/countered, driver dispatched (SMS), ETA threshold crossed, geofence arrival, late-risk escalation, dock reassignment.
- AC: every event logged with delivery status; failed SMS retries with backoff and flags the dispatcher after final failure.

**I-3.** All notifications carry a deep link to the relevant load/dock view.

---

### Module J — Admin, Tenancy & Audit · P0

**J-1. Org onboarding:** warehouse orgs and carrier orgs are created separately; a **relationship link** between a warehouse org and a carrier org is established by invitation/acceptance (like a connection request) before any tendering can occur.
- AC: carriers can't see a warehouse's loads without an accepted relationship.

**J-2. User management & RBAC:** admins invite users, assign roles, deactivate. Permission matrix in §9 is enforced server-side.

**J-3. Feature flags per tenant** and per-tenant configuration store (FDE foundation, §4).

**J-4. Audit log:** every state change, permission change, notification, and AI recommendation/decision is recorded and exportable.
- AC: a manager can answer "who approved this carrier and why" in one click.

**J-5. CSV import/export:** historical loads, carrier directory, fleet, users (drives G-4 cold start and FDE onboarding).

---

## 9. RBAC Permission Matrix

### 9.1 Warehouse Portal

| Capability | Admin | Manager | Order Picker | Worker |
|---|---|---|---|---|
| Manage org, users, docks, settings | ✅ | — | — | — |
| Manage carrier directory & relationships | ✅ | ✅ | — | — |
| Create / edit appointments | ✅ | ✅ | — | — |
| Approve / decline / reschedule appointments | ✅ | ✅ | — | — |
| **Approve AI carrier recommendations & send tenders** | ✅ | ✅ | — | — |
| Assign / override docks | ✅ | ✅ | — | ✅ |
| View live dock board | ✅ | ✅ | ✅ | ✅ |
| View outbound staging queue | ✅ | ✅ | ✅ | ✅ |
| Receive arrival alerts | opt-in | ✅ | opt-in | ✅ |
| View audit log & analytics | ✅ | ✅ | — | — |

### 9.2 Carrier Portal

| Capability | Admin | Dispatcher | Driver (mobile web) |
|---|---|---|---|
| Manage org, users, fleet, drivers | ✅ | — | — |
| Manage warehouse relationships | ✅ | ✅ | — |
| View / respond to tenders | ✅ | ✅ | — |
| Assign driver / truck / trailer | ✅ | ✅ | — |
| View active trips & ETAs | ✅ | ✅ | own trip only |
| Update trip status | ✅ | ✅ | ✅ (own trip) |
| Share location | — | — | ✅ (own trip, consented) |

### 9.3 Dual-authentication rules

1. **Separate auth stacks per portal** — separate user pools, credential stores, session policies, and token issuers. No credential is valid across portals.
2. **Drivers are not portal users** — SMS magic-link tokens, single-trip scope, auto-expiry.
3. **Cross-portal identity is never assumed** — a person who works at both a warehouse and a carrier gets two separate accounts.
4. **Cross-org data access** flows only through the shared Load object with per-load server-side authorization (Module F-2).
5. **Roadmap (P1+):** per-portal SSO (OIDC/SAML) for enterprise clients; MFA for Admin/Manager roles.

---

## 10. High-Level Data Model

| Entity | Key fields (abridged) | Notes |
|---|---|---|
| `Organization` | id, type (warehouse/carrier), name, settings JSON, feature flags | Tenant boundary |
| `OrgRelationship` | warehouse_org_id, carrier_org_id, status, invited_by | Gates tendering |
| `User` | id, org_id, role, name, email, phone, notification prefs | Scoped to one org |
| `Facility` | org_id, address, geo point, geofence radius, operating hours | A warehouse org may have several facilities (P1) |
| `Dock` | facility_id, label, equipment constraints, status | |
| `Load / Appointment` | id, warehouse_org_id, carrier_org_id, direction, state, time window, load details, dock_id, created_by | The shared object |
| `LoadEvent` | load_id, state_from, state_to, actor, timestamp, source (user/system/AI/geofence) | Immutable audit trail |
| `Vehicle` | carrier_org_id, type (truck/trailer), unit number | |
| `Driver` | carrier_org_id, name, phone, qualifications | Phone required for SMS |
| `Trip` | load_id, driver_id, truck_id, trailer_id, magic-link token, status | Driver-facing view of a load |
| `LocationPing` | trip_id, lat/lng, timestamp | Retained per privacy policy (§11) |
| `EtaPrediction` | trip_id, eta, confidence, basis (live/rules), model version | |
| `CarrierRecommendation` | load_id, carrier_org_id, score, factors JSON, manager decision | Feeds G-5 learning |
| `Notification` | recipient, channel, event, payload, delivery status | |

---

## 11. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Availability** | 99.5% monthly (MVP); dock board degradation = read-only cached view, never blank |
| **Performance** | Portal p95 page load < 2 s; dock board update latency < 10 s; SMS dispatch < 30 s from trigger |
| **SMS deliverability** | Registered A2P sender (e.g., toll-free verified / 10DLC), delivery-rate monitoring, fallback to secondary provider if delivery < 95% |
| **Location privacy** | Explicit per-trip driver consent; collection stops at trip end; pings retained 90 days then purged; drivers can stop sharing anytime (status updates still work) |
| **Multi-tenancy** | Hard data isolation per org; per-tenant AI models/rankings — no cross-tenant learning in MVP |
| **Security** | TLS everywhere; encryption at rest; least-privilege service accounts; secrets manager; quarterly dependency audits; SOC 2 Type I on the 12-month roadmap |
| **Auditability** | Immutable event log for all load transitions, permission changes, AI recommendations and human decisions |
| **Scalability (MVP ceiling)** | 50 warehouse orgs / 500 carrier orgs / 5k concurrent dock-board viewers / 100k notifications/day — headroom ~10× over pilot needs |
| **Browser support** | Latest 2 versions of Chrome/Safari/Edge; mobile web: iOS Safari + Android Chrome; functional on 3G (drivers) |
| **Accessibility** | WCAG 2.1 AA target for portals; driver UI: large touch targets, high contrast, minimal reading |
| **Internationalization** | Architecture i18n-ready; MVP ships English, Spanish planned for driver SMS/UI (P1) — large share of US drivers prefer Spanish |

---

## 12. Rollout Plan & Milestones

Rough sizing for a small senior team (2–4 engineers + founder). Adjust after sprint 0.

| Milestone | Contents | Exit criteria | Est. |
|---|---|---|---|
| **M0 — Foundations** | Tenancy, dual auth, org relationships, RBAC, audit log skeleton, CI/CD, feature-flag system | Two portals, separate logins, empty shells, orgs can link | wk 1–3 |
| **M1 — Scheduling** | Module A + B (calendar, dock config, manual assignment, dock board) | Warehouse can run a full day of appointments with no carrier side | wk 4–7 |
| **M2 — Carrier side** | Module D + E + F (tenders, dispatch, SMS magic links, driver mobile view, state machine) | A load moves tender → dispatched with real SMS | wk 8–10 |
| **M3 — Arrival warnings** | Module C + I (geofences, threshold alerts on rules-based ETA, escalation) | Crew alerted 30 min before real arrivals at pilot site | wk 11–12 |
| **M4 — AI features** | Module G + H (matching with approval gate + explainability; ETA model in shadow, then live) | Recommendations live with ≥ 1 pilot; ETA shadow accuracy measured | wk 13–16 |
| **M5 — Pilot hardening** | CSV imports, analytics on KPIs, bug burn-down, FDE runbook | 1 warehouse + 3 carriers running daily ops; KPI baseline captured | wk 17–18 |

**Pilot strategy:** start with one warehouse the founder has relationships with + 2–3 of its friendliest carriers. Historical loads imported via CSV on day one (G-4) so matching works immediately. FDE on-site during week 1 of each pilot — their observations feed the fast-follow list.

---

## 13. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Carrier-side adoption** — carriers won't log into yet another portal | Warehouse features lose their counterpart | Driver needs nothing but SMS; dispatcher onboarding < 30 min; warehouse invites its own carriers (relationship model pulls supply in) |
| **Driver location consent / privacy concerns** | Arrival warnings degrade to rules-based | Consent-first design, per-trip scope, visible indicator, manual fallback (C-4) keeps value even at low opt-in; measure opt-in rate per fleet |
| **AI cold start** — new warehouses have no carrier history | Matching looks useless at first login | CSV import in onboarding, preferred-carrier lists, honest "no recommendation" state (never fabricate) |
| **AI trust** — managers ignore or blindly accept recommendations | Feature dies or causes bad freight decisions | Explainability (G-3), approval gate (G-2), decision logging → tune ranking against real acceptance data |
| **SMS cost & deliverability** (carrier filtering, 10DLC registration) | Driver channel silently breaks | Registered sender, delivery monitoring, provider fallback, budget alerting per tenant |
| **FDE scope creep** — every client wants custom everything, core product stagnates | You become a consultancy, not a product company | Feature-flag discipline (§4); rule: custom work ships behind flags; 3-client rule before core graduation; PRD scope changes require explicit sign-off |
| **ETA model underperforms early** | Bad alerts erode crew trust fast | Shadow-mode launch (H-4) — crews never see model-driven alerts until accuracy target is proven at their site |
| **Incumbent tools** (dock scheduling point solutions, TMS suites) | Competitive pressure | Wedge is the *two-sided* handshake + arrival intelligence; most dock schedulers are warehouse-only, most TMS are carrier-only |

---

## 14. Open Questions (resolve before sprint 0 / during pilot)

1. **Pricing model** — per-warehouse-org subscription + free carrier side (recommended for network effects)? Per-load fee? Where does FDE time price in?
2. **Who pays for SMS** — absorbed in subscription (recommended at MVP volumes) or passed through?
3. **Integration targets** — which WMS/TMS systems do pilot clients run? Determines API/webhook priorities and the first FDE engagement.
4. **Multi-facility warehouses** — confirm whether pilot #1 is single-site (MVP assumes yes per org; facilities entity exists but UI is single-site-first).
5. **Rate visibility** — do tenders include rates, and does the warehouse want historical rate factored into matching (may be sensitive)?
6. **Detention tracking** — pilot sites will ask for it immediately; confirm it stays P1.
7. **Spanish-language driver SMS** — validate demand with pilot carriers; i18n plumbing is in NFRs either way.

---

## 15. Glossary

| Term | Meaning |
|---|---|
| **FDE** | Forward Deployed Engineer — engineer embedded with a client to customize/deploy the product on-site |
| **Tender** | An offer of a load from a shipper/warehouse to a carrier, which the carrier accepts, declines, or counters |
| **Dock / loading dock** | The bay where a truck is loaded/unloaded |
| **Dwell time** | Time a truck spends at a facility from arrival to departure |
| **Detention** | Dwell time beyond the free period in a carrier contract, billable to the shipper |
| **Geofence** | A virtual perimeter around the facility used to detect truck arrival |
| **BOL / POD** | Bill of Lading / Proof of Delivery — the freight paperwork (upload: P1; OCR: later) |
| **Lane** | An origin→destination route pattern (e.g., "Dallas → Atlanta") |
| **HOS** | Hours of Service — driver legal driving-time limits (roadmap input for ETA) |
| **Shadow mode** | Running a model silently alongside the production rules to measure accuracy before going live |

---

*End of PRD v0.1. Next step: review §14 open questions with your engineer partners, then break M0–M1 into tickets.*
