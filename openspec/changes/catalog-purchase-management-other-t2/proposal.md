---
title: Catalog & Purchase Management — Shillinq — Other T2
kind: code
depends_on: []
chain: []
---

# Proposal: Catalog & Purchase Management — Shillinq — Other T2

## Summary

This change implements the core **catalog and purchase management** capabilities for
Shillinq — Conduction's open-source business administration suite for freelancers,
sole proprietors, SMBs, and Dutch government entities. It covers the full procurement
lifecycle from product catalog browsing through purchase requisition, purchase order
creation and approval, blanket order management, and procurement analytics (savings,
KPIs, dossier completeness).

The 49 features in this change were extracted from 23 Dutch government tender
documents and competitor analysis of 7% market coverage. All features fall in the
**other** category of the Shillinq feature taxonomy.

---

## Business Case

### Market Demand

| Tier | Features | Avg. demand | Description |
|------|----------|-------------|-------------|
| High (≥60) | 6 | 64 | Set quote expiry, add categories, spending cards, procurement automation, PO processing, risk-aware procurement |
| Medium (30–59) | 13 | 44 | Purchase requisition, PO templates, create/generate PO, PO revision/signatures, approval status, auto-approval, mobile procurement, Dutch procurement, define procurement teams |
| Low (<30) | 30 | 14 | Catalog management, dossier management, savings tracking, KPI tracking, product classification, e-procurement, blanket orders, document attachment, tiered pricing |

### Strategic Fit

Shillinq targets Dutch freelancers, SMBs, and government entities operating on
Nextcloud. Dutch government procurement requirements (BBV, IV3, SiSa, DigiInkoop)
demand auditable purchase trails, configurable approval chains, and compliance with
Dutch public procurement law. This change delivers:

1. **Product & Service Catalog** — internal catalogs with configurable pricing,
   product classification (300+ categories), and hosted buyer catalogs with managed
   content satisfy both commercial and Dutch public procurement needs.

2. **Purchase Requisition & Approval** — configurable requisition forms with both
   direct and indirect categories, multi-entity requisition support, auto-approval for
   low-value items under configurable thresholds, and approval status visibility.

3. **Purchase Order Lifecycle** — create, generate, revise, and digitally sign
   purchase orders; blanket purchase orders with scheduled release dates; PO templates
   for standardised ordering.

4. **Procurement Automation** — no-code autopilot rules for procurement automation
   without developer involvement, auto procurement rules engine.

5. **Procurement Analytics** — track catalog order delivery status, maintain and
   check procurement dossier completeness, track procurement savings and KPIs vs
   targets.

6. **Dutch Compliance** — Dutch procurement workflows aligned with DigiInkoop/TenderNed
   standards, risk-aware procurement, and demand-based procurement planning.

---

## Scope

### In scope

- Procurement catalog management (`ProcurementCatalog`, `CatalogItem`,
  `ProcurementCategory`)
- Purchase requisition with approval workflow (`PurchaseRequisition`,
  `ApprovalChain`, `ApprovalRequest`, `ApprovalTask`)
- Purchase order full lifecycle (`PurchaseOrder`, `PurchaseOrderRevision`,
  `PurchaseOrderChange`, `BlanketPurchaseOrder`)
- Procurement quotes with expiry date management (`ProcurementQuote`, `Quote`)
- Spending card management (`SpendTransaction`, `SpendCategory`)
- Procurement analytics: savings, KPIs, dossier (`SavingsOpportunity`,
  `ProcurementAuditLog`, `ProcurementComplianceReport`)
- Procurement team and role definition (`Team`, `Role`)
- Document attachment to requisitions (`DigitalDocument`)

### Out of scope (separate changes)

- Supplier portal and supplier qualification (covered by supplier management change)
- Contract lifecycle management (covered by contract management change)
- Bank reconciliation and payment matching (covered by accounts payable change)
- VAT/tax reporting (covered by tax compliance change)
- E-invoicing / Peppol integration (covered by Peppol change)

---

## Features by Priority

### P1 — Core Catalog & PO (demand ≥ 50)

| # | Feature | Demand |
|---|---------|--------|
| 1 | Set quote expiry date | 69 |
| 2 | Add Categories to Procurement Systems | 69 |
| 3 | Virtual and physical purchasing cards (Spending Cards) | 65 |
| 4 | No-code Autopilot for custom procurement automation | 65 |
| 5 | Purchase order processing | 62 |
| 6 | Risk-aware procurement | 62 |
| 7 | Procurement cloud | 57 |
| 8 | Purchase requisition with configurable forms (direct + indirect) | 56 |
| 9 | Purchase Order Templates | 55 |
| 10 | Create purchase order | 53 |
| 11 | Generate purchase order | 53 |
| 12 | Purchase Order Revision | 53 |
| 13 | Purchase Order Signatures | 53 |

### P2 — Approval & Automation (demand 30–49)

| # | Feature | Demand |
|---|---------|--------|
| 14 | View requisition approval status | 47 |
| 15 | Auto-approval for low-value requisitions under configurable thresholds | 42 |
| 16 | Mobile Procurement Management | 40 |
| 17 | Dutch Procurement | 33 |
| 18 | Define procurement team and roles | 33 |
| 19 | Online quote/estimate creation with one-click acceptance | 31 |
| 20 | Demand-Based Procurement | 30 |
| 21 | Track catalog order delivery status | 30 |

### P3 — Analytics & Compliance (demand 10–29)

