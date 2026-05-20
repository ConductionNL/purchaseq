# Specs: Catalog & Purchase Management — Shillinq — Ai

**Change:** catalog-purchase-management-ai
**Date:** 2026-05-20

---

## Requirement Domains

| Domain | Prefix | Features covered |
|--------|--------|-----------------|
| Catalog Management | REQ-CAT | Features 1, 4, 15, 18 |
| Purchase Order Management | REQ-POM | Features 3, 9, 19, 20, 21, 22, 23 |
| AI Agents | REQ-AIA | Features 5, 7, 11, 16 |
| Spend Intelligence | REQ-SPD | Features 2, 6, 8, 10, 12, 13, 17 |
| Request Creation | REQ-REQ | Feature 14 |

---

## REQ-CAT — Catalog Management

### REQ-CAT-001: Automated catalog item validation on upload

**Feature:** Catalog content management with automated validation and update workflow (demand 1760)

The system SHALL validate all CatalogItem fields upon upload or import and surface
validation errors before items are published to ProcurementCatalog.

**Validation rules:**
- `name` is required (non-empty string, max 255 chars)
- `sku` is required and unique within the ProcurementCatalog
- `unitPrice.amount` must be a positive number
- `vatRate` must be one of `[0, 9, 21]` (Dutch BTW rates)
- `procurementCategory` must match an existing ProcurementCategory slug
- `unitOfMeasure` is required

**Scenario 1 — Valid item accepted:**
```gherkin
GIVEN a CatalogItem with all required fields populated
  AND vatRate is 21
  AND sku "OF-PAP-0080" does not exist in the target ProcurementCatalog
WHEN the validation service processes the item
THEN the item status is "valid"
  AND no validation errors are returned
```

**Scenario 2 — Missing required field rejected:**
```gherkin
GIVEN a CatalogItem where unitPrice.amount is absent
WHEN the validation service processes the item
THEN the item status is "invalid"
  AND the error list contains "unitPrice.amount is required"
  AND the item is NOT staged for import
```

**Scenario 3 — Duplicate SKU flagged:**
```gherkin
GIVEN a ProcurementCatalog containing CatalogItem with sku "OF-PAP-0080"
  AND a new import batch contains an item with sku "OF-PAP-0080"
WHEN the validation service processes the batch
THEN the duplicate item is flagged with error "Duplicate SKU: OF-PAP-0080"
  AND a suggestion to merge or replace is included in the response
```

---

### REQ-CAT-002: CIF/cXML catalog upload and managed publishing

**Feature:** CIF/cXML catalog upload and management with automated validation and publishing (demand 1594)

The system SHALL accept CIF 3.0 and cXML 1.2 formatted catalog files, parse them into
CatalogItem objects, validate them, and allow staged publishing to ProcurementCatalog.

**Scenario 1 — Successful CIF import:**
```gherkin
GIVEN a valid CIF 3.0 file containing 50 catalog items
  AND 48 items pass validation
  AND 2 items fail validation (missing vatRate)
WHEN the Inkoopmedewerker submits the file via the import endpoint
THEN 48 items are staged with status "valid"
  AND 2 items are returned with status "invalid" and field-level errors
  AND the response includes staged count and invalid count
  AND no items are published until explicit approval
```

**Scenario 2 — Dry-run mode:**
```gherkin
GIVEN a CIF 3.0 file
  AND the request parameter dryRun is true
WHEN the import endpoint processes the file
THEN validation results are returned
  AND NO CatalogItem objects are created or modified in OpenRegister
```

**Scenario 3 — Re-import idempotency:**
```gherkin
GIVEN a CIF 3.0 file with slug "cat-item-printpapier-002" already imported
WHEN the same file is imported again without changes
THEN the existing CatalogItem "cat-item-printpapier-002" is NOT duplicated
  AND the response status indicates "skipped: already exists"
```

