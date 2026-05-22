---
status: approved
created: 2026-05-22
version: 1.0
---

# MVI- en SROI-eisen in Aanbestedingen — Proposal

## Executive Summary

Enable purchasers to systematically integrate Maatschappelijk Verantwoord Inkopen (MVI — Socially Responsible Procurement) and Social Return on Investment (SROI) requirements into every tendering process, supported by a centralized criteria library, automated obligation calculation, leveraged SROI tracking, and portfolio-level reporting.

This consolidates fragmented MVI implementation (currently scattered across template fragments and manual Excel trackers) into a unified capability that enforces compliance with the MVOI 2026 Manifest, the AanbestedingsWet 2012 (art. 1.4, 2.114), and enables proof of progress toward critical portfoliodoals (CO2 -49% by 2030, Circular 50% by 2030).

---

## Business Context

### The Mandate

- **MVOI 2026 Manifest**: All decentral authorities + the Realm must demonstrate structured MVI governance.
- **AanbestedingsWet 2012 art. 1.4**: Environmental and social aspects are mandatory, not optional.
- **SROI Floor**: Minimum 2–5% of order value (depending on region, category, order type) must be channeled to SROI since 2011; **Bouwblokkenmethode** (uniform Dutch SROI methodology, TNO/PSO-managed since 2018) is the standard.
- **Portfolio Goals**: CO₂ emissions -49% by 2030, Circular spend 50% by 2030 (Nationaal Programma Circulaire Economie 2023–2030).
- **Accountability**: College/Raad, MVO Annual Report, RIVM annual census (MVOI-monitor).

### Why Now

- RVO MVI-criteriatool is production-ready and versioned quarterly.
- CO₂-Prestatieladder, PSO-sociaal indicators, and Cradle-to-Cradle certifications are standard procurement input.
- Existing practices are inefficient: MVI requirements are hand-crafted per tender, SROI tracking is spreadsheet-based, and there is no single source of truth for portfolio progress.
- Dutch purchasers are facing audit pressure and reputational risk if MVOI commitments cannot be proven.

---

## Scope & Features

### 1. MVI Criteria Library
- **Source**: RVO MVI-criteriatool (quarterly import), CO₂-Prestatieladder, PSO-register, organization-owned criteria.
- **Scope**: 8 categories (klimaat_co2, milieu_overig, social_arbeid, social_inclusie, circulair, biodiversiteit, dierenwelzijn, internationaal_iso26000, mkb_friendly, lokale_economie).
- **Three-tier hierarchy**: Basis, Significant, Ambitious (per RVO model).
- **Lifecycle**: Criteria are versioned; once a tender starts, it locks to that version.

### 2. Per-Tender MVI Recommendation
- **Intake**: Procurement officer selects procurement category (CPV or explicit).
- **Output**: Auto-generated "starter set" of MVI criteria, based on RVO recommendations, org portfolio goals, and chosen ambition level.
- **Customization**: Picker can add/remove criteria, adjust weightings, override proof types.

### 3. Scoring & Gunning Integration (EMVI)
- MVI gunning criteria are formulated for EMVI model (multi-criteria decision making).
- Score matrix is auto-suggested (e.g., CO₂-Ladder level 5→100pt, 4→80pt, …) but customizable.
- Scoring against uploaded proof (certificate expiry, etc.) is validated; override is logged.
- Awarded scores are locked into contract and become contract obligations.

### 4. SROI Obligation Automation
- **Trigger**: Tender value + category + region policy.
- **Calculation**: SROI percentage × Order Value = SROI Amount (EUR).
- **Optionality**: Below-threshold tenders may request voluntary SROI; above-threshold may apply for exemption (e.g., cloud software, no NL physical labor).
- **Model**: Bouwblokkenmethode 2018 or 2023 (regional policy choice).

