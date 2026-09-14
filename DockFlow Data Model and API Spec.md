# DockFlow — Engineering Data Model & API Specification v0.1

> **Companion to:** DockFlow MVP PRD v0.1, Vision Module Addendum v0.1, Technical Architecture v0.1
> **Date:** 2026-09-14 · **Status:** Draft — the ticketable specification for M0 onward
> **Conventions apply globally (§1) — read them before any table or endpoint.**

---

## 1. Global Conventions

| Convention | Rule |
|---|---|
| IDs | `uuid` primary keys, generated application-side (`gen_random_uuid()` acceptable); external-facing references additionally carry human-readable codes (e.g. `load_ref` = `LD-2026-00412`) |
| Timestamps | `timestamptz` everywhere, UTC; names: `created_at`, `updated_at`, event time = `occurred_at` |
| Tenancy | Every tenant-scoped table has `org_id uuid not null references organizations(id)` and an RLS policy `using (org_id = current_setting('app.current_org')::uuid)` |
| Soft state | No soft deletes on event tables (append-only). Domain entities use `status`/`deactivated_at` instead of `delete` |
| Enums | Postgres enum types, versioned by migration; new values are additive-only |
| JSON columns | `jsonb` with a JSON Schema documented in this spec; used for per-tenant config, event payloads, model factors |
| API shape | REST, JSON, snake_case fields; errors RFC 7807 (`application/problem+json`); pagination: cursor-based (`?cursor=`, `?limit=` ≤ 200); idempotent writes via `Idempotency-Key` header on POST |
| Versioning | `/v1/` path prefix; webhook/event payloads carry `schema_version` |
| Auth | `Authorization: Bearer <token>`; token audience identifies the portal pool (`wh` / `car`) or driver trip token (`drv`) or edge box certificate identity (`edge`) |

### 1.1 Authorization gate notation
Endpoints marked **`[load-gate]`** authorize against the load authorization gate (Architecture §5.2): caller's org must be a party to the specific load. Endpoints marked **`[rbac:role+]`** enforce the PRD §9 matrix server-side.

---

## 2. Core Schema (Postgres)

### 2.1 Tenancy & Identity

```sql
create type org_type as enum ('warehouse', 'carrier');

create table organizations (
  id            uuid primary key,
  type          org_type not null,
  name          text not null,
  settings      jsonb not null default '{}',   -- geofence_radius_m, alert_thresholds_min[],
                                               -- slot_duration_min, branding{logo,color,sender_name}
  feature_flags jsonb not null default '{}',   -- per-tenant flags (FDE surface, PRD J-3)
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

create type relationship_status as enum ('invited', 'active', 'suspended', 'revoked');

create table org_relationships (               -- PRD J-1: gates all tendering
  id               uuid primary key,
  warehouse_org_id uuid not null references organizations(id),
  carrier_org_id   uuid not null references organizations(id),
  status           relationship_status not null default 'invited',
  invited_by       uuid not null,              -- user id from inviting org
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  unique (warehouse_org_id, carrier_org_id)
);

create type wh_role  as enum ('admin', 'manager', 'picker', 'worker');
create type car_role as enum ('admin', 'dispatcher');
-- users.role is validated against organizations.type in app layer

create table users (
  id         uuid primary key,
  org_id     uuid not null references organizations(id),
  role       text not null,                    -- wh_role | car_role
  name       text not null,
  email      citext not null,
  phone      text,                             -- E.164
  notif_prefs jsonb not null default '{}',     -- per event-type channel prefs (PRD I-1)
  external_auth_id text not null,              -- subject in the portal's identity pool
  deactivated_at timestamptz,
  created_at timestamptz not null default now(),
  unique (org_id, email)
);
-- RLS: org members see own org's users only
```

### 2.2 Facilities, Docks, Fleet

