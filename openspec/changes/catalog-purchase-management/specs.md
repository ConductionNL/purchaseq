# Specs: Catalog & Purchase Management — Shillinq

Requirements are derived from the user stories and feature set in `context-brief.md`.
Format: `REQ-{DOMAIN}-{NNN}` — each requirement has a rationale and one or more GIVEN/WHEN/THEN scenarios.

---

## REQ-CAT: Procurement Catalog management

### REQ-CAT-001 — Create and activate a procurement catalog

**Rationale:** Category managers need to define named catalogs scoped to an organization and validity period before adding items.

**Scenario 1 — Create catalog in draft:**
```
GIVEN I am logged in as a category manager
WHEN I open the Catalogs list and click Add
THEN a CnFormDialog opens with fields: catalogNumber, catalogName, description, catalogFormat, status, validFrom, validUntil
AND the new catalog is saved with status = draft
```

**Scenario 2 — Activate draft catalog:**
```
GIVEN a ProcurementCatalog with status = draft exists
WHEN I set status = active and save
THEN the catalog appears in the active catalog list
AND a ProcurementAuditLog entry is written with actionType = updated and the status change recorded in changes
```

---

### REQ-CAT-002 — Bulk import catalog items with validation

**Rationale:** Catalog content management with bulk import is the second-highest demand feature (demand 1621). Category managers must be able to upload CSV or cXML files containing hundreds of items without manual entry.

**Scenario 1 — Upload CSV catalog items:**
```
GIVEN a ProcurementCatalog is active
WHEN I click Import on the catalog detail page and upload a CSV file with columns: itemCode, itemName, basePrice, unit, status
THEN the CnMassImportDialog opens showing a field mapping preview and row count
AND after confirmation, all valid rows are saved as CatalogItem objects linked to the catalog
AND rows failing validation (missing required fields, invalid price) are listed in an error summary
```

**Scenario 2 — Duplicate item codes rejected:**
```
GIVEN a CatalogItem with itemCode = IT-LAPTOP-001 already exists in catalog CAT-2025-001
WHEN a bulk import includes a row with the same itemCode
THEN the duplicate row is rejected with message "Itemcode IT-LAPTOP-001 bestaat al in catalogus CAT-2025-001"
AND the remaining valid rows are imported successfully
```

---

### REQ-CAT-003 — Bulk price update from contract renewal

**Rationale:** Story 2 — When a contract is renegotiated, all linked catalog items must show old and new prices for review before bulk update is applied.

**Scenario 1 — Initiate bulk price update:**
```
GIVEN a contract has been renegotiated and new prices are known
WHEN a category manager selects affected CatalogItems and initiates a price update
THEN a review dialog lists each item with its current basePrice and the proposed new basePrice
AND items are not updated until the category manager confirms
```

**Scenario 2 — Apply bulk price update:**
```
GIVEN I have reviewed and confirmed the bulk price update
WHEN the update is applied
THEN each affected CatalogItem basePrice is updated to the new value
AND a ProcurementAuditLog entry is written per item with actionType = updated, changes.basePrice.old and changes.basePrice.new recorded
AND the update reason references the contract number
```

**Scenario 3 — Historical order prices preserved:**
```
GIVEN CatalogItem basePrice has been updated via bulk price update
WHEN I view a PurchaseRequisition created before the update
THEN the line item price shown matches the price at the time of requisition creation
AND the new basePrice applies only to new requisitions created after the update
```

---

### REQ-CAT-004 — Deactivate catalog items linked to expiring contracts

**Rationale:** Story 3 — Requisitioners must not be able to order from expired contracts.

**Scenario 1 — 60-day expiry notification:**
```
GIVEN a CatalogItem has validUntil set and the expiry date is 60 or fewer days away
WHEN the ContractExpiryNotificationJob runs
THEN the category manager receives a Nextcloud notification listing the affected catalog items and their expiry dates
AND a ProcurementAuditLog entry is written with actionType = updated and reason = "Automatisch signaal: verloopdatum nadert"
```

