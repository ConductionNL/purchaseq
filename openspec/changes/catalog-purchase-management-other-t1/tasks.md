# Tasks: Catalog & Purchase Management — Shillinq

## Deduplication Check

- [ ] DED-001: Verify no overlap with `ObjectService`, `RegisterService`, `SchemaService`,
  `ConfigurationService` for CRUD, bulk import/export, search, and audit trail operations.
  Document: CRUD via `ObjectService.saveObject()`/`findAll()`/`deleteObject()` — no rebuild.
  Import via `ImportService` + `CnMassImportDialog` — no rebuild. Search via `IndexService` +
  `CnFacetSidebar` — no rebuild. Audit via `AuditTrailService` (automatic) — no rebuild.
  Custom logic required: `ThreeWayMatchService`, `ApprovalRoutingService`,
  `ProcurementThresholdService` — no equivalent in platform. Finding: no duplication.

---

## 1. Seed Data

- [ ] 1.1 Add `ProcurementCatalog` seed objects (3) to `lib/Settings/shillinq_register.json`
  using `@self` envelope with Dutch municipality data per design.md
- [ ] 1.2 Add `CatalogItem` seed objects (5) with realistic Dutch SKUs, prices (EUR),
  and lead times referencing the 3 catalogs above
- [ ] 1.3 Add `PurchaseRequisition` seed objects (3) covering approved, pending-approval,
  and draft statuses
- [ ] 1.4 Add `PurchaseOrder` seed objects (3) referencing the requisitions above,
  with `matchStatus` values: matched, pending, pending
- [ ] 1.5 Add `BlanketPurchaseOrder` seed objects (2) with `releasedAmount` and
  `consumedAmount` populated to show consumption progress
- [ ] 1.6 Add `PricingRule` seed objects (3): volume discount, fixed-price framework,
  early-payment discount
- [ ] 1.7 Verify idempotency: re-importing seed data via `ConfigurationService::importFromApp()`
  must not create duplicates (slug matching per ADR-001)

---

## 2. Backend — Services

- [ ] 2.1 Create `lib/Service/ThreeWayMatchService.php`
  - `match(string $purchaseOrderId): array` — compares PO × GoodsReceipt × VendorBill
    line by line within 0.01 EUR tolerance
  - Returns `['status' => 'matched'|'partial'|'disputed', 'lines' => [...]]`
  - Called automatically when `GoodsReceipt` or `VendorBill` is linked to a PO
  - `@spec openspec/changes/catalog-purchase-management-other-t1/tasks.md#task-2.1`

- [ ] 2.2 Create `lib/Service/ApprovalRoutingService.php`
  - `resolveChain(string $requisitionId): ApprovalChain` — selects chain by category,
    amount, and organisation
  - `createApprovalRequests(string $requisitionId, ApprovalChain $chain): void` — creates
    `ApprovalRequest` records and dispatches notifications via `NotificationService`
  - `processApproval(string $approvalRequestId, string $decision, string $comment): void`
  - `@spec openspec/changes/catalog-purchase-management-other-t1/tasks.md#task-2.2`

- [ ] 2.3 Create `lib/Service/ProcurementThresholdService.php`
  - `evaluate(float $amount, string $category, string $organisationType): ProcurementProcedure`
    — compares amount to EU thresholds stored in `ProcurementProcedure` objects
  - `getThresholds(): array` — returns current threshold table from OpenRegister
  - `@spec openspec/changes/catalog-purchase-management-other-t1/tasks.md#task-2.3`

---

## 3. Backend — Controllers

- [ ] 3.1 Create `lib/Controller/CatalogController.php`
  - `resolvePrice(string $catalogItemId, float $quantity, ?string $promoCode): array`
    — applies `PricingRule` precedence (supplier > category > global)
  - `applyPromoCode(string $lineId, string $promoCode): array` — validates and applies
    promotion code discount; returns HTTP 422 if expired or invalid
  - `validateCostAllocation(array $lines): void` — asserts each line's cost allocation
    percentages sum to 100%; returns HTTP 422 if not
  - Annotate all methods with `@spec` PHPDoc tags per ADR-003
  - `@spec openspec/changes/catalog-purchase-management-other-t1/tasks.md#task-3.1`

