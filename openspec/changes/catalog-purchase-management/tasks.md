# Tasks: Catalog & Purchase Management — Shillinq

Implementation tasks for the `catalog-purchase-management` change.
All tasks are checkboxes — mark `[x]` when complete.

---

## 0. Deduplication check

- [ ] **Task 0.1** — Verify no overlap with OpenRegister core services: confirm `ObjectService`, `ImportService`, `AuditTrailService`, `NotificationService`, `WorkflowEngineController`, `AuthorizationService`, and `VectorizationService` are used as-is without reimplementation
- [ ] **Task 0.2** — Verify no overlap with `@conduction/nextcloud-vue`: confirm `CnIndexPage`, `CnDetailPage`, `CnFormDialog`, `CnMassImportDialog`, `CnDashboardPage`, `CnChartWidget`, `CnStatsBlock`, `CnObjectSidebar`, and `CnTimelineStages` are used without custom replacements
- [ ] **Task 0.3** — Verify no overlap with existing Shillinq specs: check that `PurchaseOrder`, `Contract`, `FrameworkAgreement`, `Lot`, `CallOffOrder`, `MaverickSpendAlert`, `ApprovalRequest`, and `Organization` entities are referenced (not redefined) — document findings in design.md Reuse Analysis
- [ ] **Task 0.4** — Document deduplication findings: note "No overlap found for DigiInkoop adapter, OCI punchout handler, CPV classification prompt wrapper, and spend cube dimension configuration — these are domain-specific and have no platform equivalent"

---

## 1. Register and schema definition

- [ ] **Task 1.1** — Create `lib/Settings/shillinq_register.json`: OpenAPI 3.0 + `x-openregister` register template for the `shillinq` register
  - Add schemas: `ProcurementCatalog`, `CatalogItem`, `PricingRule`, `PurchaseRequisition`, `BlanketPurchaseOrder`, `StatementOfWork`, `ProcurementAuditLog`
  - Each schema: schema.org `@type`, explicit property types, `required` array, `description` field per property
  - Mark register with `x-openregister.type: "application"`
  - Relations defined using OpenRegister relation mechanism (register + schema + objectId) — NO foreign keys
- [ ] **Task 1.2** — Add `ProcurementCatalog` schema (`schema:Catalog`): properties `catalogNumber`, `catalogName`, `description`, `catalogFormat` (enum: internal/cxml/cif), `status` (enum: draft/active/archived), `validFrom`, `validUntil`
- [ ] **Task 1.3** — Add `CatalogItem` schema (`schema:Product`): properties `itemCode`, `itemName`, `description`, `basePrice`, `unit`, `minimumQuantity`, `leadTime`, `status` (enum: active/discontinued), `validFrom`, `validUntil`; relations to `ProcurementCatalog`, `Product`
- [ ] **Task 1.4** — Add `PricingRule` schema (`schema:PriceSpecification`): properties `ruleCode`, `description`, `ruleType` (enum: volumeDiscount/tierPricing/bundleDiscount/periodDiscount), `minQuantity`, `maxQuantity`, `discountPercentage`, `discountAmount`, `priority`, `validFrom`, `validUntil`; relation to `CatalogItem`
- [ ] **Task 1.5** — Add `PurchaseRequisition` schema (`schema:Order`): properties `requisitionNumber`, `requisitionDate`, `status` (enum: draft/submitted/approved/rejected/ordered), `purpose`, `deliveryDate`, `customFields` (object), `totalAmount`; relations to `Person` (requester), `Organization`, `ApprovalRequest`
- [ ] **Task 1.6** — Add `BlanketPurchaseOrder` schema (`schema:Order`): properties `blanketPoNumber`, `validFrom`, `validUntil`, `totalAuthorizedAmount`, `consumedAmount`, `remainingAmount`, `releaseSchedule` (array of `{ releaseDate, amount, description }`), `status` (enum: active/closed/cancelled); relations to `Organization`, `ProcurementCatalog`, `PurchaseOrder`, `ApprovalRequest`
- [ ] **Task 1.7** — Add `StatementOfWork` schema (`schema:CreativeWork`): properties `sowNumber`, `sowDate`, `title`, `description`, `scope`, `deliverables` (array of `{ title, description, dueDate, acceptanceCriteria }`), `milestones` (array of `{ title, completionDate, invoiceAmount, invoiceTrigger }`), `totalValue`, `currency`, `status` (enum: draft/active/completed/cancelled); relations to `Organization`, `Person`, `Contract`, `PurchaseOrder`
- [ ] **Task 1.8** — Add `ProcurementAuditLog` schema (`schema:Action`): properties `auditId`, `entityType` (enum: requisition/purchaseOrder/invoice/payment/approval/catalogItem), `entityId`, `actionType` (enum: created/updated/approved/rejected/posted/received/deactivated), `timestamp`, `reason`, `changes` (object), `referenceDocuments` (array); relations to `Person` (actor), `Organization`
- [ ] **Task 1.9** — Mark `ProcurementAuditLog` as immutable in schema definition: set `x-openregister.immutable: true` so the platform blocks update and delete operations on audit log entries