```sql
create table facilities (
  id               uuid primary key,
  org_id           uuid not null references organizations(id),  -- warehouse orgs only (MVP: 1 per org)
  name             text not null,
  address          jsonb not null,             -- {line1, city, state, zip, country}
  geo              geography(point) not null,
  geofence_radius_m int not null default 1609, -- 1 mi (PRD C-2)
  operating_hours  jsonb not null,             -- {mon:[["06:00","22:00"]], ...}
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now()
);

create type equipment_type as enum ('dry_van', 'reefer', 'flatbed');
create type dock_status as enum ('free', 'occupied', 'reserved', 'out_of_service');

create table docks (
  id           uuid primary key,
  facility_id  uuid not null references facilities(id),
  label        text not null,                  -- "Dock 4"
  constraints  jsonb not null default '{}',    -- {equipment:[reefer], max_weight_kg, notes}
  status       dock_status not null default 'free',
  created_at   timestamptz not null default now(),
  unique (facility_id, label)
);

create type vehicle_kind as enum ('truck', 'trailer');

create table vehicles (
  id          uuid primary key,
  org_id      uuid not null references organizations(id),   -- carrier org
  kind        vehicle_kind not null,
  unit_number text not null,
  equipment   equipment_type not null default 'dry_van',
  deactivated_at timestamptz,
  created_at  timestamptz not null default now(),
  unique (org_id, kind, unit_number)
);

create table drivers (                          -- carrier-side record; NOT a portal user (PRD §5.3)
  id          uuid primary key,
  org_id      uuid not null references organizations(id),
  name        text not null,
  phone       text not null,                    -- E.164, required (PRD D-4)
  qualifications jsonb not null default '[]',   -- ["reefer","hazmat",...]
  deactivated_at timestamptz,
  created_at  timestamptz not null default now(),
  unique (org_id, phone)
);
```

### 2.3 Load / Appointment — the shared object

```sql
create type load_direction as enum ('inbound', 'outbound');
create type load_state as enum (
  'draft', 'pending_approval', 'scheduled', 'tendered', 'accepted',
  'dispatched', 'en_route', 'arrived', 'at_dock', 'loading_unloading',
  'completed', 'cancelled', 'reschedule_requested'
);

create table loads (
  id               uuid primary key,
  load_ref         text not null unique,       -- LD-2026-00412
  warehouse_org_id uuid not null references organizations(id),
  carrier_org_id   uuid references organizations(id),      -- null until tendered/accepted
  facility_id      uuid not null references facilities(id),
  direction        load_direction not null,
  state            load_state not null default 'draft',
  window_start     timestamptz not null,
  window_end       timestamptz not null,
  details          jsonb not null,             -- {po_refs[], pallets, weight_kg, commodity,
                                               --  equipment, special_instructions, rate_cents?}
  dock_id          uuid references docks(id),
  created_by       uuid not null,              -- user id (warehouse side)
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  check (window_end > window_start)
);
create index on loads (warehouse_org_id, window_start);
create index on loads (carrier_org_id, window_start) where carrier_org_id is not null;
create index on loads (facility_id, dock_id, window_start) where dock_id is not null;
```

**State transition table** (owned by Core API, Architecture §4.2 — stored as config, enforced in code):

| From | To | Actor | Side effects |
|---|---|---|---|
| draft | pending_approval | wh manager/admin | notify manager |
| pending_approval | scheduled | wh manager/admin | notify carrier if carrier-requested |
| scheduled | tendered | wh manager/admin (**human only**, PRD G-2) | tender → carrier inbox, expiry set |
| tendered | accepted / cancelled | car dispatcher / wh | notify opposite side |
| accepted | dispatched | car dispatcher | assign trip → SMS to driver (≤30 s, PRD D-2) |
| dispatched | en_route | driver tap **or** dispatcher | notify warehouse |
| en_route | arrived | geofence **or** driver **or** vision | C-2 dock alert |
| arrived | at_dock | worker / vision | dock → occupied |
| at_dock | loading_unloading | worker / driver / vision | dwell timer starts |
| loading_unloading | completed | worker / driver | dwell recorded → scorecard |
| any (pre-completed) | cancelled / reschedule_requested | per RBAC | notify opposite side |

```sql
create type event_source as enum ('user', 'system', 'ai', 'geofence', 'vision');

create table load_events (                     -- append-only (PRD F-1, J-4)
  id          uuid primary key,
  load_id     uuid not null references loads(id),
  state_from  load_state,
  state_to    load_state not null,
  actor_type  text not null,                   -- 'user' | 'driver' | 'system'
  actor_id    uuid,                            -- user/driver id when applicable
  source      event_source not null,
  verification text not null default 'self_reported',  -- 'vision_confirmed' | 'self_reported' | 'divergent'
  occurred_at timestamptz not null default now(),
  payload     jsonb not null default '{}'
) partition by range (occurred_at);
create index on load_events (load_id, occurred_at);
```