- [ ] 3.2 Create `lib/Controller/ProcurementController.php`
  - `createOrderFromRequisition(string $requisitionId): JsonResponse` — creates
    `PurchaseOrder` pre-filled from approved `PurchaseRequisition`
  - `triggerMatch(string $purchaseOrderId): JsonResponse` — calls `ThreeWayMatchService`
    and updates `matchStatus`; POST `/api/purchase-orders/{id}/match`
  - `createSpendTransaction(string $purchaseOrderId): void` — creates `SpendTransaction`
    when match status becomes `matched`
  - `createCallOff(string $blanketPoId, array $data): JsonResponse` — creates
    `CallOffOrder` and increments `BlanketPurchaseOrder.releasedAmount`; validates
    headroom before save
  - Annotate all methods with `@spec` PHPDoc tags per ADR-003
  - `@spec openspec/changes/catalog-purchase-management-other-t1/tasks.md#task-3.2`

- [ ] 3.3 Register new routes in `appinfo/routes.php` (specific routes before wildcard):
  - `GET/POST /api/procurement-catalogs` + `GET/PUT/DELETE /api/procurement-catalogs/{id}`
  - `GET/POST /api/catalog-items` + `GET/PUT/DELETE /api/catalog-items/{id}`
  - `GET /api/catalog-items/{id}/resolve-price`
  - `GET/POST /api/purchase-requisitions` + `GET/PUT/DELETE /api/purchase-requisitions/{id}`
  - `POST /api/purchase-requisitions/{id}/submit`
  - `POST /api/purchase-requisitions/{id}/approve`
  - `POST /api/purchase-requisitions/{id}/reject`
  - `GET/POST /api/purchase-orders` + `GET/PUT/DELETE /api/purchase-orders/{id}`
  - `POST /api/purchase-orders/{id}/match`
  - `POST /api/purchase-orders/{id}/create-from-requisition`
  - `GET/POST /api/blanket-purchase-orders` + `GET/PUT/DELETE /api/blanket-purchase-orders/{id}`
  - `POST /api/blanket-purchase-orders/{id}/call-offs`
  - `GET/POST /api/pricing-rules` + `GET/PUT/DELETE /api/pricing-rules/{id}`
  - `GET/POST /api/rate-cards` + `GET/PUT/DELETE /api/rate-cards/{id}`

- [ ] 3.4 Register `CatalogController` and `ProcurementController` in DI container in
  `lib/AppInfo/Application.php` using constructor injection with `private readonly`

---

## 4. Frontend — Object Stores

- [ ] 4.1 Create `src/store/modules/procurementCatalogStore.js` using `createObjectStore`
  with `files`, `auditTrails`, `relations` plugins
- [ ] 4.2 Create `src/store/modules/catalogItemStore.js` using `createObjectStore`
  with `files`, `auditTrails`, `relations`, `search` plugins
- [ ] 4.3 Create `src/store/modules/purchaseRequisitionStore.js` using `createObjectStore`
  with `files`, `auditTrails`, `relations`, `lifecycle` plugins
- [ ] 4.4 Create `src/store/modules/purchaseOrderStore.js` using `createObjectStore`
  with `files`, `auditTrails`, `relations`, `lifecycle` plugins
- [ ] 4.5 Create `src/store/modules/blanketPurchaseOrderStore.js` using `createObjectStore`
  with `auditTrails`, `relations` plugins
- [ ] 4.6 Create `src/store/modules/pricingRuleStore.js` using `createObjectStore`
- [ ] 4.7 Create `src/store/modules/spendTransactionStore.js` using `createObjectStore`
- [ ] 4.8 Register all stores in `src/store/store.js` via
  `objectStore.registerObjectType(name, schemaSlug, registerSlug)` inside `initializeStores()`