---

## 2. Seed data

- [ ] **Task 2.1** — Add seed data section to `lib/Settings/shillinq_register.json` under `components.objects[]` using `@self` envelope (`register`, `schema`, `slug`):
  - 3 `ProcurementCatalog` objects (ICT Middelen Utrecht 2025, Kantoorartikelen 2025, Inhuur 2024 archief)
  - 4 `CatalogItem` objects (Dell Latitude 5540, Microsoft 365 licentie, HAG Capisco stoel, A4 papier doos)
  - 3 `PricingRule` objects (laptop volume korting, M365 volume korting, kantoor Q4 promotie)
  - 3 `PurchaseRequisition` objects (REQ-2025-0142 approved, REQ-2025-0189 submitted, REQ-2025-0201 draft)
  - 3 `BlanketPurchaseOrder` objects (BPO ICT 2025, BPO Facilitair 2025, BPO Inhuur 2024 gesloten)
  - 3 `StatementOfWork` objects (zaaksysteem migratie, cybersec audit, website herontwerp)
  - 4 `ProcurementAuditLog` objects (approval, price update, created, deactivation)
- [ ] **Task 2.2** — Verify seed data slugs are unique across all schemas in the register file — no duplicate slugs
- [ ] **Task 2.3** — Verify all Dutch values: real Dutch street/city references, valid ISO dates with `+01:00`/`+02:00` offsets, EUR currency, Dutch language descriptions

---

## 3. Repair step (schema import)