**Scenario 2 — Deactivated items hidden from search:**
```
GIVEN a CatalogItem has status = discontinued
WHEN a requisitioner searches the catalog
THEN discontinued items do not appear in search results
AND the category manager can still find them via the Catalog Item list with status filter = discontinued
```

---

### REQ-CAT-005 — Catalog item usage statistics

**Rationale:** Story 5 — Category managers need to rationalize the catalog by identifying high-use and unused items.

**Scenario 1 — View usage statistics for a date range:**
```
GIVEN I open a CatalogItem detail page
WHEN I select a date range and open the Usage Statistics tab
THEN I see: total orders placed, total quantity ordered, total spend, number of unique requisitioners
AND statistics are scoped to the selected date range
```

**Scenario 2 — Bulk deactivate unused items:**
```
GIVEN I view the Unused Items report (items with zero orders in 12 months)
WHEN I select multiple items and click Deactivate
THEN all selected items have their status changed to discontinued
AND ProcurementAuditLog entries are written for each with reason = "Gedeactiveerd via ongebruikte artikelen rapport"
```

---

### REQ-CAT-006 — User-specific catalog views by role and department

**Rationale:** Feature 7 (demand 404) — Different roles and departments see only relevant catalog items.

**Scenario 1 — Department-filtered catalog view:**
```
GIVEN a CatalogItem is configured with allowed organizations or roles
WHEN a requisitioner in a department without access searches the catalog
THEN that catalog item does not appear in their search results
AND an administrator can configure visibility rules via the Settings page using AuthorizationService
```

---

## REQ-PRQ: Purchase Requisition workflow

### REQ-PRQ-001 — Create a purchase requisition

**Rationale:** Central workflow artifact — a requisitioner submits a formal purchase request that triggers approval and order creation.

**Scenario 1 — Create draft requisition:**
```
GIVEN I am logged in as a requisitioner
WHEN I navigate to Purchase Requisitions and click Add
THEN a CnFormDialog opens with fields: purpose, deliveryDate, customFields
AND a requisitionNumber is auto-generated
AND status is set to draft
AND my user identity is recorded as requester (via IUserSession — not frontend-sent user ID)
```

**Scenario 2 — Submit requisition for approval:**
```
GIVEN a PurchaseRequisition with status = draft exists and has at least one line item linked via catalog
WHEN I click Submit
THEN status changes to submitted
AND an ApprovalRequest is created and routed via the WorkflowEngineController
AND a ProcurementAuditLog entry is written with actionType = updated, changes.status = draft → submitted
```

---

### REQ-PRQ-002 — Approval workflow for requisitions

**Rationale:** Feature 3 (demand 1046) — Purchase requisitions must pass through configurable approval steps.

**Scenario 1 — Approve requisition:**
```
GIVEN a PurchaseRequisition with status = submitted has an open ApprovalRequest
WHEN the approver approves the request
THEN PurchaseRequisition status changes to approved
AND a ProcurementAuditLog entry is written with actionType = approved and the approver's identity
AND a Nextcloud notification is sent to the requester
```

**Scenario 2 — Reject requisition with reason:**
```
GIVEN a PurchaseRequisition with status = submitted is under review
WHEN the approver rejects it with a reason
THEN PurchaseRequisition status changes to rejected
AND a ProcurementAuditLog entry is written with actionType = rejected and the rejection reason
AND the requester receives a notification with the reason
```

**Scenario 3 — Requisition timeline visible:**
```
GIVEN a PurchaseRequisition exists
WHEN I open its detail page
THEN a CnTimelineStages component shows the stages: Draft → Submitted → Approved → Ordered
AND the current stage is highlighted with the correct status color
AND past stages show completion timestamps
```

---

### REQ-PRQ-003 — Detect off-contract (maverick) spend

**Rationale:** Stories 8–10 — Purchases outside contracted channels represent compliance risk and budget waste.

**Scenario 1 — Identify maverick spend:**
```
GIVEN PurchaseOrders exist without a contract reference
WHEN the MaverickSpendDetectionJob runs
THEN each unlinked order is cross-referenced by supplier and category against active Contracts
AND if a matching Contract exists, the order is flagged as maverick spend with the contract that should have been used
AND a MaverickSpendAlert object is created (existing entity, referenced from context-brief other entities)
```