### 2.4 Trips, Location, ETA

```sql
create table trips (
  id           uuid primary key,
  load_id      uuid not null unique references loads(id),   -- 1 active trip per load (MVP)
  driver_id    uuid not null references drivers(id),
  truck_id     uuid not null references vehicles(id),
  trailer_id   uuid references vehicles(id),
  token_hash   text not null unique,           -- magic-link token, hashed at rest
  token_expires_at timestamptz not null,       -- trip completion +24 h (PRD E-1)
  location_consent boolean not null default false,         -- per-trip (PRD E-3)
  status       text not null default 'assigned',
  created_at   timestamptz not null default now()
);

create table location_pings (                  -- retained 90 days (PRD §11), then purged by job
  id         bigint generated always as identity,
  trip_id    uuid not null references trips(id),
  geo        geography(point) not null,
  accuracy_m real,
  recorded_at timestamptz not null default now(),
  primary key (recorded_at, id)
) partition by range (recorded_at);
create index on location_pings (trip_id, recorded_at desc);

create table eta_predictions (
  id            bigint generated always as identity,
  trip_id       uuid not null references trips(id),
  eta           timestamptz not null,
  confidence_min int not null,                 -- ± minutes band (PRD H-1)
  basis         text not null,                 -- 'live_location' | 'rules'
  model_version text not null,
  shadow        boolean not null default false,-- H-4 shadow-mode rows
  created_at    timestamptz not null default now()
);
create index on eta_predictions (trip_id, created_at desc);
```

### 2.5 AI Matching

```sql
create table carrier_recommendations (
  id             uuid primary key,
  load_id        uuid not null references loads(id),
  carrier_org_id uuid not null references organizations(id),
  score          numeric(5,4) not null,
  factors        jsonb not null,               -- explainability (PRD G-3):
                                               -- [{factor:"lane_history", detail:"18 loads Dallas→Atlanta",
                                               --   contribution:0.31}, ...]
  decision       text,                         -- null=pending | 'approved' | 'declined' (G-5 learning)
  decided_by     uuid,
  decided_at     timestamptz,
  created_at     timestamptz not null default now(),
  unique (load_id, carrier_org_id)
);
```

### 2.6 Notifications

```sql
create type notif_channel as enum ('sms', 'email', 'in_app');
create type notif_status as enum ('queued','sent','delivered','failed','failed_final');

create table notifications (
  id           uuid primary key,
  org_id       uuid not null references organizations(id),
  recipient    jsonb not null,                 -- {user_id} | {driver_id, phone}
  channel      notif_channel not null,
  event_type   text not null,                  -- catalog in PRD I-2
  payload      jsonb not null,                 -- includes deep_link (PRD I-3)
  provider     text,                           -- which SMS/email provider handled it
  status       notif_status not null default 'queued',
  attempts     int not null default 0,
  load_id      uuid references loads(id),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);
create index on notifications (org_id, created_at desc);
```

### 2.7 Audit

```sql
create table audit_log (                       -- permission changes, authz decisions, bundle shares (PRD J-4)
  id          bigint generated always as identity,
  org_id      uuid,
  actor_id    uuid,
  action      text not null,                   -- 'user.invited', 'role.changed', 'claims_bundle.shared', ...
  target      jsonb not null,
  occurred_at timestamptz not null default now(),
  primary key (occurred_at, id)
) partition by range (occurred_at);
```

---

## 3. Vision Schema (Module K)