- [ ] **Task 3.1** — Create `lib/Migration/RepairImportShillinqRegister.php` implementing `IRepairStep`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-3.1`
  - Calls `ConfigurationService::importFromApp('shillinq', $data, $version, false)` for idempotent schema and seed data import
  - Uses `version_compare` to skip import if version has not changed
  - Registers in `appinfo/info.xml` under `<repair-steps><post-migration>`

---

## 4. Backend services

### CatalogService

- [ ] **Task 4.1** — Create `lib/Service/CatalogService.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.1`
  - `bulkUpdatePrices(array $itemIds, array $newPrices, string $reason): array` — updates `basePrice` on multiple CatalogItems and writes a ProcurementAuditLog entry per item (REQ-CAT-003)
  - `deactivateCatalogItem(string $itemId, string $reason): void` — sets `status = discontinued`, writes audit log entry (REQ-CAT-004)
  - `getUsageStatistics(string $itemId, \DateTime $from, \DateTime $to): array` — aggregates PurchaseRequisition line items for the item in the date range (REQ-CAT-005)
  - Stateless — no instance state between requests
  - Constructor injection via `private readonly` for `ObjectService`, `AuditLogService`

### RequisitionService

- [ ] **Task 4.2** — Create `lib/Service/RequisitionService.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.2`
  - `submitRequisition(string $requisitionId): void` — transitions status to `submitted`, creates `ApprovalRequest` via `WorkflowEngineController`, writes audit log (REQ-PRQ-001)
  - `approveRequisition(string $requisitionId, string $reason): void` — transitions to `approved`, writes audit log, sends notification (REQ-PRQ-002)
  - `rejectRequisition(string $requisitionId, string $reason): void` — transitions to `rejected`, writes audit log, sends notification (REQ-PRQ-002)
  - User identity resolved via `IUserSession::getUser()->getUID()` — NEVER trusts frontend-sent user IDs (ADR-005)

### BlanketOrderService

- [ ] **Task 4.3** — Create `lib/Service/BlanketOrderService.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.3`
  - `trackConsumption(string $blanketPoId, float $amount): void` — increments `consumedAmount`, recomputes `remainingAmount`, saves object (REQ-BPO-001)
  - `validateSpendLimit(string $blanketPoId, float $proposedAmount): bool` — returns false and writes audit log if `proposedAmount > remainingAmount` (REQ-BPO-001 Scenario 3)

### AuditLogService

- [ ] **Task 4.4** — Create `lib/Service/AuditLogService.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.4`
  - `writeEntry(string $entityType, string $entityId, string $actionType, string $reason, array $changes, array $refDocs): void`
  - Generates `auditId` (UUID v4), sets `timestamp` to server time (NOT client-supplied), records `actor` from `IUserSession`
  - NEVER stores display names — uses `$user->getUID()` only (ADR-015, ADR-005)
  - Saves via `ObjectService::saveObject()` with `_rbac: false` to ensure immutable write path

### DigiInkooopOrderService

- [ ] **Task 4.5** — Create `lib/Service/DigiInkooopOrderService.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.5`
  - `dispatch(string $requisitionId): void` — translates approved PurchaseRequisition to UBL 2.1 XML, POSTs to configured DigiInkoop endpoint using stored IAppConfig credentials, writes audit log with actionType = posted (REQ-INT-001)
  - Endpoint URL, API key, and sender OIN retrieved via `IAppConfig` — NEVER hardcoded
  - On HTTP error: logs real error internally, returns generic error response to controller (ADR-015)

### OciPunchoutService

- [ ] **Task 4.6** — Create `lib/Service/OciPunchoutService.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.6`
  - `initiateSession(string $catalogId): string` — builds OCI session URL for supplier endpoint, returns redirect URL
  - `processPunchoutReturn(array $cxmlData): string` — parses CXML_TO payload, creates PurchaseRequisition with pre-filled CatalogItem data, returns new requisitionId (REQ-INT-002)

### Background jobs

- [ ] **Task 4.7** — Create `lib/BackgroundJob/ContractExpiryNotificationJob.php` (extends `TimedJob`, interval: daily)
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.7`
  - Queries CatalogItems with `validUntil` within 60 days and status = active
  - Sends `NotificationService` alert to category managers per catalog
  - Writes `ProcurementAuditLog` entry per notified item (REQ-CAT-004 Scenario 1)
- [ ] **Task 4.8** — Create `lib/BackgroundJob/MaverickSpendDetectionJob.php` (extends `TimedJob`, interval: weekly)
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.8`
  - Finds PurchaseOrders without contract reference; cross-references by supplier + category against active Contracts; creates `MaverickSpendAlert` objects (REQ-PRQ-003 Scenario 1)
- [ ] **Task 4.9** — Create `lib/BackgroundJob/MaverickSpendMonthlyReportJob.php` (extends `TimedJob`, interval: monthly on 1st)
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-4.9`
  - Aggregates maverick spend by department, sends email notification to department managers above threshold configured in IAppConfig (REQ-PRQ-003 Scenario 3)

---

## 5. API controllers

- [ ] **Task 5.1** — Create `lib/Controller/CatalogController.php`
  - `@spec openspec/changes/catalog-purchase-management/tasks.md#task-5.1`
  - Routes: `GET /api/catalogs`, `POST /api/catalogs`, `GET /api/catalogs/{id}`, `PUT /api/catalogs/{id}`, `DELETE /api/catalogs/{id}`
  - Additional: `POST /api/catalogs/{id}/bulk-price-update` (delegates to CatalogService::bulkUpdatePrices)
  - All mutation endpoints check `IGroupManager::isAdmin()` or appropriate role — NEVER rely on frontend auth (ADR-005)
  - Controllers: thin (<10 lines/method) — routing + validation + response only (ADR-003)
  - Error responses: static generic messages, no `$e->getMessage()` in JSONResponse (ADR-015)
