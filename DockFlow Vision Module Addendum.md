# DockFlow — Vision Module (Module K) PRD Addendum

> **Addendum to:** DockFlow MVP PRD v0.1 (2026-08-18)
> **Version:** 0.1 (Draft) · **Date:** 2026-09-14
> **Scope note:** The base PRD excludes WMS functionality (§6.3). Module K does **not** reverse that — it is not inventory management, pick-path, or put-away logic. It is a **physical ground-truth layer**: cameras verify and record what physically happened, and reconcile it against what the system thinks happened.

---

## K.1 Problem Statement (from operator experience)

The founding team's warehouse experience identifies the primary pain:

> Pickers move pallets by forklift in the course of daily work and sometimes forget to return them. The pallet remains "at its location" in the system while physically sitting in a staging lane, an aisle corner, or the wrong rack. It surfaces days or weeks later at cycle count — or never, becoming shrinkage.

Consequences:

- **Lost pallets** — inventory exists physically but is unfindable; written off or re-ordered.
- **Shrinkage & disputes** — pallet-count mismatches vs. BOL discovered after the carrier leaves, when claims are nearly unwinnable; no evidence either way.
- **Existing cameras don't help** — security CCTV covers entrances and yards, not individual aisles, and footage is never tied to operational events. Reviewing it is hours of scrubbing per incident.

**Root cause:** the system of record only knows what humans told it. Every untracked forklift move is a silent divergence between the record and the floor.

---

## K.2 Core Design Decision: Movement Ledger, Not Identity

Module K deliberately does **not** attempt camera-based pallet identification (label/barcode reading). Shrink wrap, damaged labels, lighting, and angles make it unreliable at commodity-camera quality.

Instead, the system maintains a **movement ledger**: every observed pallet zone transition is logged — *what* (a pallet, a forklift carrying a pallet), *from zone → to zone*, *when*, *with/without an associated system task*. Reconciliation against WMS/order events separates logged moves from unlogged ones.

When a pallet is "lost" in the system, the answer to *"where did it actually go?"* is a **ledger lookup** (last observed zone + timestamp + video clip), not a warehouse-wide search.

| Requirement | Rationale |
|---|---|
| Track zone transitions, not pallet identity | Identity is unreliable at this camera class; transitions are not |
| Every movement event linked to system context (or explicitly "unlogged") | The divergence between record and floor IS the product |
| Every event carries a retrievable video clip | Turns disputes and investigations from hours into seconds |
| Anomaly alerts fire same-shift, not at cycle count | Value decays with time; a forgotten pallet found in 4 hours is found |

---

## K.3 Camera Topology: Choke-Point Coverage

Per-rack or per-aisle-interior cameras are rejected: racking occludes sightlines, and camera count scales with rack rows (100+ cameras for a mid-size facility). Instead, cover the **choke points** every pallet must pass through:

| Camera position | Covers | Typical count (mid-size site) |
|---|---|---|
| Dock doors (interior, angled at threshold) | Trailer presence, pallet count in/out, door open/close | 1 per active door (4–8) |
| Aisle ends (wide-angle, aimed down the aisle mouth) | Everything entering/exiting each aisle | 1 per 1–2 aisle ends (6–12) |
| Staging lanes / wrap machine / outbound lanes | Freight staged, left behind, wrapped | 2–4 |
| Yard gate (optional, existing CCTV often reusable if IP-based) | Trailer in/out, plate capture (P1) | 0–2 |

**Standard Vision Kit (the product):** edge inference box + up to 12 PoE cameras + install + zone labeling + tuning. Sites needing more are quoted as FDE work — same scope discipline as PRD §4.

**Existing camera reuse:** most facilities have security CCTV. If cameras are IP-based, reachable on the LAN, and positioned at a useful choke point, the edge box can ingest their streams — potentially reducing new hardware to zero at doors/gate. Site survey (K.9) determines reuse vs. new.

---

## K.4 Architecture: Edge Inference, Event Stream Up

```
┌──────────────────────── SITE (client facility) ────────────────────────┐
│  PoE IP cameras (choke points)                                         │
│       │  RTSP streams (never leave the building)                        │
│       ▼                                                                 │
│  Edge inference box (mini-PC / Jetson-class)                            │
│   • pallet / forklift / person / trailer detection                      │
│   • zone logic ("pallet crossed aisle-7 mouth → staging lane 3")        │
│   • rolling local footage buffer (~30 days)                             │
│   • clip extraction on event                                            │
└───────┬─────────────────────────────────────────────────────────────────┘
        │  structured events only (REST/webhook, TLS)
        ▼
┌──────────────────── DOCKFLOW CORE (multi-tenant SaaS) ─────────────────┐
│  VisionEvent ingestion → reconciliation vs. loads/tasks/appointments    │
│  Anomaly rules (per-tenant config) → alerts                             │
│  Claims bundles: event timeline + clips + load record                   │
│  Ground-truth feed into LoadEvent (source=vision), dwell & scorecards   │
└─────────────────────────────────────────────────────────────────────────┘
```