**Scenario 4 — Unsupported file format rejected:**
```gherkin
GIVEN a file with MIME type "application/vnd.ms-excel"
WHEN submitted to the CIF/cXML import endpoint
THEN HTTP 422 is returned
  AND the error message is "Unsupported file format. Expected CIF 3.0 or cXML 1.2."
```

---

### REQ-CAT-003: AI-powered catalog search with natural language queries

**Feature:** AI-powered catalog search with natural language queries and smart recommendations (demand 173)

The system SHALL support natural-language search queries against ProcurementCatalog items
using semantic embeddings provided by `VectorizationService`.

**Scenario 1 — Natural language query returns relevant results:**
```gherkin
GIVEN a ProcurementCatalog containing CatalogItems including "Bureaustoelen ergonomisch NEN 1813"
WHEN a user submits the search query "comfortabele stoel voor kantoor"
THEN the result list includes "Bureaustoelen ergonomisch NEN 1813"
  AND results are ordered by semantic relevance score descending
```

**Scenario 2 — No results returns empty state:**
```gherkin
GIVEN a ProcurementCatalog with no items matching "drone onderdelen"
WHEN a user submits the search query "drone onderdelen"
THEN an empty result set is returned
  AND the response includes a "no results" indicator
```

---

### REQ-CAT-004: Smart catalog item recommendations based on purchase history

**Feature:** Smart catalog with AI-driven product recommendations based on purchase history (demand 110)

When a user views or selects a CatalogItem, the system SHALL surface alternative or
complementary items based on co-purchase patterns in historical SpendTransaction records.

**Scenario 1 — Recommendations returned for known item:**
```gherkin
GIVEN CatalogItem "cat-item-printpapier-002" has been co-purchased with
      "cat-item-balpennen-005" in at least 3 SpendTransaction records
WHEN a user views CatalogItem "cat-item-printpapier-002"
THEN the recommendations panel includes "cat-item-balpennen-005"
  AND the recommendation reason is "Vaak samen besteld"
```

**Scenario 2 — No history returns no recommendations:**
```gherkin
GIVEN a CatalogItem with no associated SpendTransaction records
WHEN a user views the item
THEN the recommendations panel is empty
  AND no error is shown
```

---

## REQ-POM — Purchase Order Management

### REQ-POM-001: OCR-based field extraction from uploaded PO documents

**Feature:** Purchase Order Management with AI-Based OCR (demand 1649)

When a PDF file is attached to a PurchaseOrder record, the system SHALL invoke
`TextExtractionService` and `ChatService` to extract order fields and populate a
pre-filled draft for user review.

**Extracted fields:**
- `supplier` (name)
- `orderDate`
- `orderNumber` (supplier reference)
- `totalAmount.amount` and `totalAmount.currency`
- Line items: `description`, `quantity`, `unitPrice`, `vatRate`

**Scenario 1 — PDF extraction populates draft:**
```gherkin
GIVEN a PurchaseOrder record with status "draft"
  AND a PDF file containing a readable purchase order is attached via FileService
WHEN the OCR endpoint is called with the purchaseOrderId and fileId
THEN a draft PurchaseOrder is returned with extracted fields populated
  AND each field has an associated confidence score between 0.0 and 1.0
  AND fields with confidence < 0.70 are marked "low_confidence"
```

**Scenario 2 — Low-confidence fields flagged for review:**
```gherkin
GIVEN an OCR extraction result where orderDate confidence is 0.55
WHEN the OcrPurchaseOrderPreview.vue component renders the result
THEN the orderDate field is highlighted in red
  AND the field is auto-focused when the dialog opens
  AND a tooltip reads "Lage betrouwbaarheid — controleer dit veld"
```

**Scenario 3 — Non-PDF file rejected:**
```gherkin
GIVEN a file of type "image/png" attached to a PurchaseOrder
WHEN the OCR endpoint is called
THEN HTTP 422 is returned
  AND the error message is "OCR ondersteunt alleen PDF-bestanden"
```

---

### REQ-POM-002: Automated PO creation from approved purchase requisitions

**Feature:** Automated PO creation from approved requisitions with configurable business rules (demand 25)

