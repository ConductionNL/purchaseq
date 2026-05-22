---
app: purchaseq
spec: inhuur-derden-wnra-wnt
title: Inhuur Derden met Wet DBA en WNT Toetsing
status: proposal
version: 1.0
date: 2026-05-22
depends_on:
  - purchaseq base
  - shillinq dba-compliance-marker
---

# Proposal: Inhuur Derden met Wet DBA en WNT Toetsing

## Executive Summary

This specification adds compliance-driven procurement workflows to PurchaseQ for managing external hiring (ZZP, temporary workers, consultants, secondments) with embedded Wet DBA (tax classification), WNT (wage cap), WAADI (intermediary registration), and internal hiring ratio controls. The feature prevents the organisation from facing back-tax assessments, WNT publication penalties, or sham employment classifications by shifting compliance checks forward into the hiring process.

## Business Drivers

1. **Risk Mitigation**: The DBA enforcement moratorium ended January 1, 2026. Tax authorities now actively enforce tax reclassification penalties for misclassified contractor relationships. Current manual controls (individual spreadsheets, post-facto audits) leave organizations exposed until after tax/wage inspections identify violations.

2. **WNT Statutory Requirement**: Public-sector organizations must publish annual reports detailing external contractors earning above wage caps. Failure to publish or incomplete disclosures trigger administrative fines.

3. **Audit Trail Compliance**: Tax and wage audits require reproducible evidence of compliance checks (contract terms, risk assessments, modeling agreements) from the moment of hire through employment termination. Current systems lack this auditability.

4. **Central Governance**: External hiring often flows through departmental budgets without central procurement oversight, fragmenting compliance responsibility and creating blind spots.

## Target Users

- **Inkoper / Contract Manager**: Initiates hiring requests, selects vendors, evaluates DBA risk, manages contract lifecycle.
- **HR Business Partner**: Advises on contract classification, flags risky arrangements, coordinates with tax/wage compliance.
- **Controller / Finance Manager**: Monitors hiring ratios, reviews spending, approves "red" DBA assessments.
- **Bestuurder (WNT Accountable Officer)**: Signs off on high-risk hires, ensures WNT disclosures are complete.
- **Internal/External Auditor**: Validates hiring controls, reproduces compliance evidence, tests auditability.

## Key Features

### Feature 1: Mandatory DBA Assessment on Hire (Demand Score: 95)

**Description**: Every ZZP/consultant hiring decision triggers a standardized 12-question DBA risk assessment (employment relationship, instruction authority, replaceability, entrepreneur risk, own tools, own insurance, multiple clients, rate setting, invoice frequency). The system blocks contract activation until assessment is complete and risk score is green or authorized red (with three-signature approval).

**Acceptance Criteria**:
- DBA assessment form auto-populates with 12 standard questions per Belastingdienst guidance.
- Risk score calculated as 0-100; green <30, orange 30-60, red >60.
- Red assessments require explicit three-signature approval (dept head, controller, executive).
- Assessment embedded in contract audit trail; reproducible for tax audit.

### Feature 2: Model Agreement Enforcement (Demand Score: 90)

**Description**: ZZP and contractor hires MUST reference a valid model agreement (Belastingdienst, branch-specific, or bespoke). The system validates or blocks missing/expired references and guides the user to register the agreement and upload evidence (PDF of signed model, Belastingdienst reference number).

**Acceptance Criteria**:
- Model agreement field mandatory for ZZP hire type.
- Automatic check: Belastingdienst model agreement registry lookup.
- Block if no valid agreement or agreement >5 years old.
- Offer three paths: select existing model, upload branch model, or prepare bespoke.
- Retain agreement reference and upload date in contract record.

### Feature 3: Real-Time WNT Wage-Cap Monitoring (Demand Score: 92)

**Description**: For WNT-obligated organizations, the system tracks cumulative payments to each contractor per calendar year against the WNT norm. Invoices received via Shillinq trigger automatic aggregation; crossing 80% of the norm triggers escalation, 100%+ auto-creates a publication record.