- [ ] **Task 5.2** — Create `lib/Controller/CatalogItemController.php`
  - Routes: `GET /api/catalog-items`, `POST /api/catalog-items`, `GET /api/catalog-items/{id}`, `PUT /api/catalog-items/{id}`, `DELETE /api/catalog-items/{id}`
  - Additional: `POST /api/catalog-items/{id}/deactivate` (delegates to CatalogService::deactivateCatalogItem)
  - `GET /api/catalog-items/{id}/statistics?from=&to=` (delegates to CatalogService::getUsageStatistics)
- [ ] **Task 5.3** — Create `lib/Controller/RequisitionController.php`
  - Routes: `GET /api/requisitions`, `POST /api/requisitions`, `GET /api/requisitions/{id}`, `PUT /api/requisitions/{id}`, `DELETE /api/requisitions/{id}`
  - Additional: `POST /api/requisitions/{id}/submit`, `POST /api/requisitions/{id}/approve`, `POST /api/requisitions/{id}/reject`
- [ ] **Task 5.4** — Create `lib/Controller/BlanketOrderController.php`
  - Routes: `GET /api/blanket-orders`, `POST /api/blanket-orders`, `GET /api/blanket-orders/{id}`, `PUT /api/blanket-orders/{id}`, `DELETE /api/blanket-orders/{id}`
- [ ] **Task 5.5** — Create `lib/Controller/StatementOfWorkController.php`
  - Routes: `GET /api/statements-of-work`, `POST /api/statements-of-work`, `GET /api/statements-of-work/{id}`, `PUT /api/statements-of-work/{id}`, `DELETE /api/statements-of-work/{id}`
  - Additional: `POST /api/statements-of-work/{id}/milestones/{milestoneIndex}/complete`
- [ ] **Task 5.6** — Create `lib/Controller/AuditLogController.php`
  - Routes: `GET /api/audit-log`, `GET /api/audit-log/{id}` (read-only — no POST/PUT/DELETE)
  - Requires admin role for access (ADR-005)
- [ ] **Task 5.7** — Create `lib/Controller/PunchoutController.php`
  - Routes: `GET /api/punchout/{catalogId}/initiate` (returns redirect URL), `POST /api/punchout/return` (public, `#[PublicPage] #[NoCSRFRequired]`, CORS OPTIONS registered)
- [ ] **Task 5.8** — Create `lib/Controller/DigiInkoopController.php`
  - Routes: `POST /api/digiinkoop/dispatch/{requisitionId}` (delegates to DigiInkooopOrderService::dispatch)
- [ ] **Task 5.9** — Register all routes in `appinfo/routes.php`: specific routes BEFORE any wildcard `{slug}` catch-all routes (ADR-003)

---

## 6. Frontend store

- [ ] **Task 6.1** — Register all entity types in `src/store/store.js` via `initializeStores()`:
  - `procurementCatalog` → `objectStore.registerObjectType('procurement-catalog', 'ProcurementCatalog', 'shillinq')`
  - `catalogItem` → `objectStore.registerObjectType('catalog-item', 'CatalogItem', 'shillinq')`
  - `pricingRule` → `objectStore.registerObjectType('pricing-rule', 'PricingRule', 'shillinq')`
  - `purchaseRequisition` → `objectStore.registerObjectType('purchase-requisition', 'PurchaseRequisition', 'shillinq')`
  - `blanketPurchaseOrder` → `objectStore.registerObjectType('blanket-purchase-order', 'BlanketPurchaseOrder', 'shillinq')`
  - `statementOfWork` → `objectStore.registerObjectType('statement-of-work', 'StatementOfWork', 'shillinq')`
  - `procurementAuditLog` → `objectStore.registerObjectType('procurement-audit-log', 'ProcurementAuditLog', 'shillinq')`
  - Type names: kebab-case — NEVER camelCase; each entity registered exactly once (ADR-015)