When a PurchaseRequisition transitions to status `approved`, the system SHALL
automatically create a draft PurchaseOrder using `RequisitionToPurchaseOrderService`.

**Business rules (configurable):**
- If a preferred Supplier exists for the ProcurementCategory, set as PO supplier.
- If matching CatalogItems exist for requisition line items, reference them.
- If the estimated amount exceeds the configured threshold (default: € 5,000), flag for
  additional approval before auto-generation.

**Scenario 1 — Approved requisition generates draft PO:**
```gherkin
GIVEN PurchaseRequisition "pr-2026-04-017" has status "approved"
  AND a preferred Supplier exists for ProcurementCategory "ICT-apparatuur"
WHEN the WorkflowEngineRegistry fires the approval-complete event
THEN a new PurchaseOrder is created with status "draft"
  AND the PurchaseOrder supplier is set to the preferred Supplier
  AND the PurchaseOrder reference links to PurchaseRequisition "pr-2026-04-017"
  AND a Nextcloud notification is sent to the Inkoopmedewerker
```

**Scenario 2 — High-value requisition NOT auto-generated:**
```gherkin
GIVEN PurchaseRequisition with estimatedAmount 7500.00 EUR is approved
  AND the auto-generation threshold is configured at 5000.00 EUR
WHEN the approval-complete event fires
THEN NO PurchaseOrder is auto-created
  AND a notification is sent: "Inkooporder vereist handmatige aanmaak (boven drempelwaarde)"
```

**Scenario 3 — Duplicate prevention:**
```gherkin
GIVEN PurchaseRequisition "pr-2026-04-017" already has linkedPurchaseOrder set
WHEN the approval event fires again (e.g. re-approval)
THEN NO additional PurchaseOrder is created
  AND the service logs "PO already exists for requisition pr-2026-04-017, skipping"
```

---

### REQ-POM-003: Automated reordering for recurring supplies

**Feature:** Automated reordering for recurring supplies with scheduled purchase generation (demand 14)

The system SHALL support configuring CatalogItems for automatic reorder when usage
patterns detected via SpendTransaction history indicate depletion.

**Scenario 1 — Reorder trigger generates PurchaseRequisition:**
```gherkin
GIVEN CatalogItem "cat-item-printpapier-002" has a reorderThreshold of 10 units
  AND SpendTransaction analysis estimates current stock at 8 units
WHEN the DemandAnomalyDetectionJob runs
THEN a new PurchaseRequisition is created with description "Automatisch herbestelling: Printpapier A4"
  AND the requestedBy field is set to the system service account
  AND a Nextcloud notification is sent to the responsible Inkoopmedewerker
```

---

### REQ-POM-004: Advanced PO approval workflows with automated routing

**Feature:** Advanced PO approval workflows with automated routing and bottleneck elimination (demand 49)

The system SHALL monitor ApprovalRequest records for stalled approvals and automatically
re-route or escalate when an approver is overdue beyond the configured SLA.

**Scenario 1 — Stalled approval escalated:**
```gherkin
GIVEN an ApprovalRequest for PurchaseOrder "po-2026-05-001" is in status "pending"
  AND the approver has not acted within 48 hours (configured SLA)
WHEN the DemandAnomalyDetectionJob runs and detects the stall
THEN the ApprovalRequest is escalated to the configured backup approver
  AND the original approver receives a notification "Goedkeuringsverzoek overgedragen wegens termijnoverschrijding"
  AND an audit trail entry is created on the PurchaseOrder
```

---

## REQ-AIA — AI Agents

### REQ-AIA-001: Named procurement AI agents for S2P task automation

**Feature:** 30+ ready-to-use Procurement AI Agents for task automation across S2P lifecycle (demand 1043)

The system SHALL provide a library of named, configurable AI agents, each implementing a
discrete procurement task. Agents are listed in `ProcurementAgentPanel.vue` and dispatched
via `ProcurementAgentController`.