---

## 5. Frontend — Index Pages

- [ ] 5.1 Create `src/views/ProcurementCatalogsIndex.vue`
  - `CnIndexPage` with `useListView(procurementCatalog, { sidebarState, objectStore })`
  - Columns: name, status, validFrom, validUntil, item count
  - Add button → router push to `ProcurementCatalogDetail` with id='new'

- [ ] 5.2 Create `src/views/CatalogItemsIndex.vue`
  - `CnIndexPage` with `CnFacetSidebar` for category, supplier, status facets
  - Columns: name, SKU, unitPrice (formatted currency), supplier, status badge
  - Row click → `CatalogItemDetail`
  - Bulk import via `CnMassImportDialog` | Bulk export via `CnMassExportDialog`

- [ ] 5.3 Create `src/views/PurchaseRequisitionsIndex.vue`
  - `CnIndexPage` with `useListView`
  - Columns: requisitionNumber, title, status badge, totalAmount, requestedBy, requiredByDate
  - Filter by status (draft / submitted / pending_approval / approved / rejected)
  - Row click → `PurchaseRequisitionDetail`

- [ ] 5.4 Create `src/views/PurchaseOrdersIndex.vue`
  - `CnIndexPage` with `useListView`
  - Columns: orderNumber, supplier, totalAmount, status, matchStatus badge, orderDate
  - `matchStatus` badge: matched=green, partial=orange, disputed=red, pending=grey
  - Filter by matchStatus

- [ ] 5.5 Create `src/views/BlanketPurchaseOrdersIndex.vue`
  - `CnIndexPage` with `useListView`
  - Columns: orderNumber, supplier, maxAmount, releasedAmount, consumedAmount, status
  - Inline `CnProgressBar` per row showing `consumedAmount / maxAmount`

---

## 6. Frontend — Detail Pages

- [ ] 6.1 Create `src/views/PurchaseRequisitionDetail.vue`
  - Header: title, status badge, Edit + Delete buttons
  - `CnDetailCard` sections: General info, Line items table, Cost allocation, Timeline
  - `CnTimelineStages` showing approval step status with approver names
  - `CnObjectSidebar` with Files, Notes, Audit Trail tabs

- [ ] 6.2 Create `src/views/PurchaseOrderDetail.vue`
  - Header: orderNumber, matchStatus badge, Edit + Delete + "Versturen" + "Match uitvoeren" buttons
  - `CnDetailCard` sections: General info, Line items (with per-line delivery location),
    Linked requisition, Linked `GoodsReceipt`, Linked `VendorBill`
  - Match status section showing per-line match result (matched / disputed)
  - `CnObjectSidebar` with Files, Audit Trail tabs

- [ ] 6.3 Create `src/views/CatalogItemDetail.vue`
  - `CnDetailPage` with `CnDetailCard` for: item info, pricing rules table,
    component items (for kits), supplier info
  - `CnObjectSidebar` with Files (datasheets), Audit Trail tabs

- [ ] 6.4 Create `src/views/BlanketPurchaseOrderDetail.vue`
  - `CnDetailPage` with: general info card, consumption `CnProgressBar` with amounts,
    `CallOffOrder` table with status and amounts
  - "Vrijgave aanmaken" button → `CnFormDialog` for new `CallOffOrder`

- [ ] 6.5 Create `src/views/SpendAnalysisDashboard.vue`
  - `CnDashboardPage` with GridStack layout
  - 4 `CnStatsBlock` KPI cards: total spend YTD, matched PO count, disputes, budget %
  - Bar chart `CnChartWidget`: spend by `SpendCategory`
  - Donut chart `CnChartWidget`: procedure type distribution
  - Filter: fiscal year, `ProcurementCategory`, organisation

---