- [ ] **Task 6.2** — Each object store uses `createObjectStore` with plugins: `files`, `auditTrails`, `relations` (ADR-004)

---

## 7. Frontend router

- [ ] **Task 7.1** — Add named routes to `src/router/index.js` (flat, no nesting; history mode; `generateUrl` base):
  - `{ name: 'Dashboard', path: '/' }` → `DashboardPage`
  - `{ name: 'CatalogList', path: '/catalogs' }` → `CatalogListPage`
  - `{ name: 'CatalogDetail', path: '/catalogs/:id', props: r => ({ catalogId: r.params.id }) }` → `CatalogDetailPage`
  - `{ name: 'CatalogItemList', path: '/catalog-items' }` → `CatalogItemListPage`
  - `{ name: 'CatalogItemDetail', path: '/catalog-items/:id', props: r => ({ itemId: r.params.id }) }` → `CatalogItemDetailPage`
  - `{ name: 'RequisitionList', path: '/requisitions' }` → `RequisitionListPage`
  - `{ name: 'RequisitionDetail', path: '/requisitions/:id', props: r => ({ requisitionId: r.params.id }) }` → `RequisitionDetailPage`
  - `{ name: 'BlanketOrderList', path: '/blanket-orders' }` → `BlanketOrderListPage`
  - `{ name: 'BlanketOrderDetail', path: '/blanket-orders/:id', props: r => ({ orderId: r.params.id }) }` → `BlanketOrderDetailPage`
  - `{ name: 'StatementOfWorkList', path: '/statements-of-work' }` → `StatementOfWorkListPage`
  - `{ name: 'StatementOfWorkDetail', path: '/statements-of-work/:id', props: r => ({ sowId: r.params.id }) }` → `StatementOfWorkDetailPage`
  - `{ name: 'AuditLog', path: '/audit-log' }` → `AuditLogPage`
  - `{ name: 'Settings', path: '/settings' }` → `SettingsPage`
  - Catch-all `*` redirects to `/`

---

## 8. Frontend pages and components

### Dashboard

- [ ] **Task 8.1** — Create `src/views/DashboardPage.vue`
  - `CnDashboardPage` with 4 `CnStatsBlock` KPI cards: Open Requisitions, Pending Approvals, Active Catalogs, Maverick Spend %
  - `CnChartWidget` (bar type) for monthly spend trend — last 12 months
  - Fetch all collections in parallel via `Promise.all` in `useDashboardView` (ADR-004)
  - SPDX header: `<!-- SPDX-License-Identifier: EUPL-1.2 -->` (ADR-015)
  - ALL user-visible strings via `this.t(appName, '...')` — no hardcoded Dutch or English strings

### Catalog pages

- [ ] **Task 8.2** — Create `src/views/CatalogListPage.vue` — `CnIndexPage` + `useListView('procurement-catalog', { sidebarState, objectStore })`; row click → `CatalogDetail`; Add button → `/catalogs/new`
- [ ] **Task 8.3** — Create `src/views/CatalogDetailPage.vue` — `CnDetailPage`; `CnDetailCard` for catalog items list (related items table); `CnObjectSidebar` with audit trail and files tabs; `isNew = catalogId === 'new'`
- [ ] **Task 8.4** — Create `src/views/CatalogItemListPage.vue` — `CnIndexPage` + `useListView`; filter by catalog, status, price range via `CnFilterBar`
- [ ] **Task 8.5** — Create `src/views/CatalogItemDetailPage.vue` — `CnDetailPage`; `CnDetailCard` for linked pricing rules; usage statistics section with date range picker

### Requisition pages

- [ ] **Task 8.6** — Create `src/views/RequisitionListPage.vue` — `CnIndexPage` + `useListView`; filter by status, requester, department; status shown via `CnStatusBadge`
- [ ] **Task 8.7** — Create `src/views/RequisitionDetailPage.vue` — `CnDetailPage`; `CnTimelineStages` showing Draft → Submitted → Approved → Ordered workflow stages; approval history in `CnDetailCard`; Submit/Approve/Reject action buttons visible per current status and user role