```sql
create table edge_boxes (
  id          uuid primary key,
  facility_id uuid not null references facilities(id),
  serial      text not null unique,
  cert_fingerprint text not null,              -- mTLS identity (Architecture §6.3)
  last_heartbeat_at timestamptz,
  sw_version  text,
  provisioned_at timestamptz not null default now()
);

create type camera_position as enum ('dock_door', 'aisle_end', 'staging', 'gate');

create table cameras (
  id           uuid primary key,
  edge_box_id  uuid not null references edge_boxes(id),
  position     camera_position not null,
  label        text not null,                  -- "Aisle 7 mouth"
  stream_uri   text not null,                  -- site-local RTSP
  source       text not null default 'new',    -- 'new' | 'reused'
  zone_ids     uuid[] not null default '{}',   -- zones this camera covers
  created_at   timestamptz not null default now()
);

create type zone_type as enum ('rack','aisle','staging','dock_door','apron','yard');

create table zones (
  id         uuid primary key,
  facility_id uuid not null references facilities(id),
  label      text not null,                    -- "Staging lane 3"
  type       zone_type not null,
  is_storage boolean not null,                 -- false → feeds K-A2 forgotten-pallet rule
  polygon    jsonb,                            -- camera-relative coords, per camera
  created_at timestamptz not null default now(),
  unique (facility_id, label)
);

create table vision_events (                   -- the movement ledger (Addendum K-1); append-only
  id           uuid primary key,               -- edge-generated UUID → exactly-once on retry
  facility_id  uuid not null references facilities(id),
  camera_id    uuid not null references cameras(id),
  object_class text not null,                  -- 'pallet' | 'forklift_with_pallet' | 'trailer' | 'person'
  zone_from_id uuid references zones(id),
  zone_to_id   uuid references zones(id),
  confidence   real not null,
  clip_ref     text,                           -- object-storage key, populated async
  linked_load_id uuid references loads(id),    -- set by reconciler
  linked_task_ref text,                        -- external WMS task ref when reconciled
  reconciliation text not null default 'pending',  -- 'pending' | 'logged' | 'unlogged' | 'divergent'
  occurred_at  timestamptz not null,
  ingested_at  timestamptz not null default now()
) partition by range (occurred_at);
create index on vision_events (facility_id, occurred_at desc);
create index on vision_events (linked_load_id) where linked_load_id is not null;
create index on vision_events (facility_id, reconciliation) where reconciliation = 'unlogged';

create type anomaly_rule as enum ('unlogged_movement','forgotten_pallet','departed_but_staged','door_count_mismatch');
create type anomaly_status as enum ('open','acknowledged','resolved','dismissed');

create table anomalies (
  id              uuid primary key,
  facility_id     uuid not null references facilities(id),
  rule            anomaly_rule not null,
  vision_event_id uuid references vision_events(id),
  load_id         uuid references loads(id),
  detail          jsonb not null,              -- rule-specific: {dwell_min: 317, zone:"staging 3"}
  status          anomaly_status not null default 'open',
  assigned_to     uuid,
  resolution_note text,
  created_at      timestamptz not null default now(),
  resolved_at     timestamptz
);
create index on anomalies (facility_id, status, created_at desc);

create table claims_bundles (
  id           uuid primary key,
  load_id      uuid not null references loads(id),
  events       jsonb not null,                 -- frozen timeline snapshot
  clip_refs    text[] not null default '{}',
  generated_by uuid not null,
  shared_with_org_id uuid references organizations(id),  -- set on share; audit-logged (K-6)
  created_at   timestamptz not null default now()
);
```

**Per-tenant vision config** lives in `organizations.settings.vision`:

```json
{
  "forgotten_pallet_alert_hours": 4,
  "forgotten_pallet_escalate_hours": 24,
  "door_count_mismatch_enabled": true,
  "clip_retention_days": 90,
  "notify": { "unlogged_movement": ["manager"], "forgotten_pallet": ["manager","worker"] }
}
```

---

## 4. Domain Events (outbox → internal consumers)

Envelope for every event published to Redis/consumers:

```json
{
  "id": "uuid",
  "type": "load.state_changed",
  "schema_version": 1,
  "org_id": "uuid",
  "occurred_at": "timestamptz",
  "data": { "...event-specific..." }
}
```

