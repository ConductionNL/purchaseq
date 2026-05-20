# Context Brief: Catalog & Purchase Management — Shillinq

**App:** Shillinq — Complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations. Combines bookkeeping, invoicing, procurement, and contract management into one self-hosted solution on Nextcloud.Named after the shilling — one of the oldest and most widely used coins in European history, from the Roman solidus to the British shilling to the East African shilling still in use today.Shillinq covers:- Bookkeeping & general ledger (double-entry accounting)- Accounts payable & receivable- Sales invoicing & e-invoicing (UBL/Peppol)- Purchase orders & procurement workflows- Supplier management & approval chains- Contract lifecycle management (creation, renewal, obligations)- Bank reconciliation & payment matching- VAT/tax reporting & compliance- Financial statements (P&L, balance sheet, cash flow)- Budget planning & forecasting- Multi-currency support- Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)
**Spec:** catalog-purchase-management
**Platform:** Nextcloud + OpenRegister

## Features (14 total, sorted by market demand)

### Full audit trail for procurement decisions meeting public sector transparency requirements
**demand: 2009** (669 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Catalog content management with bulk import and automated data validation
**demand: 1621** (537 tender mentions, 5% competitor coverage) | Category: document-management
Clustered from 2 mentions: 2 competitor features

### Purchase order amendment workflow with version tracking and approval
**demand: 1046** (346 tender mentions, 4% competitor coverage) | Category: core
Clustered from 1 mentions: 1 competitor features

### Live configurable dashboards with real-time procurement and spend data
**demand: 1025** (341 tender mentions, 1% competitor coverage) | Category: media
Clustered from 1 mentions: 1 competitor features

### DigiInkoop integration for Dutch government procurement
**demand: 500** | Category: core
Integration with DigiInkoop for electronic ordering with Dutch government organizations

### Guided buying with smart catalog recommendations and automatic policy compliance checks
**demand: 469** (155 tender mentions, 2% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### User-specific catalog views restricting purchasing options by role and department
**demand: 404** (134 tender mentions, 1% competitor coverage) | Category: security
Clustered from 1 mentions: 1 competitor features

### Guided buying experience routing casual users to correct purchasing channels with policy enforcement
**demand: 389** (129 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Source-to-pay integration connecting sourcing outcomes to procurement execution
**demand: 269** (89 tender mentions, 1% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### Team collaboration with shared procurement workspace and activity feeds
**demand: 263** (87 tender mentions, 1% competitor coverage) | Category: collaboration
Clustered from 1 mentions: 1 competitor features

### Automated purchase order creation from approved shopping carts with ERP integration
**demand: 242** (80 tender mentions, 1% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### Punchout catalog integration with automatic cart transfer and PO creation
**demand: 188** (62 tender mentions, 1% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### SOC 2 and GDPR compliance with audit trail for all procurement transactions
**demand: 184** (60 tender mentions, 2% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Version quotes
**demand: 88** (27 tender mentions, 3% competitor coverage) | Category: document-management
Clustered from 5 mentions: 1 user stories, 3 competitor features, 1 external mentions

## User Stories (70 linked)

### Story 1: Track product inventory levels
**Priority:** should
As a small business owner, I want to track stock levels of products I sell, so that I know when to reorder and never run out of stock

**Acceptance Criteria:**
GIVEN I have products in my catalog WHEN I sell or purchase items THEN inventory quantities are automatically updated AND I can see current stock levels per product

### Story 2: Update catalog prices from contract
**Priority:** should
As a category-manager, I want to bulk-update catalog item prices when a contract is renewed or price indexation is applied, so that catalog prices always reflect current contracted rates.

**Acceptance Criteria:**
GIVEN a contract has been renegotiated WHEN I initiate a price update for linked catalog items THEN all affected items are listed with old and new prices for review before applying
GIVEN I confirm the bulk price update WHEN it is applied THEN all order history retains original prices and only new orders use the updated prices

### Story 3: Deactivate items linked to expired contracts
**Priority:** should
As a category-manager, I want to receive notifications when a catalog item's underlying contract is about to expire and deactivate the item before the contract ends, so that requisitioners cannot order from an expired contract.

**Acceptance Criteria:**
GIVEN a catalog item is linked to a contract WHEN the contract expiry is 60 days away THEN I receive a notification listing affected catalog items
GIVEN the contract has expired WHEN a requisitioner searches the catalog THEN deactivated items are not shown in search results

### Story 4: Import catalog from supplier punchout
**Priority:** should
As a category-manager, I want to connect a supplier's OCI punchout catalog so that requisitioners can browse live supplier stock and pricing without me manually maintaining every product line.

**Acceptance Criteria:**
GIVEN a supplier provides an OCI punchout endpoint WHEN I configure the connection with the supplier's URL and credentials THEN requisitioners can browse the supplier catalog via punchout
GIVEN a requisitioner selects items in the punchout catalog WHEN they return to purchaseq THEN the items appear in their basket with price and product data pre-filled

### Story 5: Review catalog item usage statistics
**Priority:** should
As a category-manager, I want to see which catalog items are ordered most frequently and which have never been used, so that I can rationalize the catalog and focus on high-value items.

**Acceptance Criteria:**
GIVEN I view catalog analytics WHEN I select a date range THEN I see order frequency, total spend, and number of unique requisitioners per catalog item
GIVEN items have zero orders in 12 months WHEN I view the unused items report THEN I can deactivate them in bulk or mark them for review

### Story 6: Build spend cube by category, supplier, and department
**Priority:** should
As a procurement-data-analyst, I want to generate a spend cube that breaks down total expenditure by category, supplier, and department, so that I can identify consolidation opportunities and negotiate better contracts.

**Acceptance Criteria:**
GIVEN I select a fiscal year WHEN I generate the spend cube THEN total spend is displayed across three dimensions: CPV category, supplier, and organizational unit
GIVEN the spend cube is generated WHEN I drill down on a cell THEN I see the individual purchase orders and contracts that make up that cell's value

### Story 7: Classify spend by CPV code
**Priority:** should
As a procurement-data-analyst, I want unclassified spend to be automatically suggested a CPV code using the item description and category, so that the spend cube has complete coverage without manual data entry.

**Acceptance Criteria:**
GIVEN purchase orders exist without a CPV code WHEN the classification job runs THEN each unclassified line is assigned a suggested CPV code with a confidence score
GIVEN a suggested CPV code is presented WHEN I review the suggestion THEN I can accept it, edit it, or reject it and assign a different code

### Story 8: Detect off-contract spend
**Priority:** should
As a procurement-data-analyst, I want the system to identify purchases made from suppliers where a framework contract exists but was not referenced, so that I can quantify and report maverick spend.

**Acceptance Criteria:**
GIVEN purchase orders exist without a contract reference WHEN the maverick spend detection runs THEN orders are cross-referenced with active contracts by supplier and category
GIVEN a matching contract exists WHEN the analysis is complete THEN the order is flagged as maverick spend with the contract that should have been used

### Story 9: Calculate maverick spend rate by department
**Priority:** should
As a procurement-data-analyst, I want to calculate the maverick spend rate as a percentage of total spend per department, so that I can identify where compliance training or process improvement is most needed.

**Acceptance Criteria:**
GIVEN maverick spend has been identified WHEN I view the maverick spend dashboard THEN each department shows its maverick spend as a euro amount and percentage of total spend
GIVEN I select a department WHEN I drill down THEN I see the individual orders, requisitioners, and categories driving the maverick spend

### Story 10: Send maverick spend alert to department manager
**Priority:** should
As a procurement-data-analyst, I want to automatically send a monthly maverick spend summary to department managers, so that they are accountable for improving compliance in their teams.

**Acceptance Criteria:**
GIVEN the monthly maverick report is generated WHEN it contains maverick spend above the threshold THEN the relevant department manager receives an email with their department's summary and a link to the details
GIVEN a manager receives the alert WHEN they click the link THEN they see the specific orders and can initiate a corrective action

### Story 11: Initiate supplier offboarding request
**Priority:** should
As a supplier onboarding officer, I want to initiate a structured offboarding process for a supplier, so that all dependencies are identified and handled before the supplier is deactivated.

**Acceptance Criteria:**
GIVEN an active supplier record WHEN I initiate offboarding THEN the system checks for open purchase orders, active contracts, and pending invoices linked to the supplier and presents a dependency report
GIVEN the dependency report is shown WHEN open items exist THEN the system prevents deactivation until all items are resolved or explicitly overridden with a justification

### Story 12: View contract spend dashboard
**Priority:** should
As a financial controller, I want to view a real-time dashboard of committed and actual spend per active contract, so that I can monitor budget consumption and prevent budget overruns.

**Acceptance Criteria:**
GIVEN active contracts exist WHEN I open the contract budget dashboard THEN each contract shows the contracted value, total purchase orders raised, total invoices approved, and remaining budget as a percentage
GIVEN I filter by department or cost centre THEN the dashboard updates to show only contracts assigned to the selected filter

### Story 13: Reconcile purchase orders and invoices to contract
**Priority:** should
As a financial controller, I want to reconcile all purchase orders and approved invoices to their parent contract, so that I have a single source of truth for contract budget consumption in my financial reporting.

**Acceptance Criteria:**
GIVEN a contract is selected WHEN I open the reconciliation view THEN all linked purchase orders and invoices are listed with amounts, approval dates, and posting periods
GIVEN a purchase order or invoice references a contract but the values do not match the contract line items THEN the discrepancy is flagged for review with the amounts in question highlighted

### Story 14: Reconcile final project financials at closure
**Priority:** should
As a financial controller, I want to review and reconcile all financial records before a project is formally closed, so that the final account is accurate and any residual budget is returned.

**Acceptance Criteria:**
GIVEN a closure request is submitted WHEN I open the financial reconciliation view THEN I see total approved budget, total actual spend, open purchase orders, and remaining balance
GIVEN I approve the reconciliation WHEN the project is closed THEN the residual budget is marked for return and a reconciliation report is generated

### Story 15: Assign admitted suppliers to framework lots
**Priority:** should
As a procurement advisor, I want to assign the admitted suppliers to each lot of a framework agreement, so that call-off orders can be validated against the list of eligible suppliers.

**Acceptance Criteria:**
GIVEN a framework agreement has been created with lots
WHEN I add a supplier to a lot
THEN the supplier appears in the lot's admitted suppliers list

GIVEN a call-off order is created against a lot
WHEN the supplier is not in the admitted suppliers list for that lot
THEN the system blocks the call-off and shows an error

### Story 16: Set budget ceiling per framework lot
**Priority:** should
As a procurement advisor, I want to set the maximum budget ceiling for each lot of a framework agreement, so that the cumulative value of call-off orders does not exceed the published framework value.

**Acceptance Criteria:**
GIVEN a lot exists within a framework
WHEN I set a budget ceiling on the lot
THEN all call-off orders linked to that lot are summed and compared against the ceiling

GIVEN the ceiling would be breached by a new call-off
WHEN the call-off is submitted
THEN the system warns the procurement advisor and requires explicit justification to proceed

### Story 17: Track framework agreement expiry and lot utilisation
**Priority:** should
As a procurement advisor, I want to see the remaining budget and days until expiry for each lot of a framework agreement, so that I can plan purchases and avoid placing call-off orders after the framework has expired.

**Acceptance Criteria:**
GIVEN a framework agreement is active
WHEN I open the framework dashboard
THEN each lot shows consumed budget, remaining budget, percentage utilised, and days until expiry

GIVEN a framework lot is within 60 days of expiry
WHEN I view the dashboard
THEN an expiry warning badge is shown on the lot

### Story 18: Set delivery schedule on call-off order
**Priority:** should
As a procurement advisor, I want to set a delivery schedule with individual delivery lines on a call-off order, so that the accounts payable officer and contract manager can track partial deliveries.

**Acceptance Criteria:**
GIVEN a call-off order is being created
WHEN I add delivery lines with quantities, delivery dates, and delivery addresses
THEN the delivery schedule is stored against the call-off

GIVEN a delivery line is due
WHEN the delivery date arrives
THEN the contract manager receives a reminder to confirm receipt

### Story 19: View call-off history per supplier and lot
**Priority:** should
As a procurement advisor, I want to view all call-off orders placed with a specific supplier under a framework, so that I can assess spend concentration and dependency risk.

**Acceptance Criteria:**
GIVEN call-off orders exist for a framework
WHEN I filter by supplier
THEN all orders placed with that supplier are listed with value, date, and status

GIVEN I view the supplier call-off history
WHEN I click on an order
THEN the full order detail including delivery schedule and payment status is shown

### Story 20: Create campaign with multiple touchpoints
**Priority:** should
As a communication officer, I want to create an engagement campaign with scheduled touchpoints across email, SMS, and social media, so that citizens are reached at the right moment through the right channel.

**Acceptance Criteria:**
GIVEN I start a new campaign WHEN I add touchpoints THEN I can configure each touchpoint with: channel, scheduled date/time, target segment, and message content
GIVEN multiple touchpoints are configured WHEN I view the campaign timeline THEN all touchpoints are displayed in chronological order with their channels colour-coded
GIVEN a touchpoint's scheduled time has passed WHEN I view the campaign THEN the touchpoint shows a 'sent' status with delivery statistics

## Stakeholders (12 linked)

### Chief Procurement Officer
Senior manager responsible for procurement policy, category strategies, and organisational compliance with Dutch and EU procurement law. Oversees the procurement function and reports to the board.
**Responsibilities:** setting procurement policy, ensuring Aanbestedingswet compliance, managing procurement team, approving high-value contracts, reporting to management and supervisory bodies
**Pain points:** lack of real-time compliance dashboards, difficulty demonstrating proportionality decisions, fragmented spend visibility across departments, audit preparation overhead
**Goals:** demonstrate full legal compliance, achieve strategic savings targets, professionalise the procurement function, reduce maverick spend

### Data Engineer
Technical specialist who designs and maintains the ETL pipelines that feed source system data into the municipal data warehouse or lakehouse underpinning goviq. Bridges operational systems and the analytics layer.
**Responsibilities:** building and monitoring ETL/ELT pipelines, integrating with BRP, GBA, SZW, ERP and GIS systems, managing data freshness SLAs, implementing data quality checks, maintaining connection credentials
**Pain points:** brittle API connections breaking silently, no centralised pipeline monitoring, inconsistent data models across source systems (e.g. Centric vs. Pink Roccade), difficulty propagating schema changes without breaking downstream reports
**Goals:** declarative pipeline configuration with automatic lineage tracking, instant alerting on ingestion failures, a metadata catalogue that auto-discovers schema changes

### ICT Procurement Officer
Responsible for sourcing and procuring software solutions for a government organization. Evaluates existing open source options before initiating new procurement procedures.
**Responsibilities:** evaluate reuse candidates before procurement, assess total cost of ownership, comply with 'comply or explain' open source policy, document procurement decisions
**Pain points:** difficult to find verified government-grade software, unclear maturity signals, no standardized comparison framework, risk of duplicate procurement across municipalities
**Goals:** find compliant ready-to-use software quickly, reduce procurement lead time, demonstrate due diligence for open source reuse, avoid reinventing existing solutions

### Procurement Officer
Procurement officer managing software contracts and licenses
**Responsibilities:** Contract management; license compliance; vendor assessment; cost optimization
**Pain points:** No link between contracts and actual usage; shelfware; license non-compliance risk
**Goals:** Software usage vs contract comparison; license compliance dashboard; renewal alerts

### Procurement Officer
Procurement officer evaluating vendor sovereignty requirements in tenders
**Responsibilities:** Sovereignty requirements in tenders; vendor assessment on EU compliance; contract clauses
**Pain points:** No standard sovereignty criteria; vendor claims hard to verify; contract gaps
**Goals:** Sovereignty assessment checklist; vendor verification tools; standard contract clauses

### Customer
Buyer of products or services
**Responsibilities:** Receiving invoices, making payments, requesting quotes
**Pain points:** Incorrect invoices; difficulty getting credit notes
**Goals:** Professional invoicing; easy payment options

### Procurement Officer
Purchasing manager sourcing and negotiating with suppliers
**Responsibilities:** Supplier selection, negotiation, PO creation, contract management
**Pain points:** No visibility on spend; maverick buying
**Goals:** Centralized procurement with spend analytics

### Procurement Officer
Government procurement specialist
**Responsibilities:** Public procurement, tender management, supplier selection
**Pain points:** Aanbestedingswet compliance; lengthy procedures
**Goals:** Compliant procurement workflow; automated documentation

### Real Estate Agent
Licensed real estate professional managing property sales and client relationships
**Pain points:** ["Managing multiple property viewings", "Tracking buyer interest across listings", "Following up on leads from portals like Funda"]
**Goals:** ["Centralize property leads", "Automate viewing scheduling", "Track buyer journey from inquiry to closing"]

### Inkoper Overheid
Government procurement officer managing vendor relationships and tenders
**Pain points:** ["Vendor relationship tracking", "Tender response management", "Contract compliance monitoring"]
**Goals:** ["Vendor CRM integration", "Tender pipeline management", "Contract milestone tracking"]

## Data Model — Entities for This Spec (7)

These entities MUST be implemented as OpenRegister schemas.
OpenRegister provides: CRUD, REST API, search, import/export, audit trails, file attachments.
Do NOT rebuild these platform capabilities.

### BlanketPurchaseOrder (`schema:Order`)
_Master purchase order with authorized spend limit, scheduled release management, and consumption tracking for blanket purchasing arrangements_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| blanketPoNumber | string | Yes | Unique blanket PO identifier |
| validFrom | datetime | Yes | Blanket PO effective start date |
| validUntil | datetime | Yes | Blanket PO expiration date |
| totalAuthorizedAmount | number | Yes | Total authorized spend limit |
| consumedAmount | number | No | Amount spent against blanket PO to date |
| remainingAmount | number | No | Remaining authorized spend |
| releaseSchedule | array | No | Scheduled release dates and amounts |
| status | string | Yes | active, closed, cancelled |

**Relations:**
- → Organization (many-to-one)
- → ProcurementCatalog (many-to-one)
- → PurchaseOrder (one-to-many)
- → ApprovalRequest (many-to-one)

### CatalogItem (`schema:Product`)
_Individual product or service in a procurement catalog with pricing, availability, lead time, and purchase price information_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| itemCode | string | Yes | Unique item code within catalog |
| itemName | string | Yes | Display name of the item |
| description | string | No | Detailed item description |
| basePrice | number | Yes | Base unit price |
| unit | string | Yes | Pricing unit: piece, kg, liter, hour, etc |
| minimumQuantity | number | No | Minimum order quantity |
| leadTime | number | No | Delivery lead time in days |
| status | string | Yes | active, discontinued |
| validFrom | datetime | No |  |
| validUntil | datetime | No |  |

**Relations:**
- → ProcurementCatalog (many-to-one)
- → Product (many-to-one)
- → PricingRule (one-to-many)

### PricingRule (`schema:PriceSpecification`)
_Volume discounts, tiered pricing, bundle discounts, and promotional pricing rules with validity periods and application priorities_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| ruleCode | string | Yes | Unique pricing rule identifier |
| description | string | No | Rule description and conditions |
| ruleType | string | Yes | volumeDiscount, tierPricing, bundleDiscount, periodDiscount |
| minQuantity | number | No | Minimum quantity for rule application |
| maxQuantity | number | No | Maximum quantity for rule application |
| discountPercentage | number | No | Percentage discount (0-100) |
| discountAmount | number | No | Fixed discount amount in base currency |
| priority | number | No | Priority order for rule application |
| validFrom | datetime | No |  |
| validUntil | datetime | No |  |

**Relations:**
- → CatalogItem (many-to-one)

### ProcurementAuditLog (`schema:Action`)
_Immutable audit trail recording all procurement actions, approvals, rejections, and changes for transparency, compliance, and decision accountability_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| auditId | string | Yes | Unique audit log entry identifier |
| entityType | string | Yes | Entity type: requisition, purchaseOrder, invoice, payment, approval |
| entityId | string | Yes | ID of the entity being audited |
| actionType | string | Yes | created, updated, approved, rejected, posted, received |
| timestamp | datetime | Yes | When the action occurred |
| reason | string | No | Reason or comment for the action |
| changes | object | No | Changed fields with old and new values |
| referenceDocuments | array | No | Related document identifiers |

**Relations:**
- → Person (many-to-one)
- → Organization (many-to-one)

### ProcurementCatalog (`schema:Catalog`)
_Master catalog of products and services available for organizational procurement with support for multiple formats (cXML, CIF, internal)_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| catalogNumber | string | Yes | Unique catalog identifier |
| catalogName | string | Yes | Display name of the catalog |
| description | string | No | Catalog description and scope |
| catalogFormat | string | No | Format type: internal, cxml, cif |
| status | string | Yes | draft, active, archived |
| validFrom | datetime | No | Catalog effective start date |
| validUntil | datetime | No | Catalog expiration date |

**Relations:**
- → Organization (many-to-one)
- → CatalogItem (one-to-many)

### PurchaseRequisition (`schema:Order`)
_A formal request for goods or services with multiple line items and custom fields, supporting multi-location and multi-entity procurement workflows_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| requisitionNumber | string | Yes | Unique requisition identifier |
| requisitionDate | datetime | Yes | Date requisition was created |
| status | string | Yes | draft, submitted, approved, rejected, ordered |
| purpose | string | No | Purpose or business justification |
| deliveryDate | datetime | No | Requested delivery date |
| customFields | object | No | Custom fields for procurement-specific data |
| totalAmount | number | No | Estimated total value |

**Relations:**
- → Person (many-to-one)
- → Organization (many-to-one)
- → ApprovalRequest (one-to-many)

### StatementOfWork (`schema:CreativeWork`)
_Detailed specification of deliverables, milestones, payment terms, and service scope for statement-of-work-based procurement and service ordering_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| sowNumber | string | Yes | Unique SOW identifier |
| sowDate | datetime | Yes | Date SOW was created |
| title | string | Yes | SOW title |
| description | string | No | Detailed description of work |
| scope | string | No | Work scope and boundaries |
| deliverables | array | No | Array of deliverable items with descriptions and due dates |
| milestones | array | No | Payment milestone objects with completion dates and invoice triggers |
| totalValue | number | Yes | Total SOW value |
| currency | string | Yes | Currency code |
| status | string | Yes | draft, active, completed, cancelled |

**Relations:**
- → Organization (many-to-one)
- → Person (many-to-one)
- → Contract (many-to-one)
- → PurchaseOrder (one-to-many)

## Other App Entities (do NOT redefine, reference only)

APTransaction, Account, AccountabilityReport, Administration, AllocationRule, ApprovalChain, ApprovalRequest, ApprovalRoute, ApprovalTask, AssessmentCriteria, Assignment, Auction, AuditFinding, AuditorStatement, AwardDecision, AwardNotice, BalanceSheet, BankAccount, Bid, BidEvaluation, BiddingRound, Branch, Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CallOffOrder, CashAccount, ChargebackDispute, ComplianceAssessment, ComplianceAudit, ComplianceDocument, ComplianceReport, ComplianceRisk, ConsentRecord, ConsolidatedReport, ConsolidationGroup, Contract, ContractClause, ContractMilestone, ContractModification, ContractObligation, ContractParty, ContractPerformance, ContractRedline, ContractRenewal, ContractSpendRecord, ContractTemplate, Corporation, CostAllocation, CostCenter, CostProject, CreditNote, CurrencyBalance, DebitNote, Deduction, Delegation, DelegationRule, DepreciationSchedule, DigitalDocument, Dividend, Document, DunningNotice, Entitlement, Entity, EvaluationCriterion, Event, ExemptionCertificate, ExpenditureEscalation, ExpenditureRequest, Expense, ExpenseCategory, ExpenseClaim, ExpenseLineItem, ExpenseReport, FXExposure, FinancialDecision, FinancialReport, FiscalYear, FixedAsset, FrameworkAgreement, Freelancer, FundAllocation, FundingSource, GeneralLedgerAccount, GeneralLedgerEntry, GoodsReceipt, GovernmentEntity, Grant, GrantPortfolio, IntercompanyTransaction, InventoryItem, InventoryStock, InventoryValuation, Investment, Invoice, InvoiceLine, JointVenture, JournalEntry, LiquidityForecast, Location, Lot, ManagementLetter, Mandate, MandateAuditLog, MandateRequest, MandateScheme, MandateViolation, MarketplaceApp, MarketplaceIntegration, MaverickSpendAlert, MonetaryAmount, OAuthIntegration, Obligation, ObligationSettlement, ObligationTask, Offer, Order, Organization, Payee, Payment, PaymentBatch, PaymentFraudAssessment, PaymentRiskScore, Payroll, PeppolAccessPoint, PeppolParticipant, PerDiem, PerformanceImprovementAction, PerformanceScore, Permission, Person, PolicyRule, PolicyViolation, ProcurementCategory, ProcurementComplianceReport, ProcurementOrder, ProcurementProcedure, ProcurementQuote, Product, Project, ProjectTask, ProofOfDelivery, Property, PropertyAssessment, PublicProcurement, PublicationAmendment, PublicationLog, PublicationNotice, PurchaseOrder, PurchaseOrderChange, PurchaseOrderRevision, QualificationDeclaration, QualityManagementSystem, Quote, RateCard, Receipt, Report, RequestForQuotation, RevenueStream, RiskCriteria, Role, SavingsOpportunity, ScheduledPayment, ServiceLevelAgreement, SettlementDecision, Share, Shareholder, SigningAuthority, SourcingEvent, SpendCategory, SpendTransaction, SpendingRecord, SubmissionDossier, Subscription, SubsidyApplication, SubsidyScheme, Supplier, SupplierBid, SupplierCertificate, SupplierDocument, SupplierKPI, SupplierPerformanceReport, SupplierPerformanceScore, SupplierPerformanceScorecard, SupplierPortalAccount, SupplierPortalUser, SupplierQualification, SupplierRiskProfile, SupplierSLA, SupplierSurvey, SupplyChainRisk, TaxConfiguration, TaxDeclaration, TaxExemption, TaxLot, TaxRate, TaxReturn, TaxableTransaction, Team, Tender, TenderAmendment, TenderDocument, TenderLineItem, TenderLot, TenderNotice, TimeEntry, Timesheet, Transaction, TreasuryTask, TrialBalance, User, UserPreference, VATReturn, VendorBill, WOZAssessment, XBRLInstance, XBRLTaxonomy

## Company-Wide Architecture Rules (13 ADRs)

These rules are MANDATORY for all Conduction apps.

### ADR-001-data-layer
- ALL domain data → OpenRegister objects. NO custom Entity/Mapper for domain data.
- App config → `IAppConfig`. NOT OpenRegister.
- Schemas: PascalCase, schema.org vocabulary where equivalent exists, explicit types.
- Cross-entity references: OpenRegister relations (register+schema+objectId). NO foreign keys.
- Register templates: `lib/Settings/{app}_register.json` (OpenAPI 3.0 + x-openregister).
- Seed data: 3-5 realistic objects per schema using `@self` envelope (`register`, `schema`, `slug`).
  Use general org data (municipality/consultancy), NOT context-specific. Include in design.md.
- Breaking schema changes → new migration in repair step. NEVER modify existing migrations.

### OpenRegister + @conduction/nextcloud-vue — DO NOT REBUILD

The platform provides 258+ backend methods and 69+ frontend components. Apps ONLY build
custom logic for domain-specific business rules. Everything below is provided for FREE.

**CRUD & Data Management** (use ObjectService + CnIndexPage + CnDetailPage):
- Single & bulk create, read, update, delete — `ObjectService.saveObject()`, `deleteObject()`
- List with pagination, sorting, filtering — `ObjectService.findAll()` + `CnDataTable`
- Schema-driven forms — `CnFormDialog` (auto-generates from schema) or `CnAdvancedFormDialog`
- Detail views — `CnDetailPage` with `CnDetailGrid`, `CnDetailCard` sections
- Record merging/deduplication — `ObjectService.mergeObjects()`
- Object locking — `ObjectService.lockObject()` / `unlockObject()`

**Import & Export** (use ImportService/ExportService + CnMassImportDialog/CnMassExportDialog):
- CSV, Excel, JSON import with intelligent field mapping — `ImportService`
- CSV, Excel, JSON export with column selection — `ExportService`
- Bulk import with validation and progress — `CnMassImportDialog`
- Filtered export with format picker — `CnMassExportDialog`
- NO custom import dialogs, parsers, upload handlers, or export controllers

**Search & Discovery** (use IndexService + CnFilterBar + CnFacetSidebar):
- Full-text search with field weighting — `IndexService`
- Faceted navigation with counts — `FacetBuilder` + `CnFacetSidebar`
- Semantic search with embeddings — `VectorizationService`
- Hybrid search (keyword + semantic) — automatic
- Search analytics — `SearchTrailService` (popular terms, activity)
- NO custom search endpoints, query builders, or search pages

**File Management** (use FileService + CnObjectSidebar):
- Upload (single/multipart), download, share links — `FileService`
- File tagging, public/private toggle — `FileService`
- Bulk download as ZIP — `createObjectFilesZip()`
- Text extraction from PDFs/Office docs — `TextExtractionService`
- File tab in object sidebar — `CnObjectSidebar` → `CnFilesTab`
- NO custom file upload components, file controllers, or download handlers

**Audit & Compliance** (use AuditTrailService + CnObjectSidebar):
- Full change tracking with before/after snapshots — automatic
- Audit trail tab — `CnObjectSidebar` → `CnAuditTrailTab`
- GDPR data subject access requests — `inzageverzoek()`, `verwerkingsregister()`
- Audit export and analytics — `AuditTrailController`
- NO custom audit logging, change tracking, or compliance controllers

**Dashboard & Analytics** (use CnDashboardPage + CnChartWidget + CnStatsBlock):
- Drag-drop widget dashboard — `CnDashboardPage` with GridStack
- KPI cards — `CnKpiGrid`, `CnStatsBlock`, `CnStatsPanel`
- Charts (line/bar/pie/donut) — `CnChartWidget` (ApexCharts)
- Data tables as widgets — `CnTableWidget`
- Editable data grids — `CnObjectDataWidget`
- NO custom dashboard layouts, chart components, or KPI cards

**Forms & Dialogs** (use CnFormDialog + schema-driven generation):
- Auto-generated create/edit forms — `CnFormDialog` reads schema → generates fields
- JSON/metadata editing — `CnAdvancedFormDialog` with Properties/Data/Metadata tabs
- Schema editor — `CnSchemaFormDialog`
- Delete/Copy/Mass operations — `CnDeleteDialog`, `CnCopyDialog`, `CnMassDeleteDialog`
- NO custom form components, validation logic, or dialog wrappers

**Navigation & Pagination** (use CnPagination + CnActionsBar + useListView):
- Pagination control with size selector — `CnPagination`
- Action bar (add, search, toggle views) — `CnActionsBar`
- List state management — `useListView` composable (handles search, filter, sort, page)
- Detail state management — `useDetailView` composable
- NO custom pagination logic, debounced search, or list state management

**Authorization & RBAC** (use AuthorizationService + PropertyRbacHandler):
- Role-based access control — `AuthorizationService`
- Field-level permissions — `PropertyRbacHandler`
- Object-level restrictions — `PermissionHandler`
- Authorization audit — `AuthorizationAuditService`
- NO custom permission checks, role systems, or access control middleware

**Webhooks & Events** (use WebhookService):
- Create, test, retry webhooks — `WebhookService`
- CloudEvents format — automatic
- Event subscriptions — selective per schema/action
- NO custom webhook controllers or event dispatchers

**Notifications & Activity** (use NotificationService + ActivityService):
- Nextcloud notifications — `NotificationService`
- Activity feed — `ActivityService`
- Calendar events — `CalendarEventService`
- Deck/Kanban cards — `DeckCardService`

**Store & State** (use createObjectStore + plugins):
- Object stores — `createObjectStore(name)` generates Pinia CRUD store
- Store plugins: `auditTrails`, `files`, `lifecycle`, `relations`, `search`, `selection`
- Column/field/filter generation from schema — `columnsFromSchema()`, `fieldsFromSchema()`
- NO custom Pinia stores for CRUD, Vuex, or manual API call management

**Chat & AI** (use ChatService):
- Multi-turn conversation — `ChatService`
- RAG-based knowledge retrieval — `ContextRetrievalHandler`
- LLM response generation — `ResponseGenerationHandler`

**Data Retention & Archival** (use ArchivalService):
- Legal hold — `LegalHoldService`
- Destruction schedules — `DestructionService`
- Retention policies — `RetentionService`

**Semantic & Hybrid Search** (use SolrController + SettingsController):
- Semantic search via vector embeddings — `SettingsController.semanticSearch()`
- Hybrid search (keyword + semantic combined) — `SolrController.hybridSearch()`
- Vector embedding generation — `VectorizationService`
- NO custom search algorithms — configure via OpenRegister settings

**GraphQL API** (use GraphQLController):
- Query objects across schemas via GraphQL — `GraphQLController.execute()`
- Alternative to REST for complex cross-entity queries

**Organization / Multi-Tenancy** (use OrganisationController):
- Organization CRUD — `OrganisationController`
- Tenant-scoped data isolation — automatic via `TenantLifecycleService`
- NO custom multi-tenancy logic

**Task & Workflow Management** (use TasksController + WorkflowEngineController):
- Task creation and tracking — `TasksController`
- Workflow orchestration — `WorkflowEngineRegistry`
- Scheduled workflows — `ScheduledWorkflowController`
- NO custom task/workflow systems

**Text Extraction** (use FileTextController):
- Extract text from PDFs and Office docs — `TextExtractionService`
- Entity recognition (PII detection) — `EntityRecognitionHandler`
- Content anonymization — automatic

**Timeline & Stages** (use CnTimelineStages):
- Workflow progression visualization — `CnTimelineStages` component
- Stage tracking with status colors

### What apps SHOULD build (custom business logic only):
- External API integrations (SAP, Peppol, TenderNed, etc.)
- PDF/document generation with business-specific templates
- Workflow triggers and business rules specific to the domain
- Notification dispatch with app-specific event types
- Custom settings pages with app-specific configuration
- Background jobs for domain-specific processing

### ADR-002-api
- URL pattern: `/index.php/apps/{app}/api/{resource}` — lowercase plural, hyphens.
- Methods: GET=read, POST=create, PUT=update, DELETE=remove. No custom methods.
- Pagination: support `_page` + `_limit`. Response includes `total`, `page`, `pages`.
- Errors: appropriate HTTP status + `message` field. NO stack traces in responses.
- Auth: Nextcloud built-in only. NO custom login/session/token flows.
- Public endpoints: annotate `#[PublicPage]` + `#[NoCSRFRequired]`. Register CORS OPTIONS route.

### ADR-003-backend
- **Controller → Service → Mapper** (strict 3-layer). Controllers NEVER call mappers directly.
- Controllers: thin (<10 lines/method). Routing + validation + response only.
- Services: ALL business logic. Stateless — no instance state between requests.
- Mappers: DB CRUD only. No business logic.
- DI: constructor injection with `private readonly`. NO `\OC::$server` or static locators.
- Entity setters: POSITIONAL args only. `$e->setName('val')` — NEVER `$e->setName(name: 'val')`.
  (`__call` passes `['name' => val]` but `setter()` uses `$args[0]`.)
- Routes: `appinfo/routes.php`. Specific routes BEFORE wildcard `{slug}` routes.
- Config: `IAppConfig` with sensitive flag for secrets. NEVER read DB directly.
- Lifecycle: schema init via repair steps (`IRepairStep`), background via job queue, events via dispatcher.
- **Spec traceability**: every class and public method MUST have `@spec` PHPDoc tag(s) linking to
  the OpenSpec change that caused it: `@spec openspec/changes/{name}/tasks.md#task-N`.
  Multiple `@spec` tags allowed (code touched by multiple changes). File-level `@spec` in header docblock.
  This enables: code → docblock → spec traceability alongside code → git blame → commit → issue → spec.

### ADR-004-frontend
- **Vue 2 + Pinia + @nextcloud/vue + @conduction/nextcloud-vue**. NO Vuex. Options API only.
- State: Pinia stores in `src/store/modules/`. Use `createObjectStore` for OpenRegister CRUD.
- `fetch()` for API calls — NOT axios. Loading state with `try/finally`.
- Translations: ALL user-visible strings via `t(appName, 'text')`. NO hardcoded strings.
- CSS: ONLY Nextcloud CSS variables. NO hardcoded colors. NEVER reference `--nldesign-*`.
- Router: history mode, base `/index.php/apps/{app}/`, catch-all `*` redirects to `/`.
- OpenRegister dependency: settings returns `openRegisters` (bool) + `isAdmin`.
  Show empty state if OR missing. NEVER use `OC.isAdmin` — get from backend.

### @conduction/nextcloud-vue — ALWAYS check before building custom

**Pages & Layout:**
  `CnIndexPage` (schema-driven list+CRUD) | `CnDetailPage` (detail+sidebar) |
  `CnPageHeader` (title+icon) | `CnActionsBar` (add+search+toggle)

**Data Display:**
  `CnDataTable` (sortable+paginated) | `CnCardGrid` + `CnObjectCard` (card views) |
  `CnDetailGrid` (label-value pairs) | `CnFilterBar` (search+filters) |
  `CnFacetSidebar` (faceted filters) | `CnPagination` | `CnCellRenderer` (type-aware)

**Forms & Dialogs:**
  `CnFormDialog` (schema-driven create/edit) | `CnAdvancedFormDialog` (properties+JSON+metadata) |
  `CnSchemaFormDialog` (JSON Schema editor) | `CnTabbedFormDialog` (tabbed form framework) |
  `CnDeleteDialog` | `CnCopyDialog`

**Mass Actions:**
  `CnMassDeleteDialog` | `CnMassCopyDialog` | `CnMassExportDialog` (CSV/JSON/XML) |
  `CnMassImportDialog` (upload+summary) | `CnMassActionBar` (floating selection bar)

**Dashboard & Widgets:**
  `CnDashboardPage` (GridStack drag-drop layout) | `CnDashboardGrid` (layout engine) |
  `CnWidgetWrapper` (widget shell) | `CnWidgetRenderer` (NC Dashboard API v1/v2) |
  `CnChartWidget` (ApexCharts: area/line/bar/pie/donut/radial) |
  `CnTableWidget` (data table widget) | `CnTileWidget` (quick-access tile) |
  `CnInfoWidget` (label-value grid) | `CnKpiGrid` (responsive KPI layout) |
  `CnStatsBlock` (metric card) | `CnStatsPanel` (stats sections) | `CnProgressBar` |
  `CnObjectDataWidget` (schema-driven editable data grid, inline edit + save via objectStore) |
  `CnObjectMetadataWidget` (read-only object metadata display)

**UI Elements:**
  `CnStatusBadge` | `CnEmptyState` | `CnIcon` (MDI) | `CnCard` | `CnDetailCard` |
  `CnRowActions` | `CnTimelineStages` (workflow progression) |
  `CnUserActionMenu` (user context menu) | `CnJsonViewer` (CodeMirror)

**Detail Sidebar:**
  `CnObjectSidebar` (Files/Notes/Tags/Tasks/Audit tabs) | `CnIndexSidebar` |
  `CnNotesCard` (inline notes) | `CnTasksCard` (inline tasks)

**Settings:**
  `CnSettingsSection` + `CnVersionInfoCard` (MUST be first on admin pages) |
  `CnSettingsCard` | `CnConfigurationCard` | `CnRegisterMapping`
  User settings: `NcAppSettingsDialog` (NOT `NcDialog`)

**Composables:**
  `useListView` (search/filter/sort/pagination) | `useDetailView` (load/edit/delete) |
  `useSubResource` (related items) | `useDashboardView` (widgets/layout/edit)

**Store Plugins:**
  `auditTrailsPlugin` | `relationsPlugin` | `filesPlugin` | `lifecyclePlugin` |
  `selectionPlugin` | `searchPlugin` | `registerMappingPlugin`

**Utilities:**
  `columnsFromSchema()` | `filtersFromSchema()` | `fieldsFromSchema()` |
  `formatValue()` | `buildHeaders()` | `buildQueryString()`

### Page Construction Patterns (follow these recipes)

**App.vue:** `NcContent` → 3 states: loading (`NcLoadingIcon`), no-OpenRegister (`NcEmptyContent`),
  ready (`MainMenu` + `NcAppContent` + `router-view` + optional `CnIndexSidebar`).
  Inject `sidebarState` for child components. `created()` calls `initializeStores()`.

**MainMenu:** `NcAppNavigation` with `NcAppNavigationItem` per route (icon + name + `:to`).
  Footer: settings link via `NcAppNavigationSettings`.

**Dashboard:** `CnDashboardPage` with `CnStatsBlock` KPIs (4 cards: open/overdue/value/completed),
  status distribution chart, "My Work" list (grouped: overdue → due this week → rest).
  Fetch all collections in parallel via `Promise.all`. Widget templates via `#widget-{id}` slots.

**Index page:** `CnIndexPage` with `useListView(entityType, { sidebarState, objectStore })`.
  Inject sidebarState. Row click → `$router.push({ name: 'EntityDetail', params: { id } })`.
  Add button → new entity detail with id='new'.

**Detail page:** Two modes — edit (form component) / view (`CnDetailPage` + `CnDetailCard` sections).
  Header actions: Edit + Delete buttons. Related entities in table inside `CnDetailCard`.
  Props: `entityId` from route. `isNew = entityId === 'new'`. Sidebar via `CnObjectSidebar`.

**Settings:** `CnVersionInfoCard` (FIRST, always) → `CnRegisterMapping` → `CnSettingsSection` per feature.
  Load settings from `GET /api/settings`. Save via `POST /api/settings`.
  Re-import button calls `POST /api/settings/load`.

**Router:** Flat routes (no nesting), all named, props via arrow function for params.
  Routes: `/` (Dashboard), `/{entities}` (list), `/{entities}/:id` (detail), `/settings`.

**Store init:** `initializeStores()` in `store/store.js` — fetches settings, then calls
  `objectStore.registerObjectType(name, schemaSlug, registerSlug)` for each entity.
  Object store uses `createObjectStore` with plugins (files, auditTrails, relations).
  Settings store: Pinia `defineStore` with `fetchSettings()` and `saveSettings()`.

### ADR-005-security
- Auth: Nextcloud built-in ONLY. NO custom login, sessions, tokens, password storage.
- Admin check: `IGroupManager::isAdmin()` on BACKEND. Frontend-only checks = vulnerability.
- Multi-tenant isolation: enforce at API/service level, not UI only.
- NO PII in logs, error responses, or debug output.
- File uploads: validate type + size before storage.
- API responses: NO stack traces, SQL, or internal paths.

### ADR-006-metrics
- Every app: `GET /api/metrics` (Prometheus text, admin auth) + `GET /api/health` (JSON, public).
- Metric names: `{app}_` prefix. MUST include `{app}_health_status` and `{app}_info`.
- Health check MUST verify OpenRegister connectivity (for apps that depend on it).

### ADR-007-i18n
- Minimum: Dutch (nl) + English (en) translations.
- PHP: `$this->l->t('key')`. JS: `t(appName, 'key')`.
- API field names: English. Date/number formatting: respect user locale.
- Each app with OpenRegister: define `register-i18n` spec listing translatable fields.

### ADR-008-testing
- Every new PHP service/controller → PHPUnit tests in `tests/Unit/` (≥3 methods).
- Every new Vue component → test file (if test framework exists).
- Every new API endpoint → Newman/Postman collection in `tests/integration/`.
- Every spec scenario → browser test (GIVEN/WHEN/THEN verified via Playwright).
- All tests MUST pass in `composer check:strict`.

### ADR-009-docs
- Every user-facing feature → docs in `docs/` with screenshots from running app.
- English primary, Dutch recommended. Update docs when behavior changes.

### ADR-010-nl-design
- ALL UI: CSS custom properties from NL Design System tokens. NO hardcoded colors, fonts, spacing.
- Theme switching: support `nldesign` app's token sets (Rijkshuisstijl, Utrecht, municipality-specific).
- Components: `@nextcloud/vue` primary. Custom components styled via NL Design tokens only.
- Scoped styles: ALL `<style>` blocks MUST use `scoped` attribute.
- WCAG AA mandatory: keyboard-navigable, labelled forms, color not sole conveyor, alt text on images.
- Responsive: work from 320px to 1920px. Critical features accessible at 768px.
- Specs: reference token names ("primary action color") NOT hex values. Include a11y verification in ACs.
- Exception: PDF generation (docudesk) may use fixed dimensions. Admin screens MAY simplify but MUST meet WCAG AA.

### ADR-011-schema-standards
- schema.org types/properties as primary vocabulary (`schema:Person`, `schema:Organization`, `schema:Event`).
- Contact schemas: align with vCard properties (`fn`, `email`, `tel`, `adr`).
- Dutch government fields: mapping layer translating between international standards and Dutch APIs (VNG, ZGW).
- NO custom property names when schema.org equivalent exists.
- Relations: OpenRegister relation mechanism (register + schema + objectId). NO foreign keys or embedded objects.
- Versioning: removing/renaming properties = BREAKING → migration via repair step. Adding optional = non-breaking.
- Specs MUST define data models using schema.org vocabulary; design docs MUST include schema definitions with types, required flags, relations.
- Exception: app-specific workflow states (pipeline stages, process statuses) MAY use custom vocabularies.

### ADR-012-deduplication
- Before proposing new capability: search OpenRegister specs + services for overlap. Reference + justify if similar exists.
- Design docs MUST include "Reuse Analysis" listing which OpenRegister services are leveraged.
- If logic could benefit other apps → propose adding to OpenRegister core, not app-specific.
- Tasks MUST include "Deduplication Check" verifying no overlap with:
  ObjectService, RegisterService, SchemaService, ConfigurationService, shared specs, @conduction/nextcloud-vue.
- Document findings even if "no overlap found".
- Exception: OpenRegister checks internal duplication only. nldesign checks token sets. nextcloud-vue checks own components.

### ADR-013-container-pool
# ADR-013: Unified Container Pool

**Status:** accepted
**Date:** 2026-04-12

## Context

Specter (intelligence/research) and Hydra (build/review/merge) both run LLM workloads in Docker containers. Today they operate independently: Hydra spins up builder/reviewer/security containers on demand, Specter has a separate `run_llm_containers.sh` wrapper. Both compete for the same Claude Max rate limits.

We want to unify these into a **single priority-scheduled container pool** so that:
- Critical work (bugfixes, reviews) preempts lower-priority work (discovery, research)
- A fixed number of containers (e.g. 10) run continuously, pulling from a shared queue
- Token rotation and rate limit recovery happen at the pool level, not per-script
- Adding a new workload type (audit, spec generation, test) is just a new queue entry

## Decision

### Container types (priority order)

| Priority | Type | Source | Container image | Model |
|----------|------|--------|-----------------|-------|
| 1 | **bugfix** | Hydra: fix iteration after review failure | `hydra-builder` | opus |
| 2 | **code-review** | Hydra: PR code review | `hydra-reviewer` | sonnet |
| 3 | **security-review** | Hydra: PR security review | `hydra-security` | sonnet |
| 4 | **build** | Hydra: initial spec build | `hydra-builder` | opus |
| 5 | **audit** | Hydra: codebase audit | `hydra-builder` | sonnet |
| 6 | **spec-generation** | Specter: push_spec_pipeline | `specter-llm-worker` | sonnet |
| 7 | **schema-synthesis** | Specter: generate/dedup schemas | `specter-llm-worker` | haiku |
| 8 | **classification** | Specter: classify/redistribute features | `specter-llm-worker` | haiku |
| 9 | **translation** | Specter: translate requirements | `specter-llm-worker` | haiku |
| 10 | **discovery** | Specter: research, feature extraction | `specter-llm-worker` | haiku |

### Architecture

```
┌─────────────────────────────────────────────────────┐
│  Scheduler (cron or daemon)                         │
│                                                     │
│  reads: queue table (postgres)                      │
│  writes: container assignments, status updates      │
│                                                     │
│  ┌──────────────────────────────────────────┐       │
│  │ Pool: 10 container slots                 │       │
│  │                                          │       │
│  │  slot-1: [bugfix]     ← highest prio     │       │
│  │  slot-2: [code-review]                   │       │
│  │  slot-3: [build]                         │       │
│  │  slot-4: [build]                         │       │
│  │  slot-5: [classify]                      │       │
│  │  slot-6: [classify]                      │       │
│  │  slot-7: [translate]                     │       │
│  │  slot-8: [discovery]                     │       │
│  │  slot-9: [idle]       ← waiting for work │       │
│  │  slot-10: [idle]                         │       │
│  └──────────────────────────────────────────┘       │
│                                                     │
│  Token rotation: credentials.json (work → private)  │
│  Rate limit: pool-level tracking per account        │
│  Preemption: low-prio containers stopped when       │
│              high-prio work arrives and pool is full │
└─────────────────────────────────────────────────────┘
```

### Queue table (future)

```sql
CREATE TABLE container_queue (
    id SERIAL PRIMARY KEY,
    type VARCHAR(50) NOT NULL,        -- bugfix, code-review, build, classify, etc.
    priority INTEGER NOT NULL,         -- 1=highest
    payload JSONB NOT NULL,            -- script args, spec slug, issue URL, etc.
    status VARCHAR(20) DEFAULT 'pending', -- pending, running, completed, failed
    container_id VARCHAR(100),         -- docker container name when running
    token_account VARCHAR(50),         -- which OAuth account is assigned
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    exit_code INTEGER,
    error_message TEXT
);
```

### Phased rollout

**Phase 1 (now):** All LLM calls containerized. Specter scripts run via `run_llm_containers.sh`. Hydra containers use `run_container_with_fallback`. Both read from `credentials.json`. No shared queue yet — each system schedules its own containers.

**Phase 2:** Shared queue table. A single scheduler script replaces both `cron-hydra.sh` dispatch and `run_llm_containers.sh`. Pool size configurable. Priority enforcement by not starting low-prio work when high-prio is queued.

**Phase 3:** Preemption. Running low-priority containers can be stopped (gracefully, with checkpoint) when high-priority work arrives and all slots are occupied. Container images support checkpoint/resume via DB state.

### Current state (Phase 1)

Both systems already containerize LLM calls:
- **Hydra:** `builder`, `reviewer`, `security` images in `hydra/images/`
- **Specter:** `specter-llm-worker` image via `Dockerfile.llm-worker`
- **Shared credentials:** `hydra/secrets/credentials.json` with priority-ordered OAuth tokens
- **Token fallback:** Hydra via `credentials.sh`, Specter via `credentials.py`

## Consequences

- All LLM calls go through containers — no direct `claude -p` from host scripts
- Token management is centralized in `credentials.json`
- Future pool scheduler can enforce rate limits across both systems
- Container images are the unit of deployment — version, test, rollback independently