**Required agents (minimum viable set):**
- `catalog-validator` — validates all active CatalogItems against current supplier price lists
- `spend-classifier` — classifies unclassified SpendTransaction records into SpendCategory
- `demand-detector` — identifies anomalous spend patterns against historical baselines
- `po-generator` — creates draft POs from approved requisitions in bulk
- `strategy-advisor` — analyses spend data and generates procurement strategy recommendations
- `supplier-kpi-updater` — recalculates SupplierKPI scores from GoodsReceipt and Invoice data

**Scenario 1 — Agent listed and triggered:**
```gherkin
GIVEN the ProcurementAgentPanel lists agent "spend-classifier"
WHEN the Inkoopmedewerker clicks "Uitvoeren" on the "spend-classifier" agent
THEN a POST request is sent to /api/procurement-agents/spend-classifier/run
  AND the agent status changes to "running"
  AND a taskId is returned for status polling
```

**Scenario 2 — Agent run completes and result recorded:**
```gherkin
GIVEN agent "spend-classifier" is running
WHEN the agent task completes
THEN agent status changes to "completed"
  AND lastRunAt is updated to the current timestamp
  AND a summary is available: "47 transacties geclassificeerd, 3 overgeslagen"
```

**Scenario 3 — Agent run failure handled gracefully:**
```gherkin
GIVEN agent "catalog-validator" is running
  AND the ChatService returns an error
WHEN the error is propagated to ProcurementAgentService
THEN the agent status is set to "failed"
  AND the error message is stored (without stack trace or internal paths)
  AND the Inkoopmedewerker receives a Nextcloud notification with the failure summary
```

---

### REQ-AIA-002: Custom AI agent creation for procurement tasks

**Feature:** Custom AI Agent Creation for Procurement Tasks (demand 145)

Authorised users (shillinq:procurement-admin group) SHALL be able to define custom
AI agent configurations with a name, description, trigger condition, and prompt template.

**Scenario 1 — Custom agent created by admin:**
```gherkin
GIVEN the user is a member of group "shillinq:procurement-admin"
WHEN the user submits a new agent configuration via the agent settings form
THEN a new agent configuration is persisted via ObjectService
  AND the agent appears in ProcurementAgentPanel
  AND a non-admin user can trigger the agent
```

**Scenario 2 — Non-admin cannot create agents:**
```gherkin
GIVEN the user is NOT a member of group "shillinq:procurement-admin"
WHEN the user attempts to POST to /api/procurement-agents
THEN HTTP 403 is returned
  AND the error message is "Onvoldoende rechten voor het aanmaken van agenten"
```

---

### REQ-AIA-003: JAI Copilot — procurement insights and workflow automation

**Feature:** JAI Copilot AI assistant providing procurement insights and workflow automation (demand 286)

The Spend Analyst chat (also branded as JAI Copilot) SHALL support multi-turn conversations
with context retrieval from OpenRegister procurement data.

**Scenario 1 — Multi-turn context retained:**
```gherkin
GIVEN a user has asked "Hoeveel hebben we besteed bij Coolblue dit jaar?"
  AND the assistant replied with a spend figure
WHEN the user asks a follow-up "En in het vorige kwartaal?"
THEN the system uses the existing conversationId to maintain context
  AND the reply refers to Coolblue without requiring the user to repeat the supplier name
```

**Scenario 2 — RAG context from OpenRegister:**
```gherkin
GIVEN SpendTransaction records exist for supplier "Coolblue Business" in Q1 2026
WHEN the user asks "Hoeveel hebben we besteed bij Coolblue in Q1 2026?"
THEN the ChatService retrieves relevant SpendTransaction aggregates via ContextRetrievalHandler
  AND the reply includes the correct aggregated amount
  AND source references link to the underlying SpendTransaction records
```

---

## REQ-SPD — Spend Intelligence

### REQ-SPD-001: Automated spend category classification

**Feature:** Spend visibility across all procurement channels with automated category classification (demand 556)

The system SHALL automatically assign a SpendCategory to SpendTransaction records that
have no category set, using vector similarity between the transaction description and
SpendCategory descriptions via `VectorizationService`.

