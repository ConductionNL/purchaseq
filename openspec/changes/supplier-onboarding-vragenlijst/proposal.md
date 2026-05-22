# Proposal: Supplier Onboarding — Vragenlijst & Workflow

## Problem Statement

Supplier intake today relies on PDF questionnaires mailed to crediteurenadministratie, manually re-typed into the ERP, with KvK and BTW re-typed (and often mistyped) into the general ledger. This process is:

- **Error-prone**: Manual data re-entry introduces typos; mismatched KvK/BTW blocks supplier activation.
- **Slow**: Turnaround from intake to crediteurenstam is 2-3 weeks.
- **Non-compliant**: No automated BIBOB risk screening; MVI proof collection is ad-hoc; Peppol data is incomplete.
- **Fragmented**: Supplier data lives in multiple systems (email, PDF, ERP, ledger) with no audit trail.

## Solution

purchaseq replaces manual intake with a **self-service supplier portal** plus an **internal approval pipeline**:

1. Leverancier (supplier) selects category (logistiek, zorg, bouw, etc.), completes a risk-tiered questionnaire, and uploads proof documents (KvK extract, IBAN proof, VOG, certifications).
2. purchaseq validates data in real time against external sources (KvK-API, VIES BTW, Surepay IBAN-check, Peppol directory).
3. Dossier routes through role-based approvals (categorie-inkoper → contractmanager → financien, plus parallel BIBOB if risk-hoog).
4. Upon approval, supplier data is pushed to ERP and crediteurenstam with clean master data.
5. Annual revalidation prompts keep data fresh; unchanged suppliers get 1-click confirmation; changed suppliers re-review in expedited track.

**Outcome**: Cleaner master data, faster Peppol onboarding, MVI/SDG conformance proof on record, BIBOB screening in-process, zero manual re-entry.

## Features

| Feature | Demand | Description |
|---------|--------|-------------|
| Category-driven questionnaire branching | ★★★★★ | Leverage supplier selects category; questionnaire shows only relevant questions for that category and risk tier. A logistics supplier does not see BIG registration questions; a healthcare provider does not see UAV-GC questions. |
| Real-time KvK validation | ★★★★★ | Supplier enters KvK number; on blur, purchaseq calls KvK-API, pre-fills company name, legal form, registered address, and SBI code. Cache hit or flag as error for manual review. |
| IBAN validation with name-on-account check | ★★★★★ | On save, purchaseq validates IBAN checksum and (via Surepay if available) verifies account holder name against registered company name. Mismatch blocks dossier approval until clarified. |
| BIBOB risk tiering and routing | ★★★★☆ | Dossier in category marked risk-hoog by municipal BIBOB policy (e.g., real estate, catering, construction above threshold) automatically routes to integrity coordinator. Lower-risk categories skip this step. |
| MVI compliance proof requirement | ★★★★☆ | Expected annual turnover > €50k triggers mandatory upload of MVI proof (PSO certificate, CO2 ladder, or local MVI self-declaration). Dossier cannot move to internal review without it. |
| Peppol directory validation | ★★★☆☆ | Supplier enters Peppol ID; purchaseq queries Peppol SML/SMP to confirm endpoint is active and which document types (UBL invoice, OrderResponse) are supported. Result logged on dossier. |
| Multi-step role-based approval | ★★★★★ | Dossier transitioning to internal review triggers sequential ApprovalSteps: categorie-inkoper → contractmanager → financien (IBAN/ERP), plus parallel integrity step if BIBOB assessment exists. Supplier pushed to ERP only after all approvals. |
| Annual revalidation workflow | ★★★☆☆ | Approved dossier reaching expiry date (default +12 months) triggers annual revalidation. Daily job emails leverancier with 1-click unchanged confirmation or full re-review for changes. Unresponded dossiers marked inactive after 60 days. |

## Stakeholders & Roles

| Role | User Type | Responsibility | Goal |
|------|-----------|-----------------|------|
| **Leverancier** | Self-service supplier | Complete questionnaire, upload proof docs, respond to annual revalidation | Seamless, quick onboarding with no manual re-typing |
| **Categorie-inkoper** | Internal category buyer | First review on sourcing fit and questionnaire completeness | Ensure supplier meets category sourcing requirements |
| **Contractmanager** | Contract reviewer | Review supplier conditions, MVI conformance, contract terms | Verify compliance with municipality standards before approval |
| **Crediteurenadministratie / Financien** | Finance & ERP admin | Validate IBAN, push to ERP crediteurenstam, manage duplicate/merge | Clean master data and timely activation in accounting system |
| **Integriteitscoördinator** | Integrity assessor | Perform BIBOB assessment for risk-hoog suppliers | Protect procurement from corruption/integrity risks |
| **Inkoopcoördinator** | Procurement coordinator | Monitor dossier backlog, KPIs on cycle time, escalate delays | End-to-end visibility and oversight of supplier pipeline |

## Cross-App Dependencies

- **openregister**: Leverancier, OnboardingDossier, VragenlijstAntwoord, Bewijsstuk, BIBOBToets, ApprovalStep schemas and CRUD.
- **openconnector**: Adapters to KvK-API, VIES BTW-validation, Surepay IBAN-check, Peppol-SMP, and ERP connectors (AFAS, Unit4, Exact, SAP).
- **docudesk**: Proof document archival with retention policies (7 years fiscal, 10 years contract).
- **decidesk**: Formal approval decisions for large contracts above mandate threshold.
- **opencatalogi**: Published supplier list under Woo transparency (public procurement > threshold amount).

## Standards Compliance

- **NEN-EN-IEC 16931** — European invoice standard (Peppol BIS Billing 3.0)
- **Peppol BIS 3.0** — e-invoicing and e-ordering
- **Wet BIBOB** — integrity assessment of public procurement
- **MVI criteria PIANOo** — Socially Responsible Procurement
- **PSO 30+** — Social Entrepreneurship Performance Ladder
- **CO2 Performance Ladder (SKAO)** — Environmental performance
- **KvK API & VIES BTW validation** — Dutch supplier registry and tax ID
- **ISO 20022** — Financial messaging including SEPA

## Success Metrics

1. **Cycle time**: Reduce supplier onboarding from 14+ days to <7 days (avg).
2. **Data quality**: >98% KvK/IBAN matches on first submission; zero re-entry typos in ERP.
3. **Compliance**: 100% BIBOB screening for risk-hoog; 100% MVI proof collection for >€50k suppliers.
4. **Adoption**: 90% of new suppliers complete self-service intake (vs. email + manual today).
5. **Revalidation**: 85% of suppliers confirm unchanged data via 1-click (vs. full re-entry).

---

**Status**: Draft — Ready for design phase  
**Author**: Specter Intelligence  
**Created**: 2026-05-22
