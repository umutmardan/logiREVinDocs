# DockFlow — Technical Architecture v0.1

> **Companion to:** DockFlow MVP PRD v0.1, Vision Module Addendum v0.1
> **Date:** 2026-09-14 · **Status:** Draft for engineering review
> **Mandate from founder:** stack-agnostic. Requirements are **robust, modular, reliable, scalable, safe**. Every choice below is justified against those five words — and every choice is replaceable without redesigning the system.

---

## 1. Architectural Principles

These five principles are the contract. Any future decision that violates one needs explicit sign-off.

### 1.1 Boring technology, sharp boundaries
We spend our innovation budget on the *product* (two-sided coordination, vision ground truth, ETA/matching), not on infrastructure. Every component is a mainstream, well-documented, hire-able technology. Novelty is a liability in a system a warehouse depends on at 5 AM.

### 1.2 Modularity through explicit seams
The system decomposes into services with narrow contracts (§3). A component can be rewritten, replaced, or scaled independently as long as its contract holds. This is what makes "stack-agnostic" true in practice: the stack is a per-service decision, not a marriage.

### 1.3 Events are the spine
State changes (load transitions, vision observations, notifications) are **append-only events first**, projections second. The event log is the audit trail (PRD J-4), the reconciliation input for the vision ledger, and the replay mechanism when something breaks. We never mutate history.

### 1.4 Degrade, never die
Every external dependency has a defined degradation mode (§7): dock board falls back to cached read-only, ETA falls back to rules, edge box buffers through WAN outages, SMS falls back to a secondary provider. "Blank screen" and "lost event" are not acceptable failure modes.

### 1.5 Safety is structural, not procedural
Tenant isolation is enforced by the database (row-level security), cross-org access is enforced by the load-authorization layer, and secrets never live in code. Humans can be careless; the architecture must not be.

---

## 2. Recommended Stack (with the "why" and the exit door)

| Layer | Recommendation | Why (robust / modular / reliable / scalable / safe) | Replaceable with |
|---|---|---|---|
| Core API & portals backend | **TypeScript / Node (NestJS)** | One language across portals, core API, and realtime gateway → small team moves fast; NestJS enforces modular structure by convention; massive hiring pool | Kotlin/Spring, C#/.NET, Go — contract is REST + events |
| Edge & AI services | **Python** | The vision and ML ecosystem *is* Python (detection models, ETA features, scoring); isolating it to edge + AI services keeps the web stack clean | — (this boundary is deliberate and permanent) |
| Database | **PostgreSQL** (managed) | Row-level security for tenant isolation (§5.2), LISTEN/NOTIFY for realtime triggers, JSONB for per-tenant config, proven at 100× our MVP ceiling | None planned — deepest dependency we take, deliberately |
| Cache / queue / realtime fan-out | **Redis** (managed) | Job queues (notifications, ETA refresh), pub/sub for dock-board updates, rate limiting | RabbitMQ/SQS for queues later if volume demands |
| Frontend (both portals + driver view) | **React + Next.js** (three apps, one shared component package) | SSR for portal speed (p95 < 2 s NFR), driver view is a lightweight mobile-first route in the carrier app; shared design system keeps the two portals consistent | Any SPA framework — portals are API clients |
| Edge detection | **YOLO-class open models (Ultralytics) + custom zone/crossing logic** | "Pallet crossed this line" is a solved problem; our value is the zone ledger, not the detector — so the detector is commodity and swappable | Any ONNX-runtime detector; models versioned per site |
| Infra | **Containers on a managed runtime** (e.g., AWS ECS/Fargate class), **Terraform** for everything | No servers to babysit at MVP scale; Terraform makes the environment reproducible and auditable | Any container host — no host-specific services in app code |
| SMS / email | **Primary + secondary SMS provider** (10DLC-registered), transactional email service | PRD NFR: fallback provider if delivery < 95%; both behind an internal notification interface | Any provider — they sit behind our interface |
| Auth | Managed identity per portal (**two separate user pools**), custom token service for driver magic links | Dual-auth is a *product requirement* (PRD §9.3); separate pools make cross-portal credential leakage structurally impossible | Any OIDC provider — we never build password crypto ourselves |
| Observability | OpenTelemetry traces + structured logs + metrics, one hosted backend | "Who approved this carrier and why" (J-4) and "why didn't that SMS send" must be answerable in minutes | Vendor-neutral by using OTel |

