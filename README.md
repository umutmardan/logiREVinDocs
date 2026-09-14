# DockFlow — Master Index

> **Last updated:** 2026-09-14
> **What this is:** the map of every document in this workspace — what each is for, who should read it, and in what order. Share this file first when bringing someone new into the project.

**DockFlow in one paragraph:** a two-sided logistics coordination platform connecting warehouses and carriers — appointment scheduling, dock assignment, arrival warnings, AI carrier matching and ETA prediction — extended by a camera-based **Vision Module** that turns warehouse choke points into a physical ground-truth layer: every pallet movement logged, forgotten pallets flagged same-shift, and every load carrying dispute-proof evidence. Sold as SaaS (warehouses pay, carriers free) plus FDE-deployed vision kits (setup fee + monthly retainer).

---

## 1. Document Map

### Strategy & Product — *what we're building and why*

| Document | Contents | Primary audience |
|---|---|---|
| [DockFlow MVP PRD.md](./DockFlow%20MVP%20PRD.md) | The founding document: problem, personas, dual-portal model, modules A–J, RBAC matrix, data model sketch, NFRs, M0–M5 roadmap, risks, glossary | Everyone — read first |
| [DockFlow Vision Module Addendum.md](./DockFlow%20Vision%20Module%20Addendum.md) | Module K: movement-ledger design, choke-point camera topology, edge architecture, anomaly rules, claims bundles, privacy posture, V0–V3 phases | Everyone — read second |
| [DockFlow Decisions Register.md](./DockFlow%20Decisions%20Register.md) | All commercial/product decisions (pricing, SMS, rates, detention, Spanish SMS, multi-facility) with rationale and revisit triggers — resolves PRD §14 | Founder (ratify), then all |

### Commercial — *selling and running the business*

| Document | Contents | Primary audience |
|---|---|---|
| [DockFlow Vision Pilot ROI One-Pager.md](./DockFlow%20Vision%20Pilot%20ROI%20One-Pager.md) | Customer-facing pitch: the problem in their dollars, pricing, payback math, pilot offer. Bracketed placeholders = fill with prospect's real numbers | Prospects; founder customizes per deal |
| [DockFlow Pilot Playbook.md](./DockFlow%20Pilot%20Playbook.md) | Running pilot #1: selection criteria, discovery agenda, site survey checklist, pilot agreement terms, week-by-week onboarding, vision install week, 90-day KPI plan, FDE runbook | Founder + FDEs |

### Engineering — *how it's built*

| Document | Contents | Primary audience |
|---|---|---|
| [DockFlow Technical Architecture.md](./DockFlow%20Technical%20Architecture.md) | Five architectural principles, stack recommendations with exit doors, system topology, event spine, edge architecture, failure-mode table, security, build-vs-buy register | Engineers, technical advisors |
| [DockFlow Data Model and API Spec.md](./DockFlow%20Data%20Model%20and%20API%20Spec.md) | Postgres DDL for all 20+ tables, state transition table, domain event catalog, full REST API per portal, driver magic-link API, edge ingestion API, webhook contracts, test hooks | Engineers building M0+ |

### Execution — *sprint-ready work*

| Document | Contents | Window |
|---|---|---|
| [DockFlow M0 Ticket Breakdown.md](./DockFlow%20M0%20Ticket%20Breakdown.md) | Foundations: tenancy, dual auth, org relationships, RBAC, outbox, CI/CD. ~50–65 eng-days | wk 1–3 |
| [DockFlow M1 Ticket Breakdown.md](./DockFlow%20M1%20Ticket%20Breakdown.md) | Scheduling: appointments, dock calendar, assignment engine, live dock board, staging queue. ~55–70 eng-days | wk 4–7 |
| [DockFlow M2 Ticket Breakdown.md](./DockFlow%20M2%20Ticket%20Breakdown.md) | Carrier side: tenders, dispatch, SMS magic links, driver mobile experience. ~45–55 eng-days | wk 8–10 |
| [DockFlow M3 Ticket Breakdown.md](./DockFlow%20M3%20Ticket%20Breakdown.md) | Arrival warnings: rules-based ETA, geofences, 60/30/10 alerts, escalation. ~28–34 eng-days | wk 11–12 |
| [DockFlow M4 Ticket Breakdown.md](./DockFlow%20M4%20Ticket%20Breakdown.md) | AI features + vision foundation: matching with approval gate, ETA shadow→live, camera simulator, ingestion, reconciler, anomaly rules. ~52–62 eng-days | wk 13–16 |

*M5 (pilot hardening) and V0 (physical vision kit deployment) remain unticketed until pilot dates are real — see the Pilot Playbook for their operational shape.*

---

## 2. Reading Orders by Role

**New engineer partner (half day):**
1. PRD §1–§7 (product shape) → 2. Technical Architecture (whole) → 3. Data Model & API Spec §1–§3 → 4. M0 breakdown → skim M1

**Technical advisor / investor (one hour):**
1. This index's paragraph above → 2. PRD §1–§4 (summary, problem, model) → 3. Vision Addendum §K.1–K.3 (the second act) → 4. Architecture §1–§2 (how we think) → 5. Decisions Register (commercials)

**FDE preparing for pilot:**
1. Pilot Playbook (whole) → 2. ROI One-Pager → 3. Vision Addendum K.9 (site survey) → 4. Decisions Register (what's negotiable, what isn't)

**Prospect (external):**
1. ROI One-Pager only — customized with their numbers. Nothing else leaves the building without founder review (PRDs contain strategy, pricing logic, and risk framing).

---

## 3. Status & Change Discipline

| Document | Version | Status |
|---|---|---|
| MVP PRD | 0.1 | Draft — §14 resolved by Decisions Register (fold back on next revision) |
| Vision Addendum | 0.1 | Draft |
| ROI One-Pager | draft | ⚠️ Internal until founder confirms pricing + guarantee terms; placeholders must be replaced per prospect |
| Technical Architecture | 0.1 | Draft for engineering review — stack choices need engineer ratification in sprint 0 |
| Data Model & API Spec | 0.1 | Draft — ticketable as-is |
| M0–M4 Breakdowns | 0.1 | Estimates are starting points; teams re-estimate in sprint planning |
| Decisions Register | 0.1 | **Awaiting founder ratification** (checklist at end of doc) |
| Pilot Playbook | 0.1 | Draft — bracketed terms to finalize before first offer |

**Rules:** scope changes follow PRD §13 (explicit sign-off). Decision changes update the Decisions Register first, then ripple to affected docs. Every doc lists its companions in the header — keep those links honest when revising.

---

## 4. The Critical Path Right Now

1. **Founder ratifies the Decisions Register** (30 minutes, checklist provided)
2. **WMS/TMS survey conversation with pilot #1 contact** (D4 — shapes the first FDE engagement)
3. **Sprint-0 decisions with engineers:** cloud provider, auth provider, SMS providers — and **submit 10DLC registration immediately** (multi-week lead time, blocks M2)
4. **Engineering re-estimates M0** → build begins

---

*End of Master Index. Update this file whenever a document is added, renamed, or materially revised.*