### Blanket order pages

- [ ] **Task 8.8** — Create `src/views/BlanketOrderListPage.vue` — `CnIndexPage` + `useListView`
- [ ] **Task 8.9** — Create `src/views/BlanketOrderDetailPage.vue` — `CnDetailPage`; `CnProgressBar` showing `consumedAmount / totalAuthorizedAmount`; release schedule in `CnDetailCard` table

### Statement of work pages

- [ ] **Task 8.10** — Create `src/views/StatementOfWorkListPage.vue` — `CnIndexPage` + `useListView`
- [ ] **Task 8.11** — Create `src/views/StatementOfWorkDetailPage.vue` — `CnDetailPage`; separate `CnDetailCard` sections for Deliverables and Milestones; milestone completion button triggers `POST /api/statements-of-work/{id}/milestones/{index}/complete`

### Audit log page

- [ ] **Task 8.12** — Create `src/views/AuditLogPage.vue` — `CnIndexPage` (read-only, no Add button); `CnFilterBar` with filters: entityType, actionType, date range; `CnMassExportDialog` for CSV export

### Settings page

- [ ] **Task 8.13** — Create `src/views/SettingsPage.vue`
  - `CnVersionInfoCard` FIRST (ADR-004, ADR-015)
  - `CnRegisterMapping` for shillinq register
  - `CnSettingsSection` for DigiInkoop (endpoint URL, API key, sender OIN, test connection button)
  - `CnSettingsSection` for OCI punchout connections
  - `CnSettingsSection` for maverick spend threshold (configurable euro threshold for department alerts)
  - Load via `GET /api/settings`; save via `POST /api/settings`; re-import button calls `POST /api/settings/load`

### Navigation

- [ ] **Task 8.14** — Update `src/components/MainMenu.vue`: add `NcAppNavigationItem` per route with icon and translation; group Catalog Items under Catalogs using navigation nesting or section header; settings link in `NcAppNavigationSettings` footer

---

## 9. Translations

- [ ] **Task 9.1** — Add all user-visible strings to `l10n/nl.json` (Dutch) — every `t(appName, '...')` key must have a Dutch translation:
  - Navigation labels: Catalogussen, Catalogusartikelen, Inkoopverzoeken, Raambestellingen, Werkverklaringen, Auditlog, Instellingen
  - Status labels: Concept, Ingediend, Goedgekeurd, Afgewezen, Besteld, Actief, Gearchiveerd, Afgerond, Geannuleerd
  - Action labels: Toevoegen, Indienen, Goedkeuren, Afwijzen, Importeren, Exporteren, Deactiveren, Verbinding testen
  - Error messages: static strings for all error responses shown in the UI
- [ ] **Task 9.2** — Add corresponding `l10n/en.json` English translations for all keys
- [ ] **Task 9.3** — Verify no hardcoded Dutch or English strings remain in `.vue` files: grep for bare string literals in templates

---

## 10. Tests

- [ ] **Task 10.1** — Create `tests/Unit/Service/CatalogServiceTest.php` (≥3 test methods): test `bulkUpdatePrices`, `deactivateCatalogItem`, `getUsageStatistics` (ADR-008)
- [ ] **Task 10.2** — Create `tests/Unit/Service/RequisitionServiceTest.php` (≥3 test methods): test `submitRequisition`, `approveRequisition`, `rejectRequisition` with correct status transitions and audit log writes
- [ ] **Task 10.3** — Create `tests/Unit/Service/BlanketOrderServiceTest.php` (≥3 test methods): test `trackConsumption`, `validateSpendLimit` including rejection when limit exceeded
- [ ] **Task 10.4** — Create `tests/Unit/Service/AuditLogServiceTest.php` (≥3 test methods): test entry creation, verify `timestamp` is server-set, verify actor is `getUID()` not `getDisplayName()`
- [ ] **Task 10.5** — Create `tests/Unit/Controller/CatalogControllerTest.php` (≥3 test methods): test list, get, create endpoints; verify mutation endpoints check authorization
- [ ] **Task 10.6** — Create `tests/integration/catalog-purchase-management.postman_collection.json` with Newman test collection covering:
  - CRUD for each entity type (create, read, update, delete)
  - Bulk price update flow
  - Requisition submit → approve lifecycle
  - Audit log write verification on status change
  - Punchout return endpoint (public page)