Hard rules:

1. **Raw video never leaves the site by default.** Only event metadata and on-demand clips cross the wire — bandwidth, privacy, and legal exposure all stay manageable.
2. **The SaaS ingests events, not streams.** Platform load is independent of camera count and resolution.
3. **Site degradation mode:** if the WAN link drops, the edge box buffers events and footage locally and backfills on reconnect. Detection never depends on cloud availability.
4. **Detection stack:** open detection models (YOLO-class) + zone/crossing logic built in-house. White-label vision vendors were considered and rejected — the platform integration is the differentiation; reselling a vendor's margin is not. (Revisit if time-to-pilot demands it.)

---

## K.5 Functional Requirements

Priority: **P0** = first paid deployment, **P1** = fast-follow.

### K.5.1 Detection & Movement Ledger · P0

**K-1.** As the **system**, I log every observed pallet/forklift/trailer zone transition with timestamp, zones, direction, and confidence, and attach a 10–30 s video clip.
- AC: clip retrievable from the event record in ≤ 3 clicks; clips uploaded on event or on demand.

**K-2.** As the **system**, I reconcile each movement against active system context (pick/move tasks, load assignments, appointments) and mark it **logged** or **unlogged**.
- AC: unlogged movements are queryable as a list, not just alerts — managers can triage.

**K-3.** As the **system**, I maintain **last-known-zone** per tracked pallet-position so "where is this pallet" is answerable from the ledger.
- AC: search by PO/reference (via load/task linkage) returns zone + last-seen timestamp + clip.

### K.5.2 Anomaly Detection (per-tenant rules) · P0

| # | Rule | Default threshold (per-tenant configurable) |
|---|---|---|
| K-A1 | **Unlogged movement** — pallet zone transition with no matching task | alert on every occurrence (triage list) |
| K-A2 | **Forgotten pallet** — pallet stationary in non-storage zone (staging, aisle corner, door apron) | > 4 h → alert; > 24 h → escalate to manager |
| K-A3 | **Departed but staged** — load marked Departed while linked pallets remain in staging | immediate alert |
| K-A4 | **Door count mismatch** — pallet count across dock threshold ≠ BOL/expected count on the load | flag before driver departure is confirmed |

- AC: every alert names the load/PO (if linked), zone, timestamp, and links the clip; alerts route per the tenant's notification preferences (Module I).

### K.5.3 Dock-Door Ground Truth · P0

**K-4.** Dock-door cameras verify load state transitions: trailer present at dock → `At Dock`; door open + pallet movement → `Loading/Unloading`; trailer departs → `Departed`.
- AC: `LoadEvent` gains `source: vision`; vision-confirmed and driver-tapped states coexist, divergences flagged.

**K-5.** Measured dwell (camera-verified arrival-to-departure) feeds carrier scorecards and smart matching (Module G) as ground truth, superseding self-reported timestamps where available.
- AC: scorecard marks each dwell datum as measured vs. reported.

### K.5.4 Claims & Disputes · P0

**K-6.** As a **warehouse manager**, I can generate a **claims bundle** per load: event timeline, pallet counts at the door, discrepancy flags, and video clips — exportable, and shareable with the carrier through the existing `OrgRelationship` per-load authorization (Module F-2).
- AC: carrier sees only the bundle for loads it is party to; bundle generation is audit-logged.

### K.5.5 P1 (designed for, not first deployment)

- License-plate capture at gate, auto-matched to appointment
- Zone-level inventory presence counts → cycle-count variance reports
- Forklift telematics correlation (which forklift/driver made an unlogged move — via operator badge at choke point, **not** face recognition)
- Carrier self-service claims portal
- Detention billing evidence (extends P1 detention tracking from base PRD)

---

## K.6 Data Model Additions

| Entity | Key fields (abridged) | Notes |
|---|---|---|
| `Site` (extends `Facility`) | facility_id, vision_kit_id, edge_box_id, install_date, survey JSON | One facility → one edge box in MVP |
| `Camera` | site_id, position_type (dock/aisle_end/staging/gate), zone coverage, stream URI, source (new/reused) | |
| `Zone` | site_id, type (rack/aisle/staging/dock_door/apron/yard), polygon config, is_storage flag | `is_storage=false` zones feed K-A2 |
| `VisionEvent` | site_id, camera_id, object_class, zone_from, zone_to, timestamp, confidence, clip_ref, linked_load_id, linked_task_id, logged/unlogged | The movement ledger |
| `Anomaly` | vision_event_id, rule (K-A1…K-A4), status (open/acked/resolved), assigned_to, resolution note | |
| `ClaimsBundle` | load_id, events JSON, clip_refs, generated_by, shared_with_org_id, created_at | Audit-logged on share |
| `LoadEvent` | **add** `source` enum value `vision`; **add** `verification` (vision_confirmed / self_reported / divergent) | Extends base PRD §10 |

---

## K.7 Privacy, Legal & Trust (NFR additions)