**What we explicitly do NOT adopt at MVP:** microservices mesh, Kubernetes, Kafka, a data lake, a separate analytics warehouse. A modular monolith for the core + three satellite services (realtime gateway, notifications worker, AI services) covers the PRD's entire scalability ceiling (§7.3) with a fraction of the operational surface. We add infrastructure when a measured limit forces it, not before.

---

## 3. System Topology

```
                        ┌─────────────────────────────────────────────┐
                        │                 EDGE (per site)              │
                        │  Cameras → Edge Box (detect, zone logic,     │
                        │  local footage buffer, event queue)          │
                        └───────────────┬─────────────────────────────┘
                                        │ HTTPS, mTLS, buffered/retry
                                        ▼
┌───────────────┐   ┌───────────────┐   ┌──────────────────────────────────┐
│ WAREHOUSE     │   │ CARRIER       │   │  CLOUD CORE                      │
│ PORTAL        │   │ PORTAL        │   │                                  │
│ (Next.js app) │   │ (Next.js app  │   │  ┌────────────────────────────┐  │
│               │   │ + driver      │   │  │ Core API (modular monolith)│  │
│ Auth pool A   │   │  mobile view) │   │  │  • tenancy & RBAC          │  │
│               │   │               │   │  │  • scheduling & docks      │  │
│               │   │ Auth pool B   │   │  │  • tenders & dispatch      │  │
│               │   │ + SMS magic-  │   │  │  • load state machine      │  │
│               │   │   link tokens │   │  │  • vision ingestion &      │  │
│               │   │               │   │  │    anomaly rules           │  │
│               │   │               │   │  │  • load authorization gate │  │
└───────┬───────┘   └───────┬───────┘   │  └────────────────────────────┘  │
        │                   │           │  ┌────────────────────────────┐  │
        │    REST + WSS     │           │  │ Realtime gateway (WS/SSE)  │──┼──► dock boards,
        └───────────────────┼───────────┤  └────────────────────────────┘  │    trip maps
                            │           │  ┌────────────────────────────┐  │
                            └──────────►│  │ Notifications worker       │──┼──► SMS (primary→
                                        │  │ (queue, retry, fallback)   │  │    fallback), email,
                                        │  └────────────────────────────┘  │    in-app
                                        │  ┌────────────────────────────┐  │
                                        │  │ AI services (Python)       │  │
                                        │  │  • ETA prediction          │  │
                                        │  │  • carrier matching        │  │
                                        │  └────────────────────────────┘  │
                                        │  PostgreSQL (RLS) · Redis        │
                                        └──────────────────────────────────┘
```

**Service inventory and contracts:**

| Service | Owns | Talks to others via | Never does |
|---|---|---|---|
| Core API | All domain state, tenancy, state machine, load authorization, vision event ingestion, anomaly rules | REST (external), domain events (internal) | Touch SMS providers or models directly — goes through workers/services |
| Realtime gateway | Dock boards, trip maps, live status push | Redis pub/sub subscription to domain events | Writes to the database (read-only projections) |
| Notifications worker | Event→message mapping, SMS/email dispatch, retry/backoff, provider failover, delivery logging | Consumes `notification.requested` events; writes delivery status back | Business logic beyond templating & routing |
| AI services | ETA prediction, carrier ranking, shadow-mode evaluation | Consume domain events; write predictions/recommendations via Core API | Auto-tendering (PRD G-2: no code path exists), cross-tenant data access |
| Edge box (per site) | Detection, zone logic, footage buffer, event buffering | mTLS HTTPS to Core API ingestion endpoint | Store tenant credentials, serve footage publicly |

