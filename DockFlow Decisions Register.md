# DockFlow — Decisions Register v0.1

> **Resolves:** PRD §14 open questions + pricing decisions made during Vision Module design
> **Date:** 2026-09-14 · **Owner:** Founder · **Status:** For founder ratification, then binding for M0–M5
> **Format:** each entry = question → recommendation → rationale → revisit trigger. "Ratify" = founder confirms; "Validate" = recommendation stands as default but must be confirmed with pilot data by the date/trigger noted.

---

## D1. Platform pricing model
*PRD §14.1 — per-warehouse subscription vs. per-load fee; where does FDE time price in?*

**Recommendation (ratify):** **Per-warehouse-org subscription; carrier side free.**
- Warehouses are the buyer with the pain (dock chaos, dwell, detention); carriers are the supply the warehouse pulls in (PRD §13: relationship model pulls supply in). Charging carriers adds friction exactly where adoption is most fragile.
- Per-load pricing penalizes the behavior we want (moving coordination onto the platform) and makes revenue unpredictable at low volume.
- **Draft tiers** (placeholder numbers — set after pilot #1's willingness-to-pay signal):
  - `Pilot` — 1 facility, unlimited carriers, core modules: `[ $500–800/mo ]`
  - `Growth` — up to 3 facilities, analytics, API access: `[ $1,500–2,500/mo ]`
  - `Enterprise` — multi-facility, SSO, custom flags, dedicated FDE hours: quoted
- **FDE time prices separately**, always: setup engagements and custom work are quoted per statement of work, never bundled into subscription. This is the margin firewall between "product company" and "consultancy" (PRD §13).

**Revisit trigger:** pilot #1 pricing conversation; any prospect who balks at subscription but engages on per-load.

---

## D2. Vision module pricing
*Settled during Vision Module design (Addendum K.8) — recorded here so all commercial decisions live in one register.*

**Decided:** **One-time setup fee + monthly per-site retainer; client owns the hardware.**
- Setup: standard Vision Kit (edge box + ≤12 cameras + install + zone labeling + tuning): `[ $15,000 ]` draft, per ROI one-pager.
- Retainer: `[ $1,800/mo ]` per site draft; tier by camera count if kit standard changes.
- Client-owned hardware avoids DockFlow carrying depreciating assets; alternative (DockFlow-owned, leased into retainer) is an evaluated fallback if pilot #1 strongly prefers OpEx.
- Non-standard scope (extra cameras, odd mounting) quoted separately — §4 scope discipline applies to hardware.

**Revisit trigger:** first real BOM from a site survey (K.9); pilot #1 procurement feedback.

---

## D3. Who pays for SMS
*PRD §14.2*

**Recommendation (ratify):** **Absorbed in subscription with a fair-use allowance; overage passed through at cost + 20%.**
- At MVP volumes SMS cost per warehouse is small and predictable; charging per message creates bill-shock conversations and discourages the driver SMS channel that is our adoption wedge.
- Fair-use allowance: `[ 2,000 messages/mo ]` per warehouse org (≈65/day — well above pilot needs); overage billed transparently.
- Safety valves already built: per-tenant budget alerting (M2-E3), delivery-rate monitoring (M2-E2). Abuse or anomalous volume (e.g. a tenant running marketing blasts) → overage clause, not an argument.

**Revisit trigger:** any tenant exceeding allowance two months running; SMS provider price changes.

---

## D4. Integration targets (WMS/TMS)
*PRD §14.3 — which systems do pilot clients run? Determines API/webhook and first FDE engagement priorities.*

**Action item (validate):** **Survey before sprint 0 ends.** Ask pilot #1 (and the 2–3 friendliest carriers) directly: WMS name/version, how they export data today, who administers it.
- **Default posture until known:** CSV-first reconciliation is guaranteed (vision reconciler accepts CSV-imported task context, M4-D4); live API adapters built only for systems a real client runs, in order of client count.
- The first FDE engagement target is likely "WMS → DockFlow task/context feed" for the pilot warehouse — this is what makes vision anomaly detection precise (fewer false "unlogged" flags).

**Revisit trigger:** survey answers received; second pilot signing.

---

## D5. Multi-facility warehouses
*PRD §14.4 — MVP assumes single-site per org.*

**Recommendation (ratify):** **Single-facility UI for MVP; schema stays multi-facility.**
- `facilities` is already a separate table (Spec §2.2) — nothing to undo later. The UI simply scopes to the org's default facility.
- Adding a facility switcher is a P1 UI feature, not a migration. Do not let a prospect's "we have 4 sites" expand MVP scope; pilot site #1 is one building.

**Revisit trigger:** a signed prospect with a hard multi-site requirement (then prioritize the facility switcher in P1).

---

## D6. Rate visibility in tenders & matching
*PRD §14.5 — do tenders include rates; is historical rate a matching factor (sensitive)?*

**Recommendation (ratify):** **Rate is an optional per-load field, warehouse-controlled; excluded from matching factors in MVP.**
- `details.rate_cents` exists in the schema (Spec §2.3) but renders only if the warehouse org enables it per load — some warehouses share rates, many won't.
- Matching MVP factors: lane history, on-time %, dwell, equipment fit, recency (G-1). Rate joins the factor set only behind a per-tenant flag after pilot feedback, because surfacing rate comparisons can poison carrier relationships if handled clumsily.

**Revisit trigger:** pilot managers explicitly asking "who was cheapest on this lane?" — then ship the flagged factor.

---

## D7. Detention tracking
*PRD §14.6 — pilot sites will ask immediately; confirm P1.*

**Recommendation (ratify):** **Stays P1 — but it's a reporting feature, not a data problem.**
- Every timestamp detention math needs is already captured: arrival, at-dock, loading start, departure (`load_events`, increasingly vision-verified from V0). P1 work = free-period config per carrier relationship + detention report + export.
- When a pilot asks (they will), the honest answer is: "Your dwell data is being captured today; the detention report ships in the fast-follow, and your history will already be there." This is a sales asset, not a gap.
- Vision module strengthens it further: camera-verified dwell is dispute-resistant detention evidence (Addendum K.5.5).

**Revisit trigger:** any pilot making detention a signing condition → pull the report forward; the data layer needs nothing.

---

## D8. Spanish-language driver SMS
*PRD §14.7 — validate demand; i18n plumbing is in NFRs either way.*

**Recommendation (validate):** **English-only content at MVP; i18n plumbing built from day one; Spanish templates are a P1 fast-follow gated on one question asked during pilot carrier onboarding: "what language do your drivers prefer for texts?"**
- Message templates externalized from day one (no hardcoded strings in the notifications worker) so adding Spanish is content work, not engineering.
- Do not ship machine-translated SMS unreviewed — a wrong instruction texted to a driver at 60 mph is a liability. Spanish templates get a native-speaker review before enabling.

**Revisit trigger:** pilot carrier onboarding answer; >20% of pilot drivers preferring Spanish pulls it into P1 immediately.

---

## Ratification checklist for the founder

| # | Decision | Status needed |
|---|---|---|
| D1 | Platform pricing (warehouse pays, carriers free, FDE separate) | Ratify |
| D2 | Vision pricing (setup + retainer, client owns kit) | Already decided — confirm |
| D3 | SMS absorbed with fair-use allowance | Ratify |
| D4 | WMS/TMS survey before sprint 0 ends | Schedule the conversation |
| D5 | Single-facility UI, multi-facility schema | Ratify |
| D6 | Rate optional, excluded from matching at MVP | Ratify |
| D7 | Detention stays P1, data already capturing | Ratify |
| D8 | English at MVP, Spanish gated on pilot demand | Validate at pilot onboarding |

*Once ratified, this register feeds back into the PRD (§14 resolved) and the ROI one-pager's placeholder pricing. Changes after ratification require the same explicit sign-off as PRD scope changes (§13 FDE-discipline rule).*

---

*End of Decisions Register v0.1. Next: pilot playbook — the site survey, onboarding sequence, and first-90-days success plan for warehouse #1.*