**Acceptance Criteria**:
- Cumulative payment aggregation per contractor per calendar year.
- 80% threshold: warning notification to executive + WNT officer.
- 100% threshold: auto-create publication record with required fields (name, function, employment type, remuneration, end date).
- WNT norm per year (e.g., 2026: €150,000 for all but highest roles).
- Publication record auto-populated and ready for annual disclosure.

### Feature 4: Maximum Tenure and Re-Assessment Trigger (Demand Score: 88)

**Description**: ZZP arrangements over 6 months or budget overruns >25% trigger mandatory re-assessment to prevent "hidden" employment relationships. The system blocks new invoice approvals until re-assessment completes.

**Acceptance Criteria**:
- Automatic trigger at 6-month mark or 125% of budgeted spend.
- Block new invoices pending re-assessment completion.
- Re-assessment form: same 12 DBA questions, plus explicit question on changed relationship/reduced replaceability.
- Audit trail shows trigger date, trigger reason, re-assessment date, and updated risk score.

### Feature 5: WAADI Registration Validation (Demand Score: 87)

**Description**: Temporary staffing agencies and secondment vendors MUST have valid WAADI registration (Trade Register). The system queries the KvK adapter at vendor selection; invalid or missing registration blocks hiring.

**Acceptance Criteria**:
- WAADI validation via OpenConnector KvK adapter.
- Check triggered at vendor selection for hire type "uitzend" or "detachering".
- Block hire with red error if WAADI registration missing or expired.
- Annual re-check on contract anniversary.
- Audit trail shows validation date and result.

### Feature 6: Hiring Ratio Dashboard (Demand Score: 85)

**Description**: Controllers and executives see a cost-center or department-level dashboard showing external-to-total hiring ratio, trended over 8 quarters, with color-coded alarm against organizational norms. Drill-down to individual contracts reveals spending, tenure, and compliance status.

**Acceptance Criteria**:
- Dashboard widget: cost center selection, current ratio card (% external / total), 8-quarter trend line.
- Color code: green <30%, yellow 30-50%, orange 50-70%, red >70% (configurable per org).
- Drill-down table: contract ID, vendor, hire date, spend YTD, DBA status, WNT flag.
- Real-time aggregation from active contracts and Shillinq invoices.

### Feature 7: Three-Signature Flow for Red DBA (Demand Score: 89)

**Description**: "Red" DBA assessments (hign schijnzelfstandigheid risk) require explicit written sign-off from three roles: department head, controller, and executive. Each signature captures a required motivation; all three are logged and retained.

**Acceptance Criteria**:
- "Red" contract route to three-signature workflow (Decidesk integration).
- Each signer provides structured motivation (business justification, risk mitigation, approval date).
- Block contract activation until all three signatures obtained.
- Full audit trail: signer identity, timestamp, motivation text, signature type.

### Feature 8: End-of-Contract Closeout Checklist (Demand Score: 86)

**Description**: Within 30 days of contract end date, the system enforces a closeout checklist: final invoice received, NDA returned, access revoked, IT accounts deleted, knowledge-transfer doc uploaded, exit evaluation completed. All items must be checked before archival.

**Acceptance Criteria**:
- Trigger: contract end date entered or contract status changed to "termination".
- Mandatory checklist: final invoice, NDA/confidentiality, site access, IT accounts, knowledge transfer, exit evaluation.
- Block archival until all items checked.
- Audit trail: who checked item, when, and evidence (file uploads for knowledge transfer/eval).

### Feature 9: Annual Board Report Export (Demand Score: 91)

**Description**: WNT-obligated organizations export an annual board/annual report file (PDF and XML) containing all WNT-disclosable contracts, cross-referenced with top-executive compensation, and external hiring totals, formatted for publication on the organization website within the legal deadline (July 1).

