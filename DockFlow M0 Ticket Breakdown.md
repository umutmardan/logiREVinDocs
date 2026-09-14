# DockFlow — M0 "Foundations" Sprint-Ready Ticket Breakdown v0.1

> **Companion to:** PRD §12 (M0), Technical Architecture v0.1, Data Model & API Spec v0.1
> **Date:** 2026-09-14 · **Milestone window:** weeks 1–3 · **Team assumption:** 2–4 engineers
> **M0 exit criteria (PRD):** *Two portals, separate logins, empty shells, orgs can link.*

---

## 0. How to read this

- **Estimates** are ideal engineer-days (1d ≈ one focused day). Ranges reflect unknowns; engineers re-estimate in sprint planning — these are starting points, not commitments.
- **AC** = acceptance criteria. A ticket is done when every AC passes in staging.
- **Depends on** lists blocking tickets. Tickets with no dependency can start day one.
- **[Decision]** flags tickets that need a human choice before work starts (Architecture §11 open questions).
- Definition of Done for all M0 tickets: code reviewed, tests green in CI, deployed to staging, no secrets in repo, audit/event hooks wired where applicable.

**M0 totals:** ~50–65 engineer-days across the milestone. With 3 engineers this is a tight-but-feasible 3 weeks; with 2, cut the stretch tickets marked *(stretch)* or extend a week.

---

## 1. Sprint 0 — Decisions & Environment (before day 1, founder + lead)

| ID | Ticket | Est | AC |
|---|---|---|---|
| S0-1 | **[Decision] Cloud provider + region** — pick per Architecture §11.1; region near pilot sites | 0.5d | Decision recorded in architecture doc; Terraform bootstrapped against the account |
| S0-2 | **[Decision] Auth provider** — must support two fully separate user pools + custom token service hooks for driver magic links | 0.5d | Decision recorded; two pools provisioned (`dockflow-wh`, `dockflow-car`) |
| S0-3 | **[Decision] SMS providers (primary + fallback)** — **start 10DLC/toll-free registration immediately; it has multi-week lead time and blocks M2** | 0.5d | Registrations submitted; provider accounts created; decision recorded |
| S0-4 | Repo + engineering hygiene — monorepo (core API, both portals, shared packages, Terraform); branch protection on `main`; conventional commits; PR template | 0.5d | Empty monorepo with protection rules; `main` deployable (empty hello-world) |

---

## 2. Epic M0-A — Infrastructure & CI/CD

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-A1 | Terraform: base environments `staging` + `prod` — managed Postgres, Redis, container runtime, secrets manager, object storage (clip bucket, private) | S0-1 | 2–3d | `terraform apply` creates both envs from scratch; state in remote backend; teardown works |
| M0-A2 | CI pipeline — PR → lint, typecheck, unit tests → build containers → deploy staging | S0-4 | 1.5d | Green PR auto-deploys staging; failing tests block merge |
| M0-A3 | CD promotion — staging → prod as gated manual step; migration step runs gated (Architecture §9.2) | M0-A1, M0-A2 | 1d | Promote button/gate works; migration failure blocks deploy and alerts |
| M0-A4 | Observability baseline — OTel instrumentation in core API skeleton; structured logs with correlation IDs; hosted backend connected; uptime check on `/health` | M0-A1 | 1.5d | Trace visible end-to-end on a test request; uptime alert pages a human |
| M0-A5 | Secrets & config — secrets manager wired into runtime; per-env config; no env files in repo (Architecture §8) | M0-A1 | 1d | Rotate a secret without deploy; repo scan finds zero secrets |

## 3. Epic M0-B — Database Foundations

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-B1 | Migration framework + first migration: `organizations`, `org_relationships`, `users` (Spec §2.1) incl. enum types | M0-A1 | 1d | Migrations run forward/back clean in CI; enums match spec |
| M0-B2 | **RLS tenancy enforcement** — `app.current_org` request context; RLS policies on all tenant tables; connection middleware sets context per request | M0-B1 | 2d | Cross-org SELECT returns zero rows even with hand-written SQL; context unset → query fails closed |
| M0-B3 | **Outbox + event log** — `domain_events` outbox table; transactional write helper (`withEvent()`); publisher worker → Redis pub/sub; envelope per Spec §4 | M0-B1 | 2d | Event written in same tx as state change; crash-after-commit still publishes (test: kill worker mid-flight); consumers dedupe by event id |
| M0-B4 | Append-only `audit_log` (partitioned) + write helper (Spec §2.7) | M0-B1 | 1d | Helper callable from any module; UPDATE/DELETE on the table rejected by DB role permissions |
| M0-B5 | Remaining M0 tables: `facilities`, `docks` (Spec §2.2) — no UI yet, API-ready | M0-B2 | 1d | Migrations green; RLS policies applied |

## 4. Epic M0-C — Identity & Dual Auth (PRD §9.3)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-C1 | Warehouse auth integration — pool `dockflow-wh`; login/session flow; JWT validation middleware asserting audience `wh` | S0-2 | 2d | Warehouse user logs in; carrier-pool token is rejected with 401 |
| M0-C2 | Carrier auth integration — pool `dockflow-car`, audience `car` | S0-2 | 1.5d | Mirror of C1; warehouse-pool token rejected |
| M0-C3 | `users` ↔ identity sync — first login provisions local `users` row (org, role from invite); deactivation blocks login | M0-B1, M0-C1 | 1.5d | Deactivated user token rejected within 60 s |
| M0-C4 | RBAC guard — role claims → permission middleware enforcing PRD §9 matrix per endpoint; `403` vs `404` policy (cross-tenant resources → 404) | M0-C1 | 1.5d | Matrix covered by automated tests (Spec §7) |
| M0-C5 | **Authorization matrix test suite skeleton** — harness that attacks every endpoint × every role × cross-tenant fixtures | M0-C4 | 2d | Suite runs in CI; intentionally-broken fixture endpoint fails the suite |
| — | *(stretch)* Driver magic-link token service skeleton — mint/validate single-trip tokens, hashed at rest | M0-C4 | 1.5d | Token validates against trip scope; expired token rejected. *(Full flow lands in M2 with SMS.)* |