**Scenario 2 — Maverick spend dashboard by department:**
```
GIVEN maverick spend has been identified
WHEN I open the Maverick Spend dashboard
THEN each department shows its maverick spend as a euro amount and percentage of total spend
AND I can drill down to see individual orders and requisitioners driving the maverick spend
```

**Scenario 3 — Monthly maverick spend email alert:**
```
GIVEN the monthly maverick report is generated on the 1st of the month
WHEN the department's maverick spend exceeds the configured threshold
THEN the department manager receives an email notification with their department's summary and a deep link to the detail view
AND the manager can see the specific orders via the link
```

---

## REQ-BPO: Blanket Purchase Orders

### REQ-BPO-001 — Create a blanket purchase order with spend limit

**Rationale:** Blanket POs authorize recurring spending within defined limits, preventing over-commitment.

**Scenario 1 — Create blanket PO:**
```
GIVEN I am a procurement officer with the appropriate mandate
WHEN I create a BlanketPurchaseOrder with totalAuthorizedAmount, validFrom, validUntil, and an optional releaseSchedule
THEN the BPO is saved with status = active
AND consumedAmount is initialized to 0 and remainingAmount equals totalAuthorizedAmount
AND an ApprovalRequest is routed for high-value BPOs above the configured threshold
```

**Scenario 2 — Consumption tracking:**
```
GIVEN a BlanketPurchaseOrder has linked PurchaseOrders
WHEN a new PurchaseOrder is created against the blanket PO
THEN consumedAmount is incremented by the PO value
AND remainingAmount is recalculated
AND a progress bar on the BPO detail page shows consumption as a percentage
```

**Scenario 3 — Block spend exceeding authorized limit:**
```
GIVEN a BlanketPurchaseOrder has remainingAmount < the new PurchaseOrder total
WHEN a PurchaseOrder is submitted against the BPO
THEN the system rejects the PO with message "Bedrag overschrijdt resterende machtiging op BPO {blanketPoNumber}"
AND a ProcurementAuditLog entry is written with actionType = rejected and reason = "BPO limiet overschreden"
```

---

### REQ-BPO-002 — Release schedule management

**Rationale:** Procurement advisors plan purchases in scheduled tranches tied to budget periods.

**Scenario 1 — Add release schedule:**
```
GIVEN a BlanketPurchaseOrder is being created or edited
WHEN I add release lines with releaseDate, amount, and description
THEN the release schedule is stored in the releaseSchedule array
AND a table on the BPO detail page shows each scheduled release with its date, amount, and status (upcoming/released)
```

---

## REQ-SOW: Statement of Work

### REQ-SOW-001 — Create a statement of work with milestones

**Rationale:** Service procurement requires tracking deliverables and milestone-triggered payments.

**Scenario 1 — Create SOW with deliverables:**
```
GIVEN I am creating a StatementOfWork
WHEN I add deliverables (title, description, dueDate, acceptanceCriteria) and milestones (title, completionDate, invoiceAmount, invoiceTrigger)
THEN all deliverables and milestones are stored in the respective arrays
AND the detail page shows deliverables and milestones in separate CnDetailCard sections
```

**Scenario 2 — Milestone triggers PurchaseOrder creation:**
```
GIVEN a StatementOfWork milestone has a completionDate and invoiceAmount
WHEN the milestone is marked as completed
THEN a PurchaseOrder is created (or suggested) for the milestone invoiceAmount linked to the SOW
AND a notification is sent to the financial controller for payment approval
```

---

## REQ-AUD: Procurement Audit Trail

### REQ-AUD-001 — Immutable audit log for all procurement actions

**Rationale:** Feature 1 (demand 2009) — Public sector transparency requires a full audit trail for all procurement decisions. This is the highest-demand feature.

**Scenario 1 — Audit log entry on every status change:**
```
GIVEN any procurement entity (PurchaseRequisition, BlanketPurchaseOrder, CatalogItem, StatementOfWork) changes status
WHEN the change is saved
THEN a ProcurementAuditLog entry is automatically written with: auditId, entityType, entityId, actionType, timestamp (server time), and the authenticated user as actor
AND the entry cannot be deleted or modified via the UI
```