**Scenario 1 — Transaction classified correctly:**
```gherkin
GIVEN SpendTransaction "st-2026-05-0011" with description "Diverse kantoorartikelen"
  AND SpendCategory "Kantoorbenodigdheden" has a matching vector embedding
WHEN SpendClassificationService.classify() is called for "st-2026-05-0011"
THEN the transaction spendCategory is set to "Kantoorbenodigdheden"
  AND classifiedByAi is set to true
  AND classificationConfidence is a float between 0.0 and 1.0
```

**Scenario 2 — Low-confidence classification not auto-applied:**
```gherkin
GIVEN a SpendTransaction description with very low semantic similarity to all SpendCategories
  AND the highest similarity score is 0.42
WHEN SpendClassificationService processes the transaction
THEN spendCategory is NOT set
  AND a flag "requires_manual_classification" is set on the transaction
  AND the transaction appears in the "Handmatige classificatie vereist" queue
```

---

### REQ-SPD-002: AI procurement performance management aligned to financial goals

**Feature:** AI procurement performance management aligned to financial goals (demand 1709)

The system SHALL generate AI-driven procurement performance insights by correlating
SupplierKPI, SpendTransaction, Budget, and GoodsReceipt data, surfaced via a dashboard widget.

**Scenario 1 — Performance summary generated:**
```gherkin
GIVEN Budget records exist for the current FiscalYear
  AND SpendTransaction records exist for the same period
  AND SupplierKPI records exist for active suppliers
WHEN the Financieel Controller opens the Procurement Performance widget
THEN a summary is displayed showing:
    - spend vs. budget utilisation as a percentage
    - number of suppliers with KPI score below threshold
    - top 3 categories by spend deviation from budget
```

---

### REQ-SPD-003: Demand anomaly detection and alerting

**Feature:** AI-powered demand detection identifying spending pattern changes (demand 116)

The system SHALL detect statistically significant deviations in spend per ProcurementCategory
compared to the historical baseline (trailing 3 months) and notify the responsible
Inkoopmedewerker.

**Scenario 1 — Anomaly detected and notified:**
```gherkin
GIVEN ProcurementCategory "Facilitaire diensten" has an average monthly spend of € 2,000
  AND current month spend is € 5,500 (175% above baseline)
  AND the anomaly threshold is configured at 50% deviation
WHEN DemandAnomalyDetectionJob runs
THEN a MaverickSpendAlert is created for the category
  AND a Nextcloud notification is sent to the responsible user
  AND the DemandAnomalyWidget shows the category in the alert list
```

**Scenario 2 — Anomaly below threshold not alerted:**
```gherkin
GIVEN ProcurementCategory "ICT-apparatuur" has current month spend 30% above baseline
  AND the anomaly threshold is 50%
WHEN DemandAnomalyDetectionJob runs
THEN NO MaverickSpendAlert is created for this category
  AND NO notification is sent
```

---

### REQ-SPD-004: Navi AI Analytics Agent — interactive procurement reporting

**Feature:** Navi AI Analytics Agent: interactive procurement data visualization and reporting (demand 432)

The Navi agent SHALL produce structured procurement reports from historical data on demand,
including spend-by-category, supplier performance ranking, and budget variance analysis.

**Scenario 1 — Spend report generated:**
```gherkin
GIVEN SpendTransaction records exist for the last 12 months
WHEN the Navi agent is triggered via ProcurementAgentController with task "spend-report-q2"
THEN a report object is created containing:
    - total spend per SpendCategory
    - top 10 suppliers by spend
    - month-over-month trend for each category
  AND the report is linked to the triggering user via ActivityService
```

---

### REQ-SPD-005: AI Strategies Agent — actionable procurement strategies from spend data

**Feature:** AI Strategies Agent transforming spend data into actionable procurement strategies (demand 281)

The strategy agent SHALL analyse aggregated SpendTransaction records and generate
concrete procurement recommendations (consolidation, preferred supplier designation,
spot-buy elimination).

