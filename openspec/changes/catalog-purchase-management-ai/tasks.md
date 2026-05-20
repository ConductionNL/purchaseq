# Tasks: Catalog & Purchase Management — Shillinq — Ai

**Change:** catalog-purchase-management-ai
**Date:** 2026-05-20
**Spec:** @spec openspec/changes/catalog-purchase-management-ai/tasks.md

---

## Deduplication Check

Per ADR-012, the following OpenRegister services have been audited for overlap:

| Capability | OpenRegister service | Finding |
|------------|---------------------|---------|
| CRUD for all entities | `ObjectService` | Fully covered — no custom CRUD needed |
| File upload / text extraction | `TextExtractionService`, `FileService` | Fully covered — wrapped only |
| Semantic / hybrid search | `VectorizationService`, `SolrController` | Fully covered — no custom search needed |
| AI conversation / RAG | `ChatService`, `ContextRetrievalHandler` | Fully covered — wrapped only |
| Approval workflow routing | `WorkflowEngineRegistry`, `ApprovalChain` | Fully covered — event listener only |
| Notifications | `NotificationService` | Fully covered — no custom dispatcher |
| Background jobs | `IJob` / job queue | Framework provided — custom job logic only |
| Dashboard widgets | `CnDashboardPage`, `CnWidgetWrapper` | Fully covered — custom widget content only |
| Bulk import UI | `CnMassImportDialog` | Fully covered — custom parser only |

**Conclusion:** No overlap with existing platform services. Custom code is limited to
procurement-domain business rules and format-specific parsers that have no generic analogue.

---

## Task Groups