- [ ] **Task 10.7** — Create Playwright browser tests verifying GIVEN/WHEN/THEN scenarios for:
  - REQ-CAT-001 (create catalog, activate catalog)
  - REQ-PRQ-001 (create draft requisition, submit)
  - REQ-PRQ-002 (approve, reject with reason, timeline display)
  - REQ-AUD-001 (audit log entry appears after status change)
  - REQ-BPO-001 (create BPO, consumption tracking)
  - REQ-DSH-001 (dashboard KPI blocks load)
  - Accessibility: keyboard navigation through requisition creation flow (WCAG AA, ADR-010)

---

## 11. SPDX headers and licensing

- [ ] **Task 11.1** — Verify every new PHP file has `// SPDX-License-Identifier: EUPL-1.2` on the line after `<?php` (ADR-014, ADR-015)
- [ ] **Task 11.2** — Verify every new Vue file has `<!-- SPDX-License-Identifier: EUPL-1.2 -->` as the first line
- [ ] **Task 11.3** — Verify every new JS file has `// SPDX-License-Identifier: EUPL-1.2` as the first line
- [ ] **Task 11.4** — Run `grep -rL 'SPDX-License-Identifier' src/ lib/ --include='*.php' --include='*.vue' --include='*.js'` — output must be empty

---

## 12. Metrics and health endpoints

- [ ] **Task 12.1** — Add metrics to `lib/Controller/MetricsController.php` (or create if missing) — Prometheus text format, admin auth required (ADR-006):
  - `shillinq_active_catalogs_total` — count of ProcurementCatalogs with status = active
  - `shillinq_open_requisitions_total` — count of PurchaseRequisitions with status = submitted
  - `shillinq_blanket_orders_active_total` — count of BlanketPurchaseOrders with status = active
  - `shillinq_audit_log_entries_total` — total count of ProcurementAuditLog entries
- [ ] **Task 12.2** — Verify `GET /api/health` verifies OpenRegister connectivity and responds within 2 seconds (ADR-006)

---

## 13. Documentation

- [ ] **Task 13.1** — Create `docs/catalog-management.md` covering: creating a catalog, adding items, bulk import, bulk price update, contract expiry notifications (ADR-009)
- [ ] **Task 13.2** — Create `docs/purchase-requisitions.md` covering: creating a requisition, approval workflow, timeline view, audit trail (ADR-009)
- [ ] **Task 13.3** — Create `docs/blanket-orders.md` covering: creating a BPO, consumption tracking, release schedule (ADR-009)
- [ ] **Task 13.4** — Create `docs/integrations.md` covering: DigiInkoop setup and dispatch, OCI punchout configuration and session flow (ADR-009)

---

## 14. Pre-commit verification

- [ ] **Task 14.1** — Run all pre-commit checks from ADR-015 before final commit:
  1. SPDX headers: zero files missing (Task 11.4)
  2. ObjectService calls: all use 3 positional args `($register, $schema, $idOrParams)`
  3. Error responses: zero `$e->getMessage()` in JSONResponse
  4. Auth checks: every POST/PUT/DELETE method has backend authorization check
  5. Store registration: each entity registered exactly once, kebab-case names
  6. `npm run lint`: zero errors
  7. Translations: zero hardcoded strings in templates
  8. try/catch: every `await store.action()` wrapped in try/catch with user feedback
  9. No raw `fetch()`: all API calls via `@nextcloud/axios`
  10. Import source: zero `from '@nextcloud/vue'` — must use `@conduction/nextcloud-vue`
  11. Component imports: every `<NcFoo>` and `<CnFoo>` imported AND listed in `components: {}`
  12. Type slug consistency: zero camelCase entity type strings in store, search, routes, or views
  13. Route consistency: every entity type has a matching named route
  14. Tasks completeness: every `[x]` task is fully implemented, not a stub