---

## 4. Event Spine & State Machine

### 4.1 Append-only event log
Every state change is an immutable row: `LoadEvent`, `VisionEvent`, `NotificationEvent`, `AnomalyEvent`, `AuthzDecision`. Current state is a **projection** of the event stream — fast to read, always rebuildable.

- **Why:** PRD J-4 (audit) becomes free; vision reconciliation needs an unalterable ledger; "replay from events" is the ultimate recovery tool.
- **How:** events are written in the same transaction as the state transition (outbox pattern) — a worker publishes from the outbox to Redis/consumers. No lost events, no dual-write inconsistency.

### 4.2 Load state machine (PRD Module F) lives in exactly one place
The Core API owns the transition table: valid `from → to` pairs, required role/authorization, side effects (notify, re-sequence picker queue, escalate). Vision events *request* transitions (`source=vision`) and pass the same validation as human taps — the state machine doesn't care where truth comes from, only that it's authorized and legal.

### 4.3 Vision reconciliation loop
`VisionEvent`s arrive independently of system tasks. A reconciler continuously joins the movement ledger against active tasks/loads/appointments:
- match → mark event `logged`, link it
- no match → `unlogged`, feed anomaly rules (K-A1…K-A4)
- divergence (vision says trailer present, status says Departed) → `divergent` flag on the load, manager alerted

This is the technical heart of the vision module — and it's just a stream-join over the event log, no exotic machinery.

---

## 5. Data & Tenancy Architecture

### 5.1 Single database, hard row-level isolation
All tenant data lives in one Postgres cluster; **every tenant-scoped table carries `org_id`, enforced by Postgres Row-Level Security**. The application connects with a per-request context (`SET app.current_org`), so a buggy query returns nothing rather than another tenant's rows. Defense in depth: RLS at the database *and* org-scoping in every repository method.

### 5.2 The load authorization gate
Cross-org access (warehouse ↔ carrier) flows only through loads both orgs are party to (PRD F-2). One module — the **load authorization gate** — answers "can org X perform action Y on load Z"; every cross-org endpoint calls it. Claims bundles (vision module K-6) use the same gate when sharing evidence with a carrier.

### 5.3 Data classes and their homes

| Data class | Home | Retention | Notes |
|---|---|---|---|
| Domain state (orgs, users, loads, docks, fleet) | Postgres | Life of tenant | RLS-protected |
| Event log (all event types) | Postgres (append-only, partitioned by month) | 24 months hot, then archive | Audit + replay |
| Location pings | Postgres (partitioned) | 90 days, then purge (PRD §11 privacy) | Consent-scoped |
| ETA predictions / recommendations | Postgres | 24 months | Includes shadow-mode records |
| Raw video | **Edge box only**, rolling ~30-day ring buffer | On-site, never uploaded | K.7 legal posture |
| Event clips | Object storage (cloud), tenant-scoped, signed-URL access | 90 days hot → archive/purge per tenant | Access audit-logged |
| Per-tenant config & feature flags | Postgres JSONB + flag service | Life of tenant | FDE surface (PRD §4) |

### 5.4 Migrations and versioning
All schema changes via versioned migrations in CI; no manual console changes. Event schemas are versioned (`v1`, `v2`…) — old events remain replayable forever, which the audit requirement demands.

---

## 6. Edge Architecture (Vision Kit)

### 6.1 Hardware baseline
Industrial mini-PC (or Jetson-class where GPU density is needed), PoE switch, ≤12 cameras per standard kit. Sized to run detection on all streams at 5–10 fps — zone-crossing logic doesn't need full frame rate, which is what keeps the box cheap.

### 6.2 Software (all containerized, Docker Compose on the box)