**Scenario 1 — Consolidation recommendation generated:**
```gherkin
GIVEN SpendTransaction records show purchases from 5 different suppliers
      for ProcurementCategory "Kantoorbenodigdheden" in the past quarter
  AND combined spend is € 4,200 with no single supplier above 30%
WHEN the "strategy-advisor" agent runs
THEN the agent produces a recommendation: "Consolideer kantoorbenodigdheden bij 1-2 leveranciers
     voor geschatte besparing van 12-18% op jaarbasis"
  AND the recommendation is stored as a ProcurementComplianceReport or advisory note
```

---

### REQ-SPD-006: Automated report generation from historical procurement data

**Feature:** Automated report generation from historical procurement data (demand 352)

The system SHALL support scheduled and on-demand generation of standard procurement reports
using `WorkflowEngineRegistry` + `ChatService`.

**Scenario 1 — Scheduled monthly report generated:**
```gherkin
GIVEN a scheduled workflow is configured for "monthly-spend-report" on the 1st of each month
WHEN the ScheduledWorkflowController fires the job
THEN a procurement report is generated covering the previous calendar month
  AND the report is stored as a Document in OpenRegister
  AND all Financieel Controller users receive a Nextcloud notification with a link
```

---

## REQ-REQ — Request Creation

### REQ-REQ-001: AI Request Creation Agent — unstructured documents to purchase requisitions

**Feature:** AI Request Creation Agent: unstructured documents to purchase requisitions (demand 179)

The system SHALL accept an uploaded document (PDF, email export, scanned form) and use
`TextExtractionService` + `ChatService` to extract requisition data and create a draft
PurchaseRequisition.

**Scenario 1 — Email export converted to requisition:**
```gherkin
GIVEN a user uploads an email export (PDF) containing a purchase request from a department head
WHEN the AI Request Creation Agent processes the file
THEN a draft PurchaseRequisition is created with:
    - description extracted from email body
    - estimatedAmount extracted if mentioned
    - procurementCategory inferred from item descriptions
    - requestedBy set to the uploading user
  AND the requisition status is "draft" pending user review
```

**Scenario 2 — Extraction failure creates empty draft:**
```gherkin
GIVEN a user uploads a scanned form that is unreadable (low DPI, blurry)
  AND TextExtractionService returns empty text
WHEN the AI Request Creation Agent processes the file
THEN a draft PurchaseRequisition is created with empty fields
  AND a warning is shown: "Tekst kon niet worden uitgelezen — vul de velden handmatig in"
  AND the uploaded file is attached to the requisition for manual reference
```

---

## Non-Functional Requirements

### REQ-NFR-001: AI response latency
```gherkin
GIVEN a Spend Analyst chat query
WHEN the user submits the message
THEN the first token of the response MUST appear within 5 seconds
  AND the full response MUST complete within 30 seconds
```

### REQ-NFR-002: OCR extraction latency
```gherkin
GIVEN a PDF PurchaseOrder of at most 10 pages
WHEN OCR extraction is triggered
THEN the extracted fields MUST be returned within 60 seconds
```

### REQ-NFR-003: Classification throughput
```gherkin
GIVEN a batch of 500 unclassified SpendTransaction records
WHEN SpendClassificationService processes the batch
THEN all records MUST be processed within 5 minutes
```

### REQ-NFR-004: Accessibility (ADR-010)
```gherkin
GIVEN any AI feature UI component (chat panel, OCR preview, agent panel)
WHEN inspected with an accessibility checker
THEN all interactive elements MUST have ARIA labels
  AND keyboard navigation MUST be fully functional
  AND colour contrast MUST meet WCAG AA (minimum 4.5:1)
```

### REQ-NFR-005: Idempotent catalog import
```gherkin
GIVEN a CIF/cXML catalog file previously imported successfully
WHEN the same file is imported again
THEN no duplicate CatalogItem objects are created
  AND the response indicates items were skipped as already existing
```