| Event type | Emitted when | Key `data` fields | Primary consumers |
|---|---|---|---|
| `load.created` | Load inserted | load_id, warehouse_org_id, window | Matching service (G) |
| `load.state_changed` | Any legal transition | load_id, from, to, source, verification | Realtime gateway, notifications, ETA service |
| `load.tendered` | scheduled→tendered | load_id, carrier_org_id, expires_at | Notifications (carrier inbox + email) |
| `trip.dispatched` | Trip created | trip_id, driver phone, load summary | Notifications → SMS ≤30 s (D-2) |
| `location.ping_received` | Driver ping accepted | trip_id, geo | ETA service, geofence evaluator |
| `eta.updated` | New prediction written | trip_id, eta, basis, confidence | Realtime gateway, threshold evaluator (C-1) |
| `geofence.entered` | Ping inside facility radius | trip_id, facility_id | State machine (→arrived), dock alert (C-2) |
| `vision.event_ingested` | VisionEvent persisted | vision_event_id, facility_id, zones, object_class | Reconciler |
| `anomaly.raised` | Anomaly rule fired | anomaly_id, rule, facility_id, load_id? | Notifications per tenant vision config |
| `notification.requested` | Any module requests a message | notification_id | Notifications worker |
| `notification.failed_final` | Retries exhausted | notification_id, channel | Notifications → dispatcher alert (I-2) |

---

## 5. REST API — `/v1`

Portal prefix determines auth pool: `warehouse.dockflow.app/api/v1/...` vs `carrier.dockflow.app/api/v1/...`. Same core API, different audience claim required.

### 5.1 Warehouse Portal

| Method & path | Purpose | Authz |
|---|---|---|
| `POST /appointments` | Create load/appointment (A-1) | rbac:manager+ |
| `GET /appointments?from=&to=&dock_id=&state=&direction=` | Calendar feed (A-3) | rbac:worker+ |
| `PATCH /appointments/{id}` | Edit; `state` transitions validated by state machine | rbac:manager+ |
| `POST /appointments/{id}/approve` / `/decline` / `/reschedule` | Decisions on carrier requests (A-2) | rbac:manager+ |
| `POST /loads/{id}/dock-suggestion` | System-suggested dock w/ one-line reason (B-1) | rbac:worker+ |
| `POST /loads/{id}/assign-dock` | Confirm/override dock (B-1) | rbac:worker+ |
| `GET /dock-board` | Live board snapshot (B-2); WS channel `dock-board:{facility_id}` for push | rbac:worker+ |
| `GET /staging-queue` | Outbound queue sequenced by predicted arrival (B-3) | rbac:picker+ |
| `GET /loads/{id}/recommendations` | Ranked carriers + factors (G-1/G-3) | rbac:manager+ |
| `POST /loads/{id}/tender` | Send tenders to selected carriers — **the only tendering path; no auto path exists (G-2)** | rbac:manager+ |
| `GET /carriers` / `POST /carriers/{org_id}/invite` | Directory & relationship invite (J-1) | rbac:manager+ |
| `GET /anomalies?status=` / `POST /anomalies/{id}/ack` / `/resolve` | Vision anomaly triage (K.5.2) | rbac:worker+ (view) / manager+ (resolve) |
| `GET /loads/{id}/claims-bundle` / `POST /loads/{id}/claims-bundle/share` | Generate / share evidence bundle (K-6) | rbac:manager+; share → `[load-gate]` + audit |
| `GET /vision/events?zone_id=&reconciliation=&from=&to=` | Movement ledger search (K-3) | rbac:worker+ |
| `GET /vision/clips/{ref}` | Short-lived signed URL for clip (audit-logged, K.7) | rbac:worker+ |
| `GET /audit-log?action=&from=` | Audit export (J-4) | rbac:admin |
| `POST /imports/{entity}` / `GET /exports/{entity}` | CSV import/export: `loads`, `carriers`, `docks`, `users` (J-5, G-4) | rbac:admin |
| `GET/PATCH /settings` | Facility, geofence, thresholds, vision config, flags (A-4, J-3) | rbac:admin |

### 5.2 Carrier Portal

