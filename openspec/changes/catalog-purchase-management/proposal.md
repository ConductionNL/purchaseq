---
kind: code
depends_on: []
chain: []
---

# Proposal: Catalog & Purchase Management — Shillinq

## Problem statement

Shillinq currently lacks a structured procurement catalog and purchase management workflow. Procurement officers, category managers, and financial controllers have no unified system for managing approved product and service catalogs, submitting purchase requisitions with approval flows, or ensuring that spending stays within contracted rates and framework agreements.

The absence of a managed catalog leads to maverick buying (purchases outside contracted channels), use of expired contracts, and an inability to meet Dutch public sector transparency and audit requirements under the Aanbestedingswet 2012 and EU Directive 2014/24/EU. Category managers maintain prices manually in spreadsheets, procurement advisors cannot track lot utilisation in real time, and financial controllers reconcile spend across siloed systems.

## Proposed change

Introduce the **Catalog & Purchase Management** module for Shillinq, building on OpenRegister's data and workflow platform to deliver:

- A managed **ProcurementCatalog** with versioned **CatalogItem** records, **PricingRule** tiers (volume, bundle, period discounts), and contract linkage
- **PurchaseRequisition** workflow from draft through multi-step approval to confirmed purchase order
- **BlanketPurchaseOrder** for recurring/framework spending with consumption tracking and release scheduling
- **StatementOfWork** for service-based procurement with milestone and deliverable tracking
- Immutable **ProcurementAuditLog** satisfying public sector transparency requirements
- DigiInkoop-compatible data model for Dutch government e-ordering
- Configurable spend dashboards for real-time procurement and contract budget monitoring

## Features (14, ordered by market demand)

| # | Feature | Demand | Category |
|---|---------|-------:|---------|
| 1 | Full audit trail for procurement decisions meeting public sector transparency requirements | 2009 | governance |
| 2 | Catalog content management with bulk import and automated data validation | 1621 | document-management |
| 3 | Purchase order amendment workflow with version tracking and approval | 1046 | core |
| 4 | Live configurable dashboards with real-time procurement and spend data | 1025 | media |
| 5 | DigiInkoop integration for Dutch government procurement | 500 | core |
| 6 | Guided buying with smart catalog recommendations and automatic policy compliance checks | 469 | governance |
| 7 | User-specific catalog views restricting purchasing options by role and department | 404 | security |
| 8 | Guided buying experience routing casual users to correct purchasing channels with policy enforcement | 389 | governance |
| 9 | Source-to-pay integration connecting sourcing outcomes to procurement execution | 269 | integration |
| 10 | Team collaboration with shared procurement workspace and activity feeds | 263 | collaboration |
| 11 | Automated purchase order creation from approved shopping carts with ERP integration | 242 | integration |
| 12 | Punchout catalog integration with automatic cart transfer and PO creation | 188 | integration |
| 13 | SOC 2 and GDPR compliance with audit trail for all procurement transactions | 184 | governance |
| 14 | Version quotes | 88 | document-management |

## Stakeholders

| Stakeholder | Role in this change |
|-------------|---------------------|
| **Chief Procurement Officer** | Consumes compliance dashboards; drives policy enforcement requirements; primary sponsor for audit trail feature (demand 2009) |
| **Category Manager** | Manages catalog lifecycle — creates/updates items, triggers bulk price updates when contracts renew, deactivates items before contract expiry |
| **Procurement Officer** | Creates purchase requisitions, tracks PO status, assesses supplier compliance and sovereignty requirements |
| **Procurement Advisor** | Manages framework agreements and call-off orders; sets lot budget ceilings; monitors lot utilisation and expiry |
| **Financial Controller** | Views contract spend dashboards; reconciles POs and invoices to contracts; approves project closure reconciliation |
| **Procurement Data Analyst** | Generates spend cubes by category/supplier/department; detects maverick spend; classifies spend by CPV code |
| **Supplier Onboarding Officer** | Initiates supplier offboarding with dependency checks against open POs and contracts |
| **ICT Procurement Officer** | Evaluates Shillinq for reuse; needs open-source maturity signals and Dutch compliance evidence |