**Acceptance Criteria**:
- Export triggered by WNT officer, scoped to closed calendar year.
- Output: PDF (formatted for website publication), XML (structured data per WNT schema).
- Content: all WNT-flagged contracts, name/function/income/end date, external hiring ratio, comparison to cap.
- Ready for publication; no additional reformatting needed.
- Audit trail: export date, user, year scope, output file hash.

### Feature 10: Immutable Audit Log (Demand Score: 93)

**Description**: Every change to a contract, DBA assessment, or WNT record is logged in a hash-chained audit trail with the change author, timestamp, field name, old value, and new value. The log is cryptographically tamper-evident for tax/wage audit reproduction.

**Acceptance Criteria**:
- Every create/update/delete captured: user, timestamp, field, before/after value.
- Hash chain: each log entry includes hash of previous entry (tamper detection).
- Exportable: full audit trail for contract / assessment / WNT record, in formats suitable for auditor review.
- Immutable: no log deletion or modification after creation.
- Integration: audit data indexed for compliance reporting.

## Success Metrics

| Metric | Target | Method |
|--------|--------|--------|
| DBA Red Assessments Authorized | ≤5% of total hires | Manual count from audit log |
| WNT Wage-Cap Breaches | 0 undetected | Monthly cross-check: invoices vs. publication records |
| Hiring Ratio Compliance | 100% within org norm | Dashboard KPI monthly review |
| Audit Trail Completeness | 100% of changes logged | Test: modify contract, verify log entry |
| Contract Closeout Time | <15 days median | Metric: end-date to archival-date |
| WAADI Validation Hit Rate | >95% of temp-hire vendors | Quarterly audit of WAADI checks |

## Dependencies

### Internal
- **purchaseq base**: Vendor, contract, and purchasing workflow foundation.
- **hrmq**: Personnel register for FTE counts (hiring ratio calculation).
- **decidesk**: Decision workflow for three-signature approvals.
- **docudesk**: Document vault and 7-year retention (contract archival).

### External / Cross-App
- **shillinq dba-compliance-marker**: Invoice ingestion with ZZP flag; triggers WNT aggregation and ratio updates.
- **openconnector kvk-adapter**: KvK Trade Register queries for WAADI validation and contractor profile enrichment.
- **openconnector belastingdienst-modelovereenkomst-register** (future): Automated model-agreement validity checks.

## Scope / Out of Scope

### In Scope
- DBA assessment workflow and risk scoring.
- Model agreement validation and storage.
- WNT wage-cap tracking and publication record generation.
- WAADI vendor registration checks.
- Hiring ratio aggregation and dashboard visualization.
- Three-signature approval flow for red DBA.
- End-of-contract closeout checklist.
- Annual board report export (PDF + XML).
- Immutable audit logging.
- Integration with Shillinq invoice receipts.

### Out of Scope
- CAO compliance checks (separate wage/benefits spec).
- Pension/insurance validation (vendor responsibility).
- Background checks or security vetting (HR/security function).
- Automated tax/wage form filing to Dutch government (future API integration).
- Real-time employee mis-classification re-classification workflow (tax authority recovery).

## Deliverables

1. **Backend**: DBA, WNT, WAADI, hiring-ratio, audit-log services; integrations with Shillinq, KvK, Decidesk.
2. **Frontend**: Hire request form (DBA/model-agreement inline), contract detail (assessment history, WNT tracking, closeout), hiring ratio dashboard, annual export UI.
3. **Data Model**: InhuurOpdracht, Opdrachtnemer, DbaAssessment, WntToets, InhuurRatio schemas; relations and indexes.
4. **Tests**: Unit tests for risk scoring, WNT aggregation, WAADI lookup; integration tests with Shillinq/KvK; E2E closeout checklist.
5. **Documentation**: User guide (hiring workflow), compliance guide (DBA/WNT standards), audit guide (log reproduction).

## Approval & Sign-Off

- **Product Owner**: Board Compliance / Finance
- **Architecture**: OpenRegister / Shillinq / KvK alignment
- **Compliance**: Legal / Tax review
- **Target Timeline**: Q3 2026 (6-month delivery window, phased: DBA-only → WNT → reporting)