| Method & path | Purpose | Authz |
|---|---|---|
| `GET /tenders?status=` | Tender inbox (D-1) | rbac:dispatcher+ |
| `POST /tenders/{load_id}/accept` / `/decline` / `/counter` | Respond; counter proposes new window (D-1) | rbac:dispatcher+, `[load-gate]` |
| `POST /loads/{id}/dispatch` | Assign driver+truck+trailer → creates trip, fires SMS (D-2) | rbac:dispatcher+, `[load-gate]` |
| `GET /trips?active=true` | Active trips board with ETA + risk flags (D-3) | rbac:dispatcher+ |
| `GET /loads/{id}` | Shared load view, carrier-scoped fields only (F-1) | `[load-gate]` |
| `GET/POST/PATCH /fleet/vehicles` , `/drivers` | Fleet & driver management (D-4) | rbac:admin |
| `GET /warehouses` / `POST /warehouses/{org_id}/accept` | Relationship management (J-1) | rbac:admin |
| `GET /loads/{id}/claims-bundle` | View bundle shared with this org (K-6) | `[load-gate]` |

### 5.3 Driver mobile (magic-link token, audience `drv`)

| Method & path | Purpose | Authz |
|---|---|---|
| `GET /t/{token}` | Resolve token → trip view (E-1); token single-trip, expiring | token only |
| `POST /t/{token}/status` | Big-thumb transitions (E-2); state machine validates | token, own trip |
| `POST /t/{token}/location` | Ping batch `{pings:[{lat,lng,recorded_at,accuracy}]}` (E-3) | token, consent=true |
| `POST /t/{token}/consent` | Opt in/out of location sharing for this trip | token |

### 5.4 Edge ingestion (mTLS, audience `edge`)

| Method & path | Purpose |
|---|---|
| `POST /edge/v1/vision-events` | Batch ingest `[VisionEvent]`; idempotent by event UUID; box retries until acked |
| `GET /edge/v1/config` | Box pulls zones, rule thresholds, camera assignments (FDE edits in portal → box pulls) |
| `POST /edge/v1/heartbeat` | Liveness + sw_version; missed heartbeats alert ops |
| `POST /edge/v1/clips/{event_id}` | Clip upload on event / on cloud request (K-1) |

### 5.5 Public API & webhooks (FDE integration surface, PRD §4)

API keys per tenant org (`Authorization: Bearer dk_live_...`), scoped to that org's data, same endpoints as its portal type. Webhook subscriptions:

```
POST /webhooks { "url": "...", "events": [...], "secret": "whsec_..." }
```

| Webhook event | Payload (abridged) |
|---|---|
| `appointment.created` | load object |
| `appointment.state_changed` | load_id, from, to, source, occurred_at |
| `arrival.predicted` | trip_id, load_id, eta, confidence_min, basis |
| `anomaly.raised` | anomaly_id, rule, facility, load_ref?, zone labels |
| `claims_bundle.shared` | load_ref, bundle_url (signed, 24 h) |

Delivery: signed (`X-Dockflow-Signature: t=...,v1=HMAC_SHA256(secret, t.body)`), retried with backoff 5 times, per-subscription failure log in portal.

---

## 6. Realtime Channels (WebSocket/SSE)

| Channel | Audience | Payloads |
|---|---|---|
| `dock-board:{facility_id}` | Warehouse roles | dock status, arrival alerts (C-1/C-2), load state changes |
| `trips:{carrier_org_id}` | Dispatchers | ETA updates, risk flags (D-3) |
| `load:{load_id}` | Both portals (`[load-gate]` on subscribe) | timeline events, status changes |
| `anomalies:{facility_id}` | Manager/worker | anomaly raised/acked |

Fallback: SSE → polling every 15 s on degraded connections; dock board must keep updating on a TV browser (B-2).

---

## 7. Testing Hooks (sprint-0 acceptance)

| Suite | Covers |
|---|---|
| Authorization matrix | Every endpoint × every role × cross-tenant attempts → must 403/404 (F-2, §5.1) |
| State machine | Every legal/illegal transition, incl. vision-source transitions and divergent flags |
| Outbox | Event loss impossible under worker crash; replay reproduces projections |
| Reconciler | Synthetic VisionEvents (camera simulator) → logged/unlogged/divergent classification accuracy |
| Notification failover | Provider A down → failover fires; `failed_final` alerts dispatcher |
| Magic links | Expiry, single-trip scope, cross-trip token rejection |

---

*End of Data Model & API Spec v0.1. With this document, M0–M1 can be broken into tickets: each table = migration ticket, each endpoint group = implementation ticket, each §7 suite = quality gate.*