**Scenario 2 — Audit log search and filter:**
```
GIVEN the Audit Log list page is open
WHEN I filter by entityType = requisition and date range 2025-09-01 to 2025-09-30
THEN only matching log entries are displayed
AND I can export the filtered result as CSV for regulatory submission
```

**Scenario 3 — GDPR compliance — no PII in logs:**
```
GIVEN a ProcurementAuditLog entry records an approved action
WHEN the entry is created
THEN the actor field stores only the Nextcloud user UID (immutable identifier)
AND display names (mutable) are never stored in the audit log
AND audit log entries are not included in error responses or log files containing stack traces
```

---

### REQ-AUD-002 — SOC 2 and GDPR compliance audit evidence

**Rationale:** Feature 13 (demand 184) — The audit trail must support export and evidence generation for compliance audits.

**Scenario 1 — Export audit trail for a date range:**
```
GIVEN an auditor requests evidence for a fiscal year
WHEN I open the Audit Log, apply a date range filter, and click Export
THEN a CSV export is generated via ExportService containing all log entries in the selected period
AND the export includes: auditId, entityType, entityId, actionType, timestamp, reason, actor UID
```

---

## REQ-PRC: Pricing Rules

### REQ-PRC-001 — Define volume discount pricing rules

**Rationale:** Volume discounts are the most common pricing structure in procurement contracts.

**Scenario 1 — Create volume discount rule:**
```
GIVEN a CatalogItem exists
WHEN I add a PricingRule with ruleType = volumeDiscount, minQuantity = 10, discountPercentage = 5
THEN the rule is linked to the catalog item
AND the rule detail shows: minimum quantity, discount percentage, validity period
```

**Scenario 2 — Multiple rules resolve by priority:**
```
GIVEN a CatalogItem has two PricingRules with different priorities both applicable to a given quantity
WHEN the applicable price is calculated
THEN the rule with the lower priority number (higher priority) is applied first
AND overlapping rules are documented in the rule description
```

---

## REQ-INT: Integrations

### REQ-INT-001 — DigiInkoop electronic ordering

**Rationale:** Feature 5 (demand 500) — Dutch government organizations require DigiInkoop-compatible electronic purchase orders.

**Scenario 1 — Send approved requisition to DigiInkoop:**
```
GIVEN a PurchaseRequisition has status = approved
WHEN the DigiInkoop integration is enabled in Settings and the requisition is dispatched
THEN the DigiInkooopOrderService translates the requisition to a UBL 2.1 electronic order message
AND the message is sent to the configured DigiInkoop API endpoint
AND the dispatch result (success/failure) is recorded in a ProcurementAuditLog entry with actionType = posted
```

**Scenario 2 — DigiInkoop connection configuration:**
```
GIVEN I am an administrator on the Settings page
WHEN I configure the DigiInkoop endpoint URL, API key, and sender OIN
THEN the credentials are saved via IAppConfig with the sensitive flag set
AND a Test Connection button verifies the endpoint responds correctly
```

---

### REQ-INT-002 — OCI punchout catalog integration

**Rationale:** Feature 12 (demand 188) — Suppliers with OCI punchout endpoints allow requisitioners to browse live catalog data without manual maintenance.

**Scenario 1 — Configure punchout endpoint:**
```
GIVEN a supplier provides an OCI punchout URL and credentials
WHEN a category manager configures the OCI punchout connection in the catalog settings
THEN requisitioners can initiate a punchout session from the Catalog list
```

**Scenario 2 — Import punchout basket:**
```
GIVEN a requisitioner has selected items in the supplier's OCI punchout catalog
WHEN the supplier returns the basket via the CXML_TO hook
THEN a PurchaseRequisition is created with pre-filled CatalogItem data (item name, price, unit) from the supplier response
AND the requisitioner sees the pre-filled requisition for review before submitting
```

---

## REQ-DSH: Dashboard and analytics

### REQ-DSH-001 — Procurement spend dashboard

**Rationale:** Feature 4 (demand 1025) — Real-time procurement visibility is required by procurement officers and financial controllers.