## Linked user stories (representative)

- **Story 1** — Track product inventory levels (small business owner, stock level updates)
- **Story 2** — Update catalog prices from contract (category manager, bulk price update with history retention)
- **Story 3** — Deactivate items linked to expired contracts (category manager, 60-day expiry notification)
- **Story 4** — Import catalog from supplier punchout (category manager, OCI punchout connection)
- **Story 5** — Review catalog item usage statistics (category manager, order frequency and spend analytics)
- **Story 6** — Build spend cube by category, supplier, and department (data analyst, CPV/supplier/org-unit dimensions)
- **Story 7** — Classify spend by CPV code (data analyst, automated suggestion with confidence score)
- **Story 8** — Detect off-contract spend (data analyst, maverick spend identification)
- **Story 9** — Calculate maverick spend rate by department (data analyst, euro amount and percentage per department)
- **Story 10** — Send maverick spend alert to department manager (data analyst, monthly email with department summary)
- **Story 11** — Initiate supplier offboarding request (onboarding officer, dependency report before deactivation)
- **Story 12** — View contract spend dashboard (financial controller, committed vs actual spend per contract)
- **Story 13** — Reconcile purchase orders and invoices to contract (financial controller, discrepancy flagging)
- **Story 14** — Reconcile final project financials at closure (financial controller, residual budget return)
- **Story 15** — Assign admitted suppliers to framework lots (procurement advisor, eligible supplier validation)
- **Story 16** — Set budget ceiling per framework lot (procurement advisor, ceiling breach warning)
- **Story 17** — Track framework agreement expiry and lot utilisation (procurement advisor, 60-day expiry badge)
- **Story 18** — Set delivery schedule on call-off order (procurement advisor, partial delivery tracking)
- **Story 19** — View call-off history per supplier and lot (procurement advisor, spend concentration risk)

## Business value

- **Eliminates maverick spending** — catalog-based purchasing with policy compliance checks closes off-contract purchasing channels
- **Audit trail for compliance** — immutable ProcurementAuditLog satisfies Aanbestedingswet, GDPR, and SOC 2 requirements with no custom audit code (OpenRegister AuditTrailService + ProcurementAuditLog schema)
- **Real-time contract budget control** — financial controllers see committed vs actual spend per contract, preventing budget overruns
- **Dutch government e-ordering** — DigiInkoop integration enables compliant electronic ordering with government counterparties
- **Catalog rationalization** — usage statistics and bulk deactivation tooling reduces catalog sprawl, focusing spend on high-value contracted items
- **Spend analytics** — CPV classification and maverick spend detection provide procurement data analysts with actionable savings intelligence

## Entities introduced

Seven new OpenRegister schemas defined in `openspec/architecture/adr-000-data-model.md`:

| Schema | schema.org type | Purpose |
|--------|----------------|---------|
| `ProcurementCatalog` | `schema:Catalog` | Master catalog of approved products and services |
| `CatalogItem` | `schema:Product` | Individual catalog line with pricing and availability |
| `PricingRule` | `schema:PriceSpecification` | Volume, tier, bundle, and period discount rules |
| `PurchaseRequisition` | `schema:Order` | Formal purchase request with approval workflow |
| `BlanketPurchaseOrder` | `schema:Order` | Authorized spend limit with release scheduling |
| `StatementOfWork` | `schema:CreativeWork` | Service procurement with milestones and deliverables |
| `ProcurementAuditLog` | `schema:Action` | Immutable audit trail for all procurement actions |

## Out of scope

- TenderNed publication and formal tender management (separate change)
- Invoice matching and accounts payable processing (separate change)
- Contract lifecycle management — Contract, ContractMilestone, ContractRenewal entities (separate change)
- Payment processing and bank reconciliation (separate change)
- Supplier portal and supplier self-service (separate change)
- Budget planning and forecasting (separate change)