1. [Backend — Parser & Validation Services](#1-backend--parser--validation-services)
2. [Backend — AI Intelligence Services](#2-backend--ai-intelligence-services)
3. [Backend — Workflow Services & Background Jobs](#3-backend--workflow-services--background-jobs)
4. [Backend — Controllers](#4-backend--controllers)
5. [Backend — Tests](#5-backend--tests)
6. [Frontend — Vue Components](#6-frontend--vue-components)
7. [Frontend — Tests](#7-frontend--tests)
8. [Configuration & Seed Data](#8-configuration--seed-data)
9. [Documentation](#9-documentation)

---

## 1. Backend — Parser & Validation Services

- [ ] **task-1.1** Create `lib/Service/CifCxmlParserService.php`
  - Parses CIF 3.0 files: extract `<Classification>`, `<ItemID>`, `<UnitPrice>`, `<UnitOfMeasure>` elements
  - Parses cXML 1.2 files: extract `<ItemIn>`, `<PriceBasisQuantity>`, `<ItemDetail>` elements
  - Returns an array of raw item arrays suitable for `AiCatalogValidationService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-1.1`
  - Validates MIME type before parsing (accept: `text/plain`, `application/xml`, `text/xml`)
  - Rejects files > 25 MB with a descriptive exception

- [ ] **task-1.2** Create `lib/Service/AiCatalogValidationService.php`
  - Validates required fields per REQ-CAT-001: `name`, `sku`, `unitPrice.amount`, `vatRate`, `procurementCategory`, `unitOfMeasure`
  - Checks `vatRate` against allowed values `[0, 9, 21]`
  - Detects duplicate SKU by calling `ObjectService::searchObjects` against target ProcurementCatalog
  - Calls `ChatService` for anomaly detection (price outliers > 3σ from category mean)
  - Returns structured result: `{ valid: [...], invalid: [{ line, item, errors, suggestions }] }`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-1.2`

- [ ] **task-1.3** Add translations for catalog validation messages to `l10n/nl.js` and `l10n/en.js`
  - Keys: `ai.catalog.validation.missing_field`, `ai.catalog.validation.duplicate_sku`,
    `ai.catalog.validation.invalid_vat`, `ai.catalog.validation.price_outlier`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-1.3`

---

## 2. Backend — AI Intelligence Services

- [ ] **task-2.1** Create `lib/Service/OcrPurchaseOrderService.php`
  - Accepts `purchaseOrderId` and `fileId`; retrieves file via `FileService`
  - Calls `TextExtractionService::extractText()` on the file
  - Constructs a structured prompt for `ChatService` requesting extraction of:
    supplier name, orderDate, orderNumber, totalAmount, currency, and line items
  - Maps ChatService response to PurchaseOrder field names
  - Computes per-field confidence scores (0.0–1.0) from ChatService token probabilities or
    presence heuristics
  - Returns `{ extracted: {...}, confidence: {...} }`
  - Does NOT write to ObjectService — caller decides whether to save
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-2.1`

- [ ] **task-2.2** Create `lib/Service/SpendClassificationService.php`
  - Accepts an array of SpendTransaction IDs
  - For each transaction: fetches description, embeds via `VectorizationService`,
    finds nearest SpendCategory by cosine similarity
  - If similarity ≥ 0.65: updates `spendCategory`, sets `classifiedByAi = true`,
    sets `classificationConfidence`
  - If similarity < 0.65: sets `requiresManualClassification = true`, skips category assignment
  - Processes in batches of 50 to stay within LLM rate limits
  - Returns summary: `{ classified: N, skipped: N, failed: N }`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-2.2`

- [ ] **task-2.3** Create `lib/Service/ProcurementAgentService.php`
  - Maintains a registry of named agents (from config + custom objects in OpenRegister)
  - `listAgents(): array` — returns all agents with name, slug, description, lastRunAt, status
  - `runAgent(string $slug, array $context): string` — dispatches agent task, returns taskId
  - `getAgentStatus(string $taskId): array` — polls task status via `TasksController`
  - Each built-in agent delegates to specific service method:
    - `catalog-validator` → `AiCatalogValidationService::validateAll()`
    - `spend-classifier` → `SpendClassificationService::classifyAll()`
    - `demand-detector` → `DemandAnomalyDetectionJob::runManual()`
    - `po-generator` → `RequisitionToPurchaseOrderService::generatePending()`
    - `strategy-advisor` → internal: calls `ChatService` with aggregated spend context
    - `supplier-kpi-updater` → internal: recalculates SupplierKPI from GoodsReceipt + Invoice data
  - Enforces 60-second per-user cooldown per agent slug
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-2.3`

- [ ] **task-2.4** Create `lib/Service/AiRequestCreationService.php`
  - Accepts a fileId; retrieves and extracts text via `TextExtractionService`
  - If extracted text is empty: creates draft PurchaseRequisition with empty fields + warning flag
  - Otherwise: calls `ChatService` with structured extraction prompt for:
    description, estimatedAmount, currency, procurementCategory, requestedBy hints
  - Creates draft PurchaseRequisition via `ObjectService::saveObject()`
  - Attaches the original file to the requisition via `FileService`
  - Returns the new PurchaseRequisition objectId
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-2.4`

- [ ] **task-2.5** Add translations for AI intelligence service messages to `l10n/nl.js` and `l10n/en.js`
  - Keys: `ai.po.ocr.*`, `ai.spend.classification.*`, `ai.request.creation.*`, `ai.agent.*`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-2.5`

---

## 3. Backend — Workflow Services & Background Jobs

- [ ] **task-3.1** Create `lib/Service/RequisitionToPurchaseOrderService.php`
  - Listens for `PurchaseRequisition.status = approved` event via `WorkflowEngineRegistry`
  - Guards: skip if `linkedPurchaseOrder` already set on requisition
  - Guards: skip if `estimatedAmount` exceeds configured threshold (default: 5000 EUR);
    send notification instead
  - Looks up preferred Supplier for the requisition's `procurementCategory` via
    `ObjectService::searchObjects` with filter `{ preferredFor: category }`
  - Creates PurchaseOrder via `ObjectService::saveObject()` with status `draft`
  - Sets `linkedPurchaseOrder` on the source PurchaseRequisition
  - Sends Nextcloud notification to Inkoopmedewerker group via `NotificationService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-3.1`

- [ ] **task-3.2** Create `lib/BackgroundJob/DemandAnomalyDetectionJob.php` (extends `TimedJob`)
  - Runs daily at 06:00 via Nextcloud job queue
  - `runManual()` method for on-demand execution from ProcurementAgentService
  - Algorithm:
    1. Fetch all active ProcurementCategory records
    2. For each category: compute trailing-3-month average spend from SpendTransaction records
    3. Compare current month spend to baseline; compute deviation percentage
    4. If deviation ≥ configured threshold (default: 50%): create MaverickSpendAlert via ObjectService
    5. Send notification via NotificationService to responsible user
  - Reorder detection: for CatalogItems with `reorderThreshold` set, estimate current stock
    from SpendTransaction patterns; create PurchaseRequisition if below threshold
  - Stalled approval detection: find ApprovalRequest records in `pending` status older than
    configured SLA (default: 48 h); re-route to backup approver
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-3.2`

- [ ] **task-3.3** Register `DemandAnomalyDetectionJob` in `lib/AppInfo/Application.php`
  - Add to `registerService` and `registerBackgroundJob`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-3.3`

- [ ] **task-3.4** Add translations for workflow and notification messages to `l10n/nl.js` and `l10n/en.js`
  - Keys: `ai.workflow.po_generated`, `ai.workflow.po_above_threshold`,
    `ai.workflow.approval_escalated`, `ai.anomaly.alert_title`, `ai.anomaly.reorder_created`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-3.4`

---

## 4. Backend — Controllers

- [ ] **task-4.1** Create `lib/Controller/AiCatalogController.php`
  - `POST /api/ai-catalog/validate` — calls `AiCatalogValidationService::validate()`
  - `POST /api/ai-catalog/import-cif` — calls `CifCxmlParserService::parse()` then validation,
    then stages items via `ObjectService` (respects `dryRun` param)
  - Auth: `#[NoAdminRequired]` (all authenticated users may import)
  - File size limit enforced: reject > 25 MB with HTTP 422
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-4.1`

- [ ] **task-4.2** Create `lib/Controller/OcrController.php`
  - `POST /api/purchase-orders/ocr` — calls `OcrPurchaseOrderService::extract()`
  - Auth: `#[NoAdminRequired]`
  - Returns extracted fields + confidence map; does NOT auto-save to ObjectService
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-4.2`

- [ ] **task-4.3** Create `lib/Controller/SpendAnalystController.php`
  - `POST /api/spend-analyst/conversation` — passes message + conversationId to `ChatService`
  - `GET /api/spend-analyst/conversation/{id}` — retrieves conversation history
  - Anonymises entity identifiers before passing to ChatService (no BSN/NAW data in prompts)
  - Returns ChatService response with source references mapped to ObjectService URLs
  - Auth: `#[NoAdminRequired]`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-4.3`

- [ ] **task-4.4** Create `lib/Controller/ProcurementAgentController.php`
  - `GET /api/procurement-agents` — calls `ProcurementAgentService::listAgents()`
  - `POST /api/procurement-agents/{agentSlug}/run` — calls `ProcurementAgentService::runAgent()`
  - `GET /api/procurement-agents/{agentSlug}/status/{taskId}` — calls `ProcurementAgentService::getAgentStatus()`
  - `POST /api/procurement-agents` (create custom agent) — admin only: `IGroupManager::isAdmin()`
    check on server side; HTTP 403 if not authorised
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-4.4`

- [ ] **task-4.5** Register all new routes in `appinfo/routes.php`
  - Place specific routes BEFORE any existing `{slug}` wildcard routes (ADR-003)
  - Add CORS OPTIONS route for each public-facing endpoint if applicable
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-4.5`

- [ ] **task-4.6** Add translations for controller error messages to `l10n/nl.js` and `l10n/en.js`
  - Keys: `api.error.unsupported_format`, `api.error.file_too_large`,
    `api.error.insufficient_permissions`, `api.error.agent_cooldown`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-4.6`

---

## 5. Backend — Tests

- [ ] **task-5.1** Create `tests/Unit/Service/CifCxmlParserServiceTest.php`
  - Tests: valid CIF 3.0 file parses correctly, valid cXML 1.2 file parses correctly,
    malformed XML throws ParseException, file > 25 MB throws FileTooLargeException,
    unsupported MIME type throws UnsupportedFormatException
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.1`

- [ ] **task-5.2** Create `tests/Unit/Service/AiCatalogValidationServiceTest.php`
  - Tests: all-valid batch returns empty errors, missing required field flagged,
    duplicate SKU detected, invalid vatRate rejected, price outlier flagged by AI
  - Mock `ObjectService` and `ChatService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.2`

- [ ] **task-5.3** Create `tests/Unit/Service/OcrPurchaseOrderServiceTest.php`
  - Tests: PDF extraction maps fields correctly, low-confidence fields flagged,
    non-PDF MIME type throws exception, empty extraction returns empty fields without error
  - Mock `TextExtractionService` and `ChatService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.3`

- [ ] **task-5.4** Create `tests/Unit/Service/SpendClassificationServiceTest.php`
  - Tests: high-similarity transaction classified correctly, low-similarity transaction
    skipped with `requiresManualClassification` flag, batch of 500 processed in ≤ 50-item chunks
  - Mock `VectorizationService` and `ObjectService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.4`

- [ ] **task-5.5** Create `tests/Unit/Service/RequisitionToPurchaseOrderServiceTest.php`
  - Tests: approved requisition creates PO, already-linked requisition skips,
    high-value requisition sends notification instead of creating PO,
    preferred supplier correctly set on generated PO
  - Mock `ObjectService` and `NotificationService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.5`

- [ ] **task-5.6** Create `tests/Unit/BackgroundJob/DemandAnomalyDetectionJobTest.php`
  - Tests: deviation above threshold creates MaverickSpendAlert,
    deviation below threshold creates no alert, reorder threshold triggers PurchaseRequisition,
    stalled approval escalated to backup approver
  - Mock `ObjectService` and `NotificationService`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.6`

- [ ] **task-5.7** Create `tests/Integration/AiCatalogControllerTest.json` (Newman collection)
  - Scenarios: dry-run CIF import, full CIF import with invalid rows, re-import idempotency,
    file too large rejected, unsupported format rejected
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.7`

- [ ] **task-5.8** Create `tests/Integration/SpendAnalystControllerTest.json` (Newman collection)
  - Scenarios: send message, retrieve history, empty message rejected (HTTP 422),
    follow-up uses existing conversationId
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.8`

- [ ] **task-5.9** Create `tests/Integration/ProcurementAgentControllerTest.json` (Newman collection)
  - Scenarios: list agents, run agent (non-admin), create agent (admin), create agent (non-admin → 403)
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-5.9`

---

## 6. Frontend — Vue Components

- [ ] **task-6.1** Create `src/components/SpendAnalystChat.vue`
  - Chat panel with message list, input field, send button
  - Calls `POST /api/spend-analyst/conversation`; persists `conversationId` in component state
  - Messages rendered with sanitised markdown (use existing Nextcloud markdown utility)
  - Source references rendered as links to OpenRegister object detail pages
  - "Wissen" button clears conversation (new conversationId on next message)
  - All strings via `t(appName, 'ai.spend.analyst.*')`
  - WCAG AA: input has `aria-label`, messages list has `role="log"` and `aria-live="polite"`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.1`

- [ ] **task-6.2** Create `src/components/CatalogValidationResultsDialog.vue`
  - Modal dialog (wraps `NcDialog`) showing summary counts (valid / invalid)
  - Invalid items listed with line number, error description, and action button
  - "Geldige items importeren" button calls `POST /api/ai-catalog/import-cif` without `dryRun`
  - "Annuleren" dismisses without saving
  - All strings via `t(appName, 'ai.catalog.validation.*')`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.2`

- [ ] **task-6.3** Create `src/components/OcrPurchaseOrderPreview.vue`
  - Inline panel showing extracted PO fields with confidence indicators
  - Confidence ≥ 0.90: green border; 0.70–0.89: amber; < 0.70: red + auto-focus
  - Each field is an editable `NcTextField`; changes update draft PO via `ObjectService`
  - "Bevestigen" button saves the draft PurchaseOrder and closes the panel
  - Tooltip on red fields: `t(appName, 'ai.po.ocr.low_confidence_tooltip')`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.3`

- [ ] **task-6.4** Create `src/components/ProcurementAgentPanel.vue`
  - Embedded in Settings page within a `CnDetailCard`
  - Agent list: name, description, status badge (`CnStatusBadge`), last-run timestamp, "Uitvoeren" button
  - Run button dispatches `POST /api/procurement-agents/{slug}/run`; polls status every 5 seconds
  - "Uitvoeren" button disabled during cooldown (60 s); shows countdown
  - Admin-only "Nieuwe agent" button visible only when `isAdmin` is true (from settings store)
  - All strings via `t(appName, 'ai.agent.*')`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.4`

- [ ] **task-6.5** Create `src/components/DemandAnomalyWidget.vue`
  - Dashboard widget (wrapped in `CnWidgetWrapper`)
  - KPI: count of categories with active MaverickSpendAlert
  - List: top 5 alerting categories with delta percentage and trend arrow icon
  - Row click navigates to SpendTransaction list filtered by the category
  - Empty state when no alerts: `CnEmptyState` with message `t(appName, 'ai.anomaly.no_alerts')`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.5`

- [ ] **task-6.6** Register `DemandAnomalyWidget` in the Nextcloud Dashboard API
  - Implement as a Nextcloud Dashboard widget (`IWidget`) in `lib/Dashboard/DemandAnomalyDashboard.php`
  - Register in `Application.php`
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.6`

- [ ] **task-6.7** Add `SpendAnalystChat` and `ProcurementAgentPanel` to the Shillinq dashboard page
  - Add route `/spend-analyst` for the chat view
  - Add `/procurement-agents` route for the agent management view
  - Register both routes in `src/router/index.js` (flat, named, no nesting)
  - Add navigation items to `MainMenu.vue` with Dutch labels
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.7`

- [ ] **task-6.8** Add Pinia store `src/store/modules/procurementAgent.js`
  - Uses `createObjectStore('procurementAgent')` with `searchPlugin` and `selectionPlugin`
  - Exposes `fetchAgents()`, `runAgent(slug)`, `pollAgentStatus(taskId)` actions
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.8`

- [ ] **task-6.9** Add Pinia store `src/store/modules/spendAnalyst.js`
  - Manages `conversationId` and `messages` array
  - `sendMessage(text)` action calls `SpendAnalystController`; appends reply to messages
  - `clearConversation()` action resets conversationId and messages
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-6.9`

---

## 7. Frontend — Tests

- [ ] **task-7.1** Create Playwright test: `tests/Browser/SpendAnalystChatTest.js`
  - GIVEN chat panel is open
  - WHEN user types "Wat hebben we dit kwartaal besteed aan ICT?" and submits
  - THEN a reply is rendered in the message list
  - AND source links are present in the response
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-7.1`

- [ ] **task-7.2** Create Playwright test: `tests/Browser/OcrPurchaseOrderPreviewTest.js`
  - GIVEN a PurchaseOrder detail page is open
  - WHEN user uploads a PDF via the file tab
  - AND clicks "Verwerk OCR"
  - THEN the OcrPurchaseOrderPreview panel appears with extracted fields
  - AND low-confidence fields are shown with red indicator
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-7.2`

- [ ] **task-7.3** Create Playwright test: `tests/Browser/CatalogValidationResultsDialogTest.js`
  - GIVEN a CIF file with 3 invalid rows is imported
  - WHEN the dialog opens
  - THEN invalid count shows "3 items ongeldig"
  - AND "Geldige items importeren" button is enabled
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-7.3`

---

## 8. Configuration & Seed Data

- [ ] **task-8.1** Add seed data for CatalogItem, PurchaseOrder, PurchaseRequisition, and SpendTransaction
  to `lib/Settings/shillinq_register.json` under `components.objects[]`
  - Use `@self` envelope with `register`, `schema`, `slug` fields
  - 5 CatalogItems, 4 PurchaseOrders, 3 PurchaseRequisitions, 4 SpendTransactions (see design.md)
  - Slugs must be unique and stable for idempotency
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-8.1`

- [ ] **task-8.2** Add configurable settings for AI procurement features to `IAppConfig`:
  - `shillinq.ai.auto_po_threshold` (float, default 5000.0) — max amount for auto-PO generation
  - `shillinq.ai.anomaly_threshold_pct` (int, default 50) — % deviation for demand alert
  - `shillinq.ai.classification_min_confidence` (float, default 0.65) — min score for auto-classify
  - `shillinq.ai.agent_cooldown_seconds` (int, default 60) — per-user agent cooldown
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-8.2`

- [ ] **task-8.3** Add AI settings section to `src/views/SettingsPage.vue`
  - Section title: `t(appName, 'settings.ai.title')` = "AI-instellingen"
  - Fields: auto-PO threshold, anomaly threshold, classification confidence, agent cooldown
  - Reads from `GET /api/settings`; saves via `POST /api/settings`
  - Show `ProcurementAgentPanel` component below the settings fields
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-8.3`

- [ ] **task-8.4** Add `metrics` endpoint support for AI features in the existing metrics controller:
  - `shillinq_ai_agents_total{status}` — count of agent runs by status (completed/failed)
  - `shillinq_ai_spend_classified_total` — cumulative SpendTransactions classified by AI
  - `shillinq_ai_ocr_extractions_total{result}` — OCR extractions by result (success/failure)
  - `shillinq_ai_anomaly_alerts_active` — current active MaverickSpendAlert count
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-8.4`

---

## 9. Documentation

- [ ] **task-9.1** Create `docs/ai-catalog-management.md`
  - How to upload CIF/cXML catalog files
  - How to review and approve validation results
  - How to configure reorder thresholds per CatalogItem
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-9.1`

- [ ] **task-9.2** Create `docs/ai-purchase-order-ocr.md`
  - How to trigger OCR on an uploaded PDF
  - How to review and correct extracted fields
  - Supported file formats and size limits
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-9.2`

- [ ] **task-9.3** Create `docs/ai-spend-analyst.md`
  - How to use the Spend Analyst chat
  - Example queries in Dutch
  - How conversation context works
  - Privacy note: no personal data sent to AI
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-9.3`

- [ ] **task-9.4** Create `docs/ai-procurement-agents.md`
  - List of built-in agents with descriptions and trigger conditions
  - How to create a custom agent (admin)
  - How to configure agent cooldowns and schedules
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-9.4`

- [ ] **task-9.5** Create `docs/ai-demand-anomaly.md`
  - How the demand anomaly detection works
  - How to configure the deviation threshold
  - How to interpret DemandAnomalyWidget alerts
  - How to act on automated reorder requisitions
  - `@spec openspec/changes/catalog-purchase-management-ai/tasks.md#task-9.5`