| Category | Requirement |
|---|---|
| **Video residency** | Raw footage stays on-site (rolling ~30-day buffer, per-tenant); only event metadata + extracted clips leave the site |
| **No biometric identification** | No face recognition. Person detection is for safety/occlusion logic only; operators are never identified by face (badge-based correlation is P1 and opt-in) |
| **Worker transparency** | Site signage + worker notification is part of the FDE install checklist; jurisdictions with works-council/notice requirements handled per deployment |
| **Clip access control** | Clips are tenant-scoped; cross-org sharing only via ClaimsBundle under per-load authorization; all access audit-logged |
| **Retention** | Events/ledger: 24 months (claims windows). Clips: 90 days hot, then archived or purged per tenant config. Raw footage: on-site rolling buffer only |

---

## K.8 Business Model

Given founder's openness to both structures, the recommendation is **setup + retainer**, because pure SaaS cannot absorb hardware reality:

| Component | Pricing shape | Notes |
|---|---|---|
| **One-time setup** | Standard Vision Kit (edge box + ≤12 cameras) + FDE install day(s) + zone labeling + tuning | Non-standard scope (extra cameras, odd mounting, long cable runs) quoted separately — PRD §4 scope discipline applies to hardware too |
| **Monthly retainer** | Per-site SaaS fee (platform, anomaly rules, claims bundles, model improvements); optionally tiered by camera count | Hardware amortization stays out of the subscription to keep payback sane |
| **Hardware ownership** | Client owns the kit (recommended) — avoids DockFlow carrying depreciating assets and simplifies exit terms | Alternative: DockFlow-owned kit leased into the retainer (evaluate after pilot #1) |
| **FDE attach** | Install + survey + first-month tuning is the paid FDE engagement — this module makes the hybrid SaaS+FDE model concrete | Reusable tuning learnings graduate into the standard kit runbook |

---

## K.9 FDE Site Survey Checklist (install prerequisite)

1. Floor plan + rack/aisle numbering scheme walk-through → zone map draft
2. Choke-point identification: dock doors, aisle ends, staging lanes, wrap machine, gate
3. Existing CCTV audit: IP-based? reachable? positioned at choke points? → reuse list
4. Lighting survey at each camera position (night shift matters — detection must work at 2 AM)
5. Network: PoE switch capacity, VLAN for cameras, WAN uplink for event stream
6. Power + mounting feasibility per position; lift rental needs
7. WMS/task-system integration surface (what events exist to reconcile against — CSV minimum, API preferred)
8. Worker-notification / signage requirements for the jurisdiction
9. Pilot scope sign-off: which doors/aisles go live in phase 1

---

## K.10 Rollout Phases

| Phase | Scope | Primary value proven | Exit criteria |
|---|---|---|---|
| **V0 — Dock doors** | Door cameras at one pilot warehouse; K-4/K-5 state verification + dwell truth; K-A4 count mismatch | Cheapest proof; claims evidence; feeds Module C/G data quality | Dwell measured vs. reported on 100% of loads at covered doors; ≥1 discrepancy caught before departure |
| **V1 — Movement ledger** | Aisle-end + staging choke points; K-1…K-3 ledger; K-A1/K-A2/K-A3 alerts | **The lost-pallet problem dies here** — last-known location + forgotten-pallet alerts | Time-to-locate a "lost" pallet < 5 min from ledger; forgotten-pallet alerts acknowledged same-shift ≥ 80% |
| **V2 — Inventory presence** | Zone-level presence counts; cycle-count variance reports | Manual count labor reduction | Pilot site reduces full cycle-count frequency or count hours measurably |
| **V3 — Claims productization** | K-6 bundles polished + carrier-facing sharing; detention evidence | Dispute/chargeback recovery as a sellable feature | ≥1 claim settled using a bundle; bundle shared through OrgRelationship in production |

**Sequencing note:** V0 before V1 despite lost pallets being the sharper pain — door coverage is the smallest possible hardware footprint, immediately improves the base product's data quality, and de-risks the install motion before touching aisle coverage. V1 follows immediately and is the headline ROI story for warehouse #2+.

**Pilot KPIs:** time-to-locate lost pallet (baseline: hours–days → target: minutes) · misplaced-pallet incidents detected/month · discrepancies caught before driver departure · claims recovery rate · cameras-per-site install time (drives setup-fee margin).

---

## K.11 Integration with Base PRD Roadmap

Module K is **not** inserted into M0–M5 (base MVP must ship first — it provides the Load/Appointment/task objects the ledger reconciles against). Earliest sensible start: **V0 in parallel with M5 (pilot hardening)**, at the same pilot warehouse, so camera-verified dwell immediately validates the arrival-warning KPIs.

Dependencies on base modules: Module F (shared load + LoadEvent), Module I (alert routing), Module J (tenancy, audit, feature flags — all K features ship behind per-tenant flags), Module G (ground-truth dwell feed).

---

*End of Vision Module Addendum v0.1. Next: validate choke-point count and kit cost against a real floor plan from the first pilot contact.*