## 7. Frontend — Navigation

- [ ] 7.1 Add navigation items to `src/components/MainMenu.vue`:
  - "Catalogi" → `/procurement-catalogs`
  - "Catalogusartikelen" → `/catalog-items`
  - "Inkoopverzoeken" → `/purchase-requisitions`
  - "Inkooporders" → `/purchase-orders`
  - "Raambestellingen" → `/blanket-purchase-orders`
  - "Bestedingsanalyse" → `/spend-analysis`

- [ ] 7.2 Add named routes to `src/router/index.js` for all new pages (flat, no nesting):
  - `ProcurementCatalogsIndex`, `PurchaseRequisitionsIndex`, `PurchaseRequisitionDetail`
  - `PurchaseOrdersIndex`, `PurchaseOrderDetail`
  - `BlanketPurchaseOrdersIndex`, `BlanketPurchaseOrderDetail`
  - `CatalogItemsIndex`, `CatalogItemDetail`
  - `SpendAnalysisDashboard`
  - Catch-all `*` redirects to `/`

- [ ] 7.3 Verify all user-visible strings use `t(appName, 'text')` — no hardcoded Dutch strings
  in Vue components (ADR-007). Add missing keys to `l10n/nl.js` and `l10n/en.js`

---

## 8. Tests

- [ ] 8.1 Create `tests/Unit/Service/ThreeWayMatchServiceTest.php` with ≥3 methods:
  - `testFullMatchReturnsMatched()` — all lines match within tolerance
  - `testPartialReceiptReturnsPartial()` — some lines received, not all
  - `testBillAmountMismatchReturnsDisputed()` — VendorBill deviates > 0.01 EUR

- [ ] 8.2 Create `tests/Unit/Service/ApprovalRoutingServiceTest.php` with ≥3 methods:
  - `testResolvesCorrectChainByCategory()`
  - `testCreatesApprovalRequestsForEachStep()`
  - `testApprovalAdvancesToNextStep()`

- [ ] 8.3 Create `tests/Unit/Service/ProcurementThresholdServiceTest.php` with ≥3 methods:
  - `testAmountBelowThresholdReturnsMeervoudigOnderhands()`
  - `testAmountAboveEUThresholdReturnsOpenbaar()`
  - `testThresholdTableIsLoadedFromSeedData()`

- [ ] 8.4 Create `tests/Unit/Controller/CatalogControllerTest.php` with ≥3 methods:
  - `testResolvePriceAppliesSupplierRuleOverCategoryRule()`
  - `testApplyPromoCodeReturns422WhenExpired()`
  - `testValidateCostAllocationRejects101Percent()`

- [ ] 8.5 Create `tests/Unit/Controller/ProcurementControllerTest.php` with ≥3 methods:
  - `testCreateOrderFromRequisitionPrefillsFromRequisition()`
  - `testTriggerMatchCallsThreeWayMatchService()`
  - `testCreateCallOffRejectsWhenExceedsMaxAmount()`

- [ ] 8.6 Create `tests/integration/catalog-purchase-management.postman_collection.json`
  covering at minimum:
  - `POST /api/catalog-items` (create item)
  - `GET /api/catalog-items/{id}/resolve-price` (price resolution)
  - `POST /api/purchase-requisitions` (create requisition)
  - `POST /api/purchase-requisitions/{id}/submit` (submit for approval)
  - `POST /api/purchase-orders/{id}/match` (trigger three-way match)
  - `POST /api/blanket-purchase-orders/{id}/call-offs` (release call-off; verify 422 over max)

---

## 9. Documentation

- [ ] 9.1 Create `docs/features/catalog-management.md` covering:
  browsing the catalog, bulk import, composite items, user-specific views (with screenshots)
- [ ] 9.2 Create `docs/features/purchase-management.md` covering:
  creating a requisition, approval workflow, creating a PO, three-way matching,
  blanket POs and call-offs, spend analysis (with screenshots)
