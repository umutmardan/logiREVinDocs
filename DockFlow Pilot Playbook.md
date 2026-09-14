# DockFlow — Pilot Playbook v0.1

> **Companion to:** PRD §12 (pilot strategy), ROI One-Pager, Vision Addendum K.9, Decisions Register
> **Date:** 2026-09-14 · **Owner:** Founder · **Audience:** Founder + FDEs running pilot #1
> **Purpose:** the document you carry into the first customer relationship — from discovery call to referenceable case study.

---

## 1. What a pilot is for (and what it is not)

A pilot exists to produce three things:

1. **Proof** — measured KPI improvement against a captured baseline (PRD §3.2): dwell time, arrival-warning coverage, dock conflicts, and (with the vision kit) time-to-locate and discrepancy catches.
2. **A reference** — one warehouse manager who will take a call from the next prospect; one quantified case study (PRD BG-4).
3. **Product truth** — FDE observations that set the fast-follow list. Every "huh, that's odd" on-site is backlog gold.

A pilot is **not** a discount funnel or a custom-development engagement. The Decisions Register (D1) applies: FDE work is quoted separately, and scope discipline (PRD §4) is in force from day one — the pilot is where scope creep tries hardest to happen.

---

## 2. Pilot selection criteria

Choose warehouse #1 against these filters, in order:

| Criterion | Why it matters | Red flag |
|---|---|---|
| Founder relationship exists | Fast trust, honest feedback, forgiveness for MVP edges | Cold outreach as pilot #1 |
| Single facility, 4–12 dock doors | Matches MVP assumptions (D5) and standard vision kit | Multi-site mandate on day one |
| Real dock pain they can articulate | Baseline exists to improve; they feel the problem | "We're already pretty smooth" |
| 2–3 friendly carriers who'll join | Two-sided platform needs both sides live (PRD §12) | Adversarial carrier relationships |
| WMS data exportable (CSV minimum) | Cold-start matching (G-4) + vision reconciliation context | Locked-down WMS with no exports |
| A named internal champion | Someone whose job gets easier; pushes adoption | "Just talk to IT" |
| Existing IP cameras (bonus) | Potential hardware reuse (K.9) lowers setup cost | Not a blocker either way |

---

## 3. Phase 0 — Discovery & Site Survey (weeks −3 to 0, before signing)

### 3.1 Discovery call agenda (60 min)

1. Their dock day: volume, peaks, how scheduling happens now (phone/email/spreadsheet — get specifics)
2. Pain inventory: dwell/detention costs, lost-pallet stories, claim disputes, cycle-count labor — **capture their numbers for the ROI one-pager placeholders**
3. WMS/TMS survey (D4): system name/version, export capability, who administers it
4. Carrier map: their top 5 carriers, relationships, would they invite them
5. Vision appetite: shrinkage history, existing camera system (make/model/IP-based?), camera policy/union considerations
6. Decision process: who signs, procurement, timeline

### 3.2 Site walk (45–90 min, the K.9 checklist executed)

- [ ] Floor plan + rack/aisle numbering walk → zone map draft
- [ ] Choke points identified: dock doors, aisle ends, staging lanes, wrap machine, gate
- [ ] Existing CCTV audit: IP-based? reachable? positioned usefully? → reuse list
- [ ] Lighting at each camera position, **including night-shift lighting**
- [ ] Network: PoE capacity, VLAN, WAN uplink
- [ ] Power + mounting feasibility; lift rental needs
- [ ] Worker-notification/signage requirements for the jurisdiction (K.7)
- [ ] Wi‑Fi/cellular dead zones that affect driver mobile experience
- [ ] Pilot scope sign-off: which doors/aisles go live first

### 3.3 Exit of Phase 0
ROI one-pager with **their** numbers replacing every bracket + a scoped setup quote + pilot agreement (§4). No vague verbal pilot — the agreement defines success criteria in writing.

---

## 4. Phase 1 — Pilot Agreement (signing)

Per the ROI one-pager pilot offer + Decisions Register:

| Term | Standard pilot position |
|---|---|
| Platform | `Pilot` tier pricing, first `[ 3 ]` months free, then `[ $X/mo ]` |
| Vision kit (if in scope) | `[ 25% ]` off setup fee; client owns hardware (D2) |
| Success criteria | Written, measurable, agreed: e.g. "≥80% of arrivals with ≥30-min warning by week 8; first forgotten-pallet catch within 60 days of vision go-live" |
| Exit clause | The honest exit from the one-pager: no value in 60 days → kit removed, setup refunded |
| Their commitment | Named champion, CSV historical export in week 1, 2–3 carriers invited in weeks 2–3, FDE access during install week |
| Case study | Right to publish anonymized metrics; named case study negotiable for additional discount |

---

## 5. Phase 2 — Platform Onboarding (weeks 1–3, software first)

Software value lands **before** any camera is mounted. Sequence:

| Week | Actions | Owner | Done when |
|---|---|---|---|
| **1** | Org + facility created; users invited (admin, manager, pickers, workers); docks configured (labels, constraints, hours, geofence radius); CSV historical loads/carriers imported (G-4); WMS export cadence agreed (D4) | FDE + champion | Manager schedules real appointments in DockFlow; matching shows real recommendations from imported history |
| **2** | Carrier invites sent to 2–3 friendliest carriers; carrier onboarding calls (30 min each — dispatcher sets up fleet/drivers); first real tenders flow | Champion + FDE | ≥1 carrier accepts a tender through the platform; first driver SMS delivered to a real phone |
| **3** | Dock board mounted at the dock (cheap tablet/TV); alert thresholds tuned per site (M3-F3 — this is the fatigue-sensitive step); pickers working from staging queue; weekly check-in cadence set | FDE on-site | Crew uses the board unaided for a full day; ≥70% of appointments created on-platform (BG-2 trajectory) |

**Training philosophy:** 20-minute role-based sessions, on the floor, on their data — no slide decks. Drivers need nothing but the SMS (that's the design); verify it with a real driver in week 2.

---

## 6. Phase 3 — Vision Install Week (V0, weeks 4–5 if in scope)

Prerequisites: M4 software complete (ingestion/reconciler/anomaly rules proven against simulator), BOM finalized, hardware ordered with 2-week lead.

| Day | Plan |
|---|---|
| **Mon** | FDE + installer on-site; mount door cameras (V0 scope: dock doors first, per Addendum K.10); edge box racked, network up, mTLS provisioned |
| **Tue** | Zone labeling in portal with warehouse manager (they know the floor); detection validation per camera — walk pallets, verify crossings; night-shift lighting check |
| **Wed** | Reconciliation tuning: WMS task feed live; unlogged-move false-positive review with manager (first calibration of trust) |
| **Thu** | Anomaly rules enabled in **observe mode** (alerts to FDE only); thresholds tuned; door-count verification against a live load |
| **Fri** | Alerts enabled to manager + crew; staff walkthrough (what an alert means, how to triage); signage confirmed; handover doc signed |

**Aisle/staging cameras (V1)** follow 2–4 weeks later after V0 trust is established — the movement-ledger pitch lands much better when the manager has already seen a door discrepancy caught.

---

## 7. First 90 Days — Success Management

### Cadence

| Rhythm | Content |
|---|---|
| Weeks 1–4: twice-weekly 15-min check-in | Adoption blockers, alert tuning, quick wins surfaced |
| Weeks 5–12: weekly 30-min | KPI review vs baseline, anomaly triage quality, carrier engagement |
| Day 90: exec review | The case-study meeting — numbers below, reference ask, expansion conversation (V1 cameras, more doors, growth tier) |

### KPI baseline → target (capture baseline in week 1 — before we change anything)

| KPI (PRD §3.2) | Baseline capture method | Target |
|---|---|---|
| Avg driver dwell | Their records + our measured timestamps from week 2 | −20% |
| Arrivals with ≥30-min warning | `notifications` + `load_events` report | ≥80% |
| Double-booked dock-hours | Calendar conflict log | <2% of slots |
| Tender acceptance rate | Tender events | ≥60% |
| Manager approval rate of recommendations | `carrier_recommendations.decision` | ≥50% |
| ETA accuracy (±15 min @ 1h out) | Shadow dashboard (M4-C2) | ≥70% before model goes live |
| SMS delivery rate | Provider webhooks | ≥98% |
| *(Vision)* time-to-locate a lost pallet | Timed lookup drill + real incidents | minutes, not hours |
| *(Vision)* discrepancies caught before departure | Anomaly log | first catch within 60 days |

### Failure protocol
If a KPI trend is flat at day 45: diagnose honestly (adoption? tuning? product gap?), fix what's ours, say plainly what isn't. A pilot that fails honestly teaches more than one that lingers — and the exit clause protects the relationship for the next product.

---

## 8. FDE Runbook Essentials

| Topic | Rule |
|---|---|
| On-site presence | Install week + week-1 onboarding on-site; after that remote with scheduled visits |
| Config over code | Every client-specific need goes to tenant config/flags first; code only with founder sign-off (PRD §4) |
| Observation log | Every site visit produces a written log: surprises, workarounds staff invented, requests. Weekly review → fast-follow backlog |
| Three-client rule | A customization three clients want → core feature candidate; one client → stays flagged (PRD §4) |
| Escalation | Product bug → engineering ticket same day; safety/tenant-data issue → immediate, founder notified |
| Hardware spares | One spare camera + one spare edge box in the kit inventory per region; swap-and-reprovision, never debug-on-ladder |

---

## 9. What "won" looks like at day 90

- Warehouse runs its dock day on DockFlow without the champion pushing
- 2–3 carriers tender and dispatch through the platform as normal habit
- The KPI table above has their real numbers in both columns
- One story worth telling: "the system flagged a pallet sitting behind door 6 for 26 hours — it was the missing Hartline order" *(real version TBD by reality)*
- Manager agrees to be a reference; case study drafted with their sign-off
- Expansion order on the table: more doors, V1 aisle cameras, or growth tier

---

*End of Pilot Playbook v0.1. The document set is now complete for pilot #1: ROI one-pager opens the conversation, this playbook runs it, and the decisions register keeps the commercials straight.*