**Scenario 1 — Dashboard KPI blocks:**
```
GIVEN I open the Shillinq procurement dashboard
WHEN the page loads
THEN I see four CnStatsBlock KPI cards:
  - Open Purchase Requisitions (count, status = submitted)
  - Pending Approvals (count)
  - Active Catalogs (count, status = active)
  - Maverick Spend % (percentage of total spend flagged as maverick, current fiscal year)
AND data is fetched in parallel from the relevant object stores
```

**Scenario 2 — Spend trend chart:**
```
GIVEN the dashboard is loaded
WHEN I view the spend trend widget
THEN a CnChartWidget (bar or area type) shows monthly spend totals for the last 12 months
AND I can click a bar to drill through to the individual PurchaseRequisitions for that month
```

---

### REQ-DSH-002 — Contract budget dashboard

**Rationale:** Story 12 — Financial controllers monitor committed and actual spend per active contract.

**Scenario 1 — Contract budget overview:**
```
GIVEN active Contracts exist with linked PurchaseOrders and Invoices
WHEN I open the Contract Budget dashboard
THEN each contract shows: contracted value, total POs raised, total invoices approved, remaining budget as a percentage
AND I can filter by department or cost centre and the dashboard updates to show only matching contracts
```

---

### REQ-DSH-003 — Spend cube by category, supplier, and department

**Rationale:** Story 6 — Procurement data analysts use spend cubes to identify consolidation opportunities and negotiate better contracts.

**Scenario 1 — Generate spend cube:**
```
GIVEN I am logged in as a procurement data analyst
WHEN I select a fiscal year and generate the spend cube
THEN total spend is displayed across three dimensions: CPV category, supplier (Organization), and organizational unit
AND I can drill down on any cell to see the individual PurchaseOrders and Contracts making up that cell's value
```

---

### REQ-DSH-004 — CPV code classification

**Rationale:** Story 7 — Unclassified spend lines must be automatically suggested a CPV code.

**Scenario 1 — Suggest CPV code for unclassified spend:**
```
GIVEN PurchaseOrders exist without a CPV code
WHEN the CPV classification job runs (using VectorizationService semantic similarity against CPV vocabulary)
THEN each unclassified line receives a suggested CPV code and a confidence score (0–100%)
AND the suggestion is presented for review in the Spend Analysis view
```

**Scenario 2 — Accept, edit, or reject CPV suggestion:**
```
GIVEN a CPV code suggestion is shown for a PurchaseOrder line
WHEN I review the suggestion
THEN I can accept it (CPV code is applied), edit it (select a different code from the CPV hierarchy), or reject it (line remains unclassified with a note)
AND my decision is recorded in the audit trail
```

---

## REQ-FRM: Framework agreements and call-off orders

### REQ-FRM-001 — Assign admitted suppliers to framework lots

**Rationale:** Story 15 — Call-off orders must be validated against the eligible supplier list per lot.

**Scenario 1 — Add supplier to lot:**
```
GIVEN a FrameworkAgreement has been created with Lots (existing entities referenced from context-brief)
WHEN I add a Supplier to a Lot's admitted suppliers list
THEN the supplier appears in the lot's admitted suppliers list
```

**Scenario 2 — Block call-off from non-admitted supplier:**
```
GIVEN a CallOffOrder is being created against a Lot
WHEN the selected supplier is not in the admitted suppliers list for that Lot
THEN the system blocks the call-off and shows error message "Leverancier niet toegelaten voor perceel {lotNumber}"
AND the block is recorded in the audit trail
```

---

### REQ-FRM-002 — Framework lot budget ceiling

**Rationale:** Story 16 — Cumulative call-off order values must not exceed the published framework ceiling.

**Scenario 1 — Set budget ceiling on lot:**
```
GIVEN a Lot exists within a FrameworkAgreement
WHEN I set a budget ceiling on the Lot
THEN all CallOffOrders linked to that Lot are summed and compared against the ceiling in real time
```