| Container | Job |
|---|---|
| `stream-ingest` | Pull RTSP from cameras (new + reused security cams), health-check feeds |
| `detector` | Pallet / forklift / person / trailer detection on configured zones |
| `zone-engine` | Our code: zone polygons, crossing logic, dwell timers — turns detections into `VisionEvent`s. **This is the product; the detector is a commodity** |
| `footage-buffer` | Rolling local ring buffer; clip extraction on event or on cloud request |
| `uplink` | Local persistent queue → mTLS HTTPS to Core API; buffers through WAN outage, backfills on reconnect; exactly-once via event IDs |
| `agent` | Remote management: config pull, container updates, health heartbeat |

### 6.3 Site operations model
- **Offline-tolerant:** WAN loss never stops detection; events queue locally (days of headroom), dock-critical logic (door state) runs regardless.
- **Fleet-managed:** site config (zones, rules, thresholds) is pulled from the cloud — FDE edits zones in the portal, box picks them up. No SSH-ing into sites as a lifestyle; a per-site VPN overlay exists for emergencies only.
- **Secure by default:** mTLS client certificate per box (issued at provisioning, rotatable), no inbound ports, footage never exposed off-LAN, disk encrypted at rest.

---

## 7. Reliability, Scalability & Degradation

### 7.1 Failure modes and their answers (PRD §11 mapped to mechanisms)

| Failure | Behavior |
|---|---|
| Core API instance dies | Stateless API behind load balancer; health-checked; auto-replaced. Sessions are tokens — no sticky state to lose |
| Database failover | Managed Postgres with standby; events in the outbox survive; writes resume automatically |
| Redis down | Queues pause; outbox accumulates in Postgres; dock board serves last-known state (read-only, labeled "stale") |
| SMS provider degraded | Delivery-rate monitor trips at <95% → notifications worker fails over to secondary provider; dispatcher alerted after final failure (PRD I-2) |
| ETA model unavailable | Rules-based fallback ETA, labeled "estimated" (PRD H-2); alerts keep firing |
| Site WAN down | Edge box buffers events + footage locally; backfills on reconnect; cloud marks site "offline" in UI |
| Edge box dies | Cameras still record if the client's NVR exists; cloud alerts ops on missed heartbeat; box is a swap-and-reprovision unit (config lives in cloud) |
| Whole cloud region impaired | Dock board and driver view degrade to cached read-only; never blank (PRD NFR) |

### 7.2 Performance budget
Portal p95 < 2 s (SSR + edge-cached static assets); dock-board update latency < 10 s (event → Redis → WebSocket push, typically < 1 s); SMS dispatch < 30 s from trigger (queue-based, not request-path).

### 7.3 Scalability ceiling and headroom
PRD §11 ceiling: 50 warehouse orgs / 500 carrier orgs / 5k concurrent dock-board viewers / 100k notifications/day. The recommended topology hits this with single-digit container counts and a modest managed Postgres — roughly **10× headroom** as specified. Growth levers, in order, when measured limits approach: read replicas → table partitioning → notifications on a dedicated queue broker → splitting realtime gateway. Each is an incremental change, not a re-architecture, because the seams already exist.

---

## 8. Security & Safety

| Domain | Mechanism |
|---|---|
| Portal auth | Two separate managed user pools (PRD §9.3); short-lived sessions; MFA-ready for Admin/Manager (roadmap) |
| Driver auth | Single-trip, single-purpose magic-link tokens; expire at trip completion +24 h; no password surface at all |
| API authorization | RBAC matrix (PRD §9) enforced server-side per endpoint; cross-org only via the load authorization gate (§5.2) |
| Tenant isolation | Postgres RLS + org-scoped repositories (§5.1); isolation covered by automated authorization tests in CI |
| Edge security | Per-box mTLS certs, no inbound listeners, encrypted disk, signed config, least-privilege cloud credentials (ingest-only) |
| Secrets | Cloud secrets manager; nothing in env files or repos; per-service least-privilege |
| Video & privacy | Raw footage on-site only; no face recognition; clip access via short-lived signed URLs, fully audit-logged (K.7) |
| Audit | Append-only event log + authz decisions logged; exportable (J-4) |
| Supply chain | Dependency scanning + quarterly audits (PRD §11); containers built from pinned, scanned base images |
| Transport | TLS everywhere; mTLS on the edge uplink |