| # | Feature | Demand |
|---|---------|--------|
| 22 | Blanket purchase orders with scheduled release dates and quantity limits | 29 |
| 23 | Procurement automation | 26 |
| 24 | Maintain procurement dossier | 24 |
| 25 | Track procurement savings | 21 |
| 26 | Track procurement KPIs vs targets | 21 |
| 27 | Product Classification (300+ categories) | 19 |
| 28 | Procurement Module | 18 |
| 29 | Hosted Buyer Catalogs with Managed Content | 17 |
| 30 | E-Procurement | 17 |
| 31 | Add Catalog Items to Order Basket | 15 |
| 32 | Product & Service Catalog | 15 |
| 33 | Check procurement dossier completeness | 15 |
| 34 | Custom Internal Product Catalogs | 14 |
| 35 | Confirm receipt of catalog order | 12 |
| 36 | Multi-entity requisition | 11 |
| 37 | Internal Product Catalog with Configurable Pricing | 11 |
| 38 | Purchase and Spend Management | 10 |
| 39 | Auto Procurement Rules | 10 |

### P4 — Extended Features (demand < 10)

| # | Feature | Demand |
|---|---------|--------|
| 40 | Blanket Orders | 9 |
| 41 | Internal Catalog with Curated Product Selection | 6 |
| 42 | Attach supporting documents to requisition | 6 |
| 43 | Commercial Proposals | 5 |
| 44 | PO templates for standardized ordering with pre-populated fields | 5 |
| 45 | Volume-Based Pricing with Quantity Discounts | 5 |
| 46 | Estimate Creation | 5 |
| 47 | Free-text and catalog-based ordering with custom form fields | 5 |
| 48 | Purchase Requisitions | 4 |
| 49 | Tiered Pricing with Graduated Rates | 4 |

---

## Inferred Stakeholders

| Role | Description |
|------|-------------|
| **Inkoper** (Procurement Officer) | Creates and manages purchase requisitions, orders, and catalogs |
| **Budgethouder** (Budget Holder) | Approves requisitions and POs within their budget authority |
| **Financieel medewerker** (Finance Staff) | Monitors spend, tracks savings and KPIs, manages procurement dossiers |
| **Leverancier** (Supplier) | Receives POs, confirms delivery, submits quotes |
| **Teammanager** (Team Manager) | Defines procurement teams and roles, delegates approval authority |
| **Systeembeheerder** (System Administrator) | Configures catalogs, approval chains, procurement automation rules |
| **Interne aanvrager** (Internal Requester) | Submits purchase requisitions for goods and services |

---

## Inferred User Journeys

### Journey 1: Catalog Browse & Add to Basket
**Trigger:** Interne aanvrager needs to order office supplies.
**Steps:** Browse ProcurementCatalog → filter by ProcurementCategory → add CatalogItem to basket → generate PurchaseRequisition.
**Pain points:** No central catalog; employees order ad-hoc from random suppliers.

### Journey 2: Purchase Requisition → Approval → PO
**Trigger:** Inkoper submits a requisition for IT equipment > €5.000.
**Steps:** Create PurchaseRequisition with supporting documents → ApprovalChain routes to Budgethouder → Budgethouder approves → system generates PurchaseOrder → PO sent to Supplier.
**Pain points:** Approval happens via email; no audit trail; PO generation is manual.

### Journey 3: Blanket Order Management
**Trigger:** Organisation has a framework agreement for cleaning supplies.
**Steps:** Create BlanketPurchaseOrder with quantity limits and scheduled releases → CallOffOrders created automatically on schedule → system tracks quantities consumed.
**Pain points:** Framework agreement limits are tracked in spreadsheets; manual release notifications.

### Journey 4: Procurement Analytics & KPI Review
**Trigger:** Financieel medewerker prepares quarterly procurement review.
**Steps:** Open procurement KPI dashboard → review SavingsOpportunity vs actuals → check ProcurementComplianceReport for dossier completeness → export report.
**Pain points:** Savings figures are not tracked; compliance dossiers are incomplete at audit time.

### Journey 5: Quote Expiry Management
**Trigger:** Inkoper receives a supplier quote with 30-day validity.
**Steps:** Create ProcurementQuote with expiry date → system sends reminder 7 days before expiry → Inkoper decides to accept or request extension → one-click acceptance generates PurchaseOrder.
**Pain points:** Quotes expire unnoticed; manual follow-up required.

---

## Entities Used (from master entity list — not redefined here)

`ProcurementCatalog` · `CatalogItem` · `ProcurementCategory` · `PurchaseRequisition` ·
`PurchaseOrder` · `PurchaseOrderRevision` · `PurchaseOrderChange` · `BlanketPurchaseOrder` ·
`ProcurementQuote` · `Quote` · `ApprovalChain` · `ApprovalRequest` · `ApprovalTask` ·
`SavingsOpportunity` · `SpendTransaction` · `SpendCategory` · `ProcurementAuditLog` ·
`ProcurementComplianceReport` · `ProcurementOrder` · `DigitalDocument` · `Team` · `Role` ·
`Supplier` · `Order`

---

## Acceptance Criteria (high-level)

1. Procurement officers can browse, search, and filter a managed product/service catalog.
2. Requisitions flow through configurable approval chains; status is visible to requesters.
3. Low-value requisitions below a configurable threshold are auto-approved.
4. Purchase orders can be created, revised, and signed digitally.
5. Blanket POs track consumed quantities and generate automatic call-off orders.
6. Quote expiry dates are enforced with configurable reminder notifications.
7. Procurement KPIs and savings are tracked and visible on the dashboard.
8. Procurement dossier completeness is checked automatically before PO submission.
9. All procurement actions are logged in `ProcurementAuditLog` for Dutch compliance.
10. The UI meets WCAG AA and NL Design System token requirements.