**Scenario 2 — Ceiling breach warning:**
```
GIVEN a Lot has a budget ceiling set
WHEN a new CallOffOrder is submitted that would cause the cumulative total to exceed the ceiling
THEN the system displays a warning: "Afroep overschrijdt plafond perceel {lotNumber}. Expliciete rechtvaardiging vereist."
AND the procurement advisor must provide a justification to proceed
AND the breach and justification are recorded in the audit trail
```

---

### REQ-FRM-003 — Framework lot expiry and utilisation dashboard

**Rationale:** Story 17 — Procurement advisors plan purchases around lot expiry dates.

**Scenario 1 — Lot dashboard view:**
```
GIVEN a FrameworkAgreement is active
WHEN I open the Framework Dashboard
THEN each Lot shows: consumed budget, remaining budget, utilisation percentage, and days until expiry
```

**Scenario 2 — Expiry warning badge:**
```
GIVEN a FrameworkAgreement Lot has an expiry date within 60 calendar days
WHEN I view the Framework Dashboard
THEN the Lot displays an expiry warning badge with the number of remaining days
```

---

### REQ-FRM-004 — Call-off delivery schedule

**Rationale:** Story 18 — Partial deliveries on call-off orders require a structured delivery schedule.

**Scenario 1 — Add delivery lines to call-off:**
```
GIVEN a CallOffOrder is being created
WHEN I add delivery lines with: quantity, deliveryDate, and deliveryAddress
THEN the delivery schedule is stored against the call-off
AND the detail page shows the delivery schedule table
```

**Scenario 2 — Delivery date reminder:**
```
GIVEN a CallOffOrder delivery line has a deliveryDate set
WHEN the deliveryDate arrives
THEN the contract manager receives a Nextcloud notification to confirm receipt
```

---

### REQ-FRM-005 — Call-off history per supplier and lot

**Rationale:** Story 19 — Procurement advisors assess spend concentration and dependency risk per supplier.

**Scenario 1 — Filter call-off history by supplier:**
```
GIVEN CallOffOrders exist for a FrameworkAgreement
WHEN I filter by a specific supplier
THEN all orders placed with that supplier are listed with: value, date, and status
```

**Scenario 2 — Call-off order detail:**
```
GIVEN I am viewing the call-off history filtered by supplier
WHEN I click on a CallOffOrder
THEN I see the full order detail including delivery schedule and payment status
```

---

## REQ-SUP: Supplier management

### REQ-SUP-001 — Supplier offboarding with dependency check

**Rationale:** Story 11 — Suppliers must not be deactivated while open dependencies exist.

**Scenario 1 — Initiate offboarding:**
```
GIVEN an active Supplier record exists
WHEN I initiate offboarding
THEN the system checks for: open PurchaseOrders, active Contracts, and pending Invoices linked to the supplier
AND presents a dependency report listing each open item with its type, number, and status
```

**Scenario 2 — Block deactivation with open items:**
```
GIVEN the dependency report shows open items
WHEN I attempt to deactivate the supplier without resolving them
THEN the system prevents deactivation
AND I must either resolve each open item or explicitly override with a written justification
AND the override and justification are recorded in the audit trail
```

---

## REQ-SETTINGS: Settings and configuration

### REQ-SET-001 — Settings page structure

**Rationale:** ADR-004 — Settings page must follow the standard pattern: CnVersionInfoCard first.

**Scenario 1 — Settings page layout:**
```
GIVEN I navigate to the Settings page as an administrator
WHEN the page loads
THEN CnVersionInfoCard is shown first with current app version and OpenRegister status
AND CnRegisterMapping shows the shillinq register configuration
AND separate CnSettingsSection cards appear for: DigiInkoop integration, OCI punchout connections, Maverick spend threshold, and notification settings
```

---

## Accessibility requirements (ADR-010)

All pages and dialogs MUST:
- Be keyboard-navigable (Tab, Enter, Escape, arrow keys for lists and dialogs)
- Have all form fields labelled (NcSelect requires `inputLabel`)
- Use only Nextcloud CSS custom properties — no hardcoded colors
- Meet WCAG AA color contrast on status badges (`CnStatusBadge`)
- Include `alt` text on any non-decorative images
- Work at 320px viewport width for critical flows (create requisition, view audit log)