### 5. SROI Realization & Approval Workflow
- **Leverancer inlet**: Quarterly or per-contract schedule; log hours/FTEs/persons per "bouwblok" (e.g., WW-eligible, 12+ months unemployed, placed 32+ hrs/wk).
- **Validation**: System calculates realized value (count × weighting factor), collects proof (payroll extract, placement letter, anti-stacking check).
- **Approval**: SROI Coordinator checks; approve, refuse, or request revision.
- **Escalation**: If realization lags >15% vs. plan at mid-point, escalation ticket + reminders.

### 6. Portfolio Dashboard
- **Real-time view**: % of spend that is MVI-compliant per category, cumulative SROI realized, progress toward portfolio goals (CO₂, Circular, etc.).
- **Drill-down**: Click a category → list of active contracts + their MVI/SROI status.
- **Traffic light**: Green (on target), Orange (at risk), Red (behind plan).

### 7. Annual MVOI Monitor & Governance Reporting
- **System output**: CSV+JSON in PIANOo/RIVM standard format.
- **Executive report**: College/Raad dashboard with summary, top performers, risk items.
- **Audit trail**: All decisions, score sheets, proof documents, goal revisions.

---

## User Journeys

### Procurement Officer: "Structure a Competitive Dialogue for IT Hardware"