---

## 9. Delivery & Operations

### 9.1 Environments
`dev` (per-engineer, ephemeral) → `staging` (prod-shaped, synthetic tenants + a camera simulator feeding fake VisionEvents) → `prod`. The camera simulator is a first-class citizen: it lets us build the entire vision pipeline, anomaly rules, and portal UX **before the first physical site exists**.

### 9.2 CI/CD
PR → tests (unit, authorization matrix, state-machine transition coverage, API contract tests) → build containers → deploy staging → smoke suite → promote. Database migrations run as a gated step. Feature flags gate unfinished work so `main` is always deployable (PRD §4 FDE discipline starts in our own repo).

### 9.3 Observability
OTel traces across API → outbox → workers → providers; structured logs with tenant/load/event correlation IDs; dashboards for the PRD's own NFRs (SMS delivery rate, dock-board latency, edge heartbeat age, ETA shadow accuracy). Alerts page a human only for user-visible degradation.

### 9.4 FDE extension points (architectural, not aspirational)
Per-tenant config store, feature flags, public REST API + webhooks (appointment created, status changed, arrival predicted, anomaly raised), CSV import/export (J-5), and per-site zone/rule configuration pulled by edge boxes. An FDE customizes a client by **configuring and integrating**, never by branching.

---

## 10. Build vs. Buy Register

| Capability | Decision | Rationale |
|---|---|---|
| Object detection models | **Buy/borrow** (open models) | Commodity; our moat is the zone ledger + platform integration |
| Zone/crossing logic, reconciliation, anomaly rules | **Build** | This IS the vision product (K.4) |
| Auth / identity | **Buy** (managed pools) | Never roll our own credential crypto |
| SMS / email | **Buy** (two providers, behind our interface) | Deliverability is a vendor problem; failover is ours |
| ETA / matching models | **Build** (Python services, shadow-mode first per H-4) | Core differentiation; starts rules-based, graduates to ML |
| Realtime infra | **Build thin** (Redis pub/sub + WS gateway) | Simple at our scale; no managed realtime vendor lock needed yet |
| Warehouse vision vendor (white-label) | **Rejected** (K.4) | Reselling someone else's margin; revisit only if pilot timeline forces it |

---

## 11. Open Technical Questions (resolve in sprint 0)

1. **Cloud provider + region** — pick once, near pilot sites; Terraform keeps it portable.
2. **SMS provider pair** — primary + fallback selection; 10DLC registration lead time is on the critical path for M2.
3. **Edge box BOM** — mini-PC vs. Jetson per site density; finalize in the V0 site survey (K.9).
4. **Event volume ceiling for the outbox** — load-test the outbox pattern at 10× expected VisionEvent rates before M4.
5. **Clip storage class & lifecycle automation** — cost modeling at 12 cameras × N events/day × 90 days.
6. **WMS reconciliation inputs** — CSV-first (guaranteed) vs. API (per client); determines the reconciler's adapter interface.

---

## 12. How This Maps to the Roadmap

| Milestone | Architectural deliverable |
|---|---|
| M0 | Repo skeleton, CI/CD, Terraform envs, dual auth pools, RLS tenancy, outbox + event log, feature flags |
| M1 | Core API scheduling/dock modules, realtime gateway, dock board |
| M2 | Tender/dispatch modules, notifications worker + SMS providers, magic-link token service, driver view |
| M3 | Geofence service, threshold alerting on rules ETA, escalation |
| M4 | AI services (matching + shadow ETA), **camera simulator**, vision ingestion endpoint, reconciler, anomaly rules |
| M5 + V0 | First physical edge box, fleet management agent, dock-door ground truth live at pilot site |

---

*End of Technical Architecture v0.1. Next: engineering data model + API specification (schemas, endpoints, webhook contracts) so M0 can be ticketed.*