## 5. Epic M0-D — Organizations, Invites & Relationships

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-D1 | Org onboarding API — create warehouse/carrier org + first admin (founder-operated at MVP; no self-serve signup) | M0-C1, M0-C2 | 1.5d | Org + admin created via API; audit event written |
| M0-D2 | User invites — admin invites user with role; invite email; accept flow lands in correct pool | M0-C3 | 1.5d | Invited user lands in correct portal; role matches invite; audit event |
| M0-D3 | User management — list, change role, deactivate (J-2) | M0-D2 | 1d | Last-admin protection (can't deactivate sole admin); audit events |
| M0-D4 | **Org relationship link** — invite/accept/decline/revoke (J-1): warehouse invites carrier by email → carrier admin accepts → `active` | M0-D1 | 2d | Only `active` relationships appear in carrier directory; state transitions audited; duplicate invite rejected |
| M0-D5 | Facility bootstrap — creating a warehouse org creates its default facility + sample dock config | M0-B5, M0-D1 | 1d | New warehouse org has facility with editable geofence radius + operating hours |

## 6. Epic M0-E — Feature Flags & Tenant Config (FDE foundation, PRD §4)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-E1 | Flag service — `organizations.feature_flags` read path; server-side guard helper (`requireFlag('vision_module')`); defaults-off for unknown flags | M0-B1 | 1d | Flagged endpoint 404s when flag off; flag flip takes effect ≤30 s without deploy |
| M0-E2 | Tenant settings API — `GET/PATCH /settings` for facility/geofence/thresholds (Spec §5.1); JSON-schema-validated settings writes | M0-D5 | 1.5d | Invalid settings rejected with 422 + schema errors; changes audit-logged |

## 7. Epic M0-F — Portal Shells (both Next.js apps)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-F1 | Warehouse portal shell — app scaffold, auth wiring, layout, empty nav (Calendar / Dock Board / Carriers / Settings), shared design-system package initialized | M0-C1 | 2d | Login → authenticated empty shell; p95 < 2 s on staging |
| M0-F2 | Carrier portal shell — mirror of F1 (Tenders / Trips / Fleet / Settings) | M0-C2 | 1.5d | Same; separate deployment, separate domain |
| M0-F3 | Settings screens — org profile, users & invites, facility & geofence, feature-flag visibility (read-only for non-admins) | M0-D2, M0-D5, M0-E2 | 2–3d | Admin can complete D2–D5 flows end-to-end in UI |
| M0-F4 | Relationship UI — warehouse: invite carrier + directory with status; carrier: accept/decline incoming invite | M0-D4 | 1.5d | Full link flow completable from both portals without touching an API client |
| — | *(stretch)* Shared component package hardening — table, form, toast, empty-state primitives documented with examples | M0-F1 | 1d | Both portals consume same package |

## 8. Epic M0-G — Notifications Skeleton (enough for M0 invites; full catalog in M2)

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-G1 | Notifications worker + `notifications` table (Spec §2.6) — consumes `notification.requested`, email channel only in M0 | M0-B3 | 1.5d | Invite email delivered via queue, not request path; delivery status recorded |
| M0-G2 | Provider abstraction — email behind internal interface; SMS interface defined + stub provider (real SMS in M2 after 10DLC clears) | M0-G1 | 1d | Swap email provider via config; SMS stub logs would-be sends |

## 9. Quality Gate & Milestone Exit

| ID | Ticket | Depends on | Est | AC |
|---|---|---|---|---|
| M0-Q1 | Load smoke test — k6-class script: 200 concurrent authed sessions across both portals hitting M0 endpoints | all above | 1d | p95 < 2 s under smoke load; no 5xx |
| M0-Q2 | Security pass — dependency scan gate in CI; RLS pen-check script (attempt cross-tenant reads on every table); OWASP spot-check on auth flows | M0-C5, M0-B2 | 1d | Scan gate blocks on high-severity; pen-check green |

**M0 exit checklist (maps to PRD exit criteria):**

- [ ] Warehouse portal and Carrier Portal each have independent logins; tokens never cross (C1–C2)
- [ ] Warehouse org can invite a carrier org; carrier accepts; relationship `active` (D4, F4)
- [ ] RBAC matrix enforced and tested (C4–C5)
- [ ] Audit log records org/user/relationship/flag changes (B4)
- [ ] `main` always deployable; staging→prod promotion works (A2–A3)
- [ ] Outbox publishes domain events reliably (B3) — foundation for every later module
- [ ] 10DLC registration **in flight** (S0-3) — verified, since it gates M2

---

## 10. Carry-over / explicitly NOT in M0

Driver SMS + magic links (M2), tenders (M2), calendar UI (M1), dock board realtime (M1), anything vision-related beyond schema placeholders (M4/V0). If M0 work drifts toward these, it goes back to the backlog — milestone discipline is the FDE-scope-creep defense (PRD §13).

**Biggest M0 risks:** (1) auth provider integration consuming more time than estimated — mitigate by choosing the provider with the best Next.js SDK in S0-2; (2) 10DLC lead time — mitigated only by starting in S0-3; (3) RLS subtlety — B2 and Q2 exist precisely because silent tenant leakage is the worst outcome this milestone can ship.

---

*End of M0 breakdown v0.1. Next: M1 (Scheduling) ticket breakdown, or founder review of [Decision] tickets with the engineering team.*