1. **Tender Setup**: Opens new tender, selects "ICT Hardware" category.
2. **MVI Recommendation**: System suggests (based on org's "Circular 50% by 2030" goal):
   - CO₂ Ladder Significant (gunning criterion) — weight 25pt
   - Energy Star (minimum) — binary yes/no
   - 5-year spare parts guarantee (contract obligation)
   - Cradle-to-Cradle preferred (gunning, weight 10pt)
3. **Review & Customize**: Officer adjusts weight on Circular (up to 30pt) because the goal is near.
4. **SROI Obligation**: System auto-calculates: €800k tender value, "Rotterdam region" policy → 2% SROI = €16k.
   - Officer accepts, OR requests exemption for cloud-only (not applicable here).
5. **Publish**: MVI criteria are encoded into eForms BT-755 (Accessibility), BT-805/806 (Green Procurement) + TED; SROI is listed as a contract condition.
6. **Evaluation**: Commission scores CO₂-Ladder proofs; system validates certificate expiry and auto-scores. Officer can override with logged motivation.
7. **Award**: Winning bidder's MVI scores → contract obligation (e.g., "maintain CO₂ Ladder Level 4 throughout term").

### SROI Coordinator: "Process Q2 2026 SROI Reports"

1. **Reminder**: System reminds 10 vendors with active SROI obligations 14 days before Q2 deadline.
2. **Receipt**: Vendor reports "3 WW-eligible persons, avg 15 weeks, 36 hrs/wk" via portal + uploads payroll extracts.
3. **Calculation**: System: count (3) × hrs (36) × weeks (15) × weighting (€25/hr from BB-WW-12) = €16,200 realized.
4. **Escalation flag**: If year-to-date realization is <30% of annual goal, system flags as `risicovol` and creates escalation ticket.
5. **Approval**: Coordinator reviews proof, approves or requests revision. Cumulatively tracks progress.
6. **End-of-period**: If realization is 80% of obligation, system calculates penalty (25% of missing balance) and flags for invoicing.

### Portfolio Manager: "Prepare Q4 Report to College"

1. **Dashboard refresh**: System aggregates all active contracts, SROI realizations, MVI scores.
2. **Visual**: "CO₂ Reduction: 32% achieved (target 49% by 2030)" — shows trajectory and gap.
3. **Drill-down**: Click "Circular" goal → list of top 10 circular contracts + % MVI-compliant.
4. **Report generation**: "Generate College Report" → PDF with exec summary, KPIs, top performers, risk items, policy recommendations.
5. **RIVM export**: End-of-year, system generates PIANOo/RIVM format CSV for national census.

---

## Success Criteria

| Criterion | Target |
|-----------|--------|
| **Criteria Library Coverage** | ≥90% of RVO categories have ≥5 applicable MVI criteria (Basis + Sig + Amb) |
| **MVI Adoption Rate** | ≥80% of published tenders include ≥2 MVI gunning criteria (vs. 30% today) |
| **SROI Tracking** | 100% of eligible obligations tracked; ≥95% of Q-reports submitted on time |
| **Portfolio Visibility** | Dashboard monthly refresh; executive can answer "% MVI-compliant spend" within 24h (vs. weeks today) |
| **Audit Readiness** | 100% of scoring decisions, proofs, and exemptions logged with audit trail; external accountant signs off within 2 weeks of year-end. |
| **Behavioral Change** | Within 12 months, ≥70% of inkopers report "MVI is now standard practice, not a special project." |

---

## Stakeholders & Adoption

| Role | Primary Task | Adoption Effort |
|------|--------------|-----------------|
| **Inkoper** | Select MVI criteria per tender, evaluate bidder scores | Training: 4h; Workflow: unchanged (integrated) |
| **SROI-coördinator** | Approve Q-reports, escalate delays, calculate penalties | Training: 8h; Workflow: standardized (portal + system alerts) |
| **MVI-adviseur / Duurzaamheidscoördinator** | Curate criteria library, advise on portfolio goals, generate reports | Training: 6h; Workflow: centralized (vs. fragmented today) |
| **Categoriemanager** | Monitor MVI/SROI per category; advise on next tender round | Training: 4h; Workflow: dashboard-driven |
| **Contractmanager** | Monitor contract obligations (CO₂ ladder renewal, etc.); flag boetes/bonuses | Training: 4h; Workflow: system notifications |
| **Bestuurder (Wethouder/Gedeputeerde)** | Approve portfolio goals; review quarterly & annual reports | Training: 2h; Workflow: report delivery |
| **Compliance / Audit** | Validate MVI decisions, proof chain, and SROI calculations | Training: 4h; Workflow: audit-trail query |

---

## Dependencies & Risks

### Hard Dependencies
- **openregister**: All schemas (`mvi_criterium`, `sroi_verplichting`, etc.) must be registered.
- **openconnector**: RVO API import job; SKAO CO₂-Ladder API; PSO-register API.
- **tenderned-adapter** (purchaseq): eForms BT-755/805/806 mapping.
- **docudesk**: Tender text fragments, report templates.
- **mydash**: Portfolio dashboard widget (GraphQL queries on openregister).

### Key Risks & Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| RVO API downtime | Criteria library stale | Fallback to last-good version; ops alert; manual import retry |
| Mismatch: RVO criteria vs. local org goals | Conflicting MVI guidance | MVI-adviseur owns library curation; quarterly review cycle |
| Low SROI-coordinator capacity | Delays in approval; escalations unaddressed | Process training; escalation automation; regio coordination |
| Vendor non-compliance with proof submission | Untrackable SROI | Contractual clauses; portal reminders; penalty incentive |

---

## Timeline & Phasing

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| **P0: Schemas & Import** | 6 weeks | mvi_criterium + sroi_verplichting + bouwblok schemas live in openregister; RVO import job running |
| **P1: Intake & Recommendation** | 4 weeks | Procurement UI: "Suggest MVI for category" button; testing with 5 pilot tenders |
| **P2: Scoring & Gunning** | 3 weeks | EMVI integration; score matrix UI; proof validation |
| **P3: SROI Workflow** | 4 weeks | Vendor portal; approval UI; escalation job |
| **P4: Portfolio Reporting** | 4 weeks | mydash widget; college report generator; RIVM export |
| **P5: Optimization** | 2 weeks | Bug fixes; performance tuning; training rollout |

---

## Estimated Effort

| Workstream | Effort (person-weeks) |
|------------|----------------------|
| Data model & schemas | 3 |
| Backend APIs (MVI import, SROI calc, approval) | 8 |
| Frontend (Intake, scoring, vendor portal) | 6 |
| Integration (TenderNed, docudesk, mydash) | 4 |
| Testing (functional, integration, UAT) | 4 |
| Documentation & training | 2 |
| **Total** | **27 person-weeks** |

---

## Next Step

Approve proposal → Design phase begins (detailed entity schemas, API spec, UX wireframes, pilot plan).
