# Tasks: catalog-purchase-management-other-t3

## Deduplication Check (ADR-012)

- [ ] DUP-1: Verify `PricingRuleService` has no equivalent in OpenRegister core or shared specs
- [ ] DUP-2: Verify `PurchaseOrderMatchingService` has no equivalent in platform services
- [ ] DUP-3: Verify `CatalogPunchoutService` (cXML) has no equivalent in platform integration registry
- [ ] DUP-4: Verify `RequisitionApprovalService` does not duplicate `WorkflowEngineController` — document coupling
- [ ] DUP-5: Verify `QuoteRevisionService` does not duplicate any platform versioning mechanism

---

## 1. Backend — Catalog Browsing & Punch-out

- [ ] 1.1 Register `catalog-item` and `procurement-catalog` object types in `src/store/store.js` via `createObjectStore`
  - `@spec openspec/changes/catalog-purchase-management-other-t3/tasks.md#task-1`
- [ ] 1.2 Create `lib/Service/CatalogPunchoutService.php` with:
  - `generateSetupRequest(string $catalogId, string $returnUrl): string` — builds cXML PunchOutSetupRequest
  - `receiveOrderMessage(string $cxmlPayload): array` — parses PunchOutOrderMessage, returns normalized basket items
  - SPDX header, `@spec` PHPDoc tag, constructor DI with `private readonly`
- [ ] 1.3 Create `lib/Controller/CatalogPunchoutController.php` with:
  - `setup(string $catalogId): JSONResponse` — calls `generateSetupRequest()`, returns redirect URL
  - `receive(Request $request): Response` — calls `receiveOrderMessage()`, redirects to basket view
  - `#[NoAdminRequired]` on both methods; `#[PublicPage] #[NoCSRFRequired]` on `receive()`
- [ ] 1.4 Register punchout routes in `appinfo/routes.php`:
  - `GET /api/catalog-punchout/{catalogId}/setup`
  - `POST /api/catalog-punchout/receive`
  - Routes registered BEFORE any wildcard `{slug}` catch-all
- [ ] 1.5 Write PHPUnit tests for `CatalogPunchoutService`:
  - `testGenerateSetupRequestReturnsValidCxmlString()`
  - `testReceiveOrderMessageParsesItemsCorrectly()`
  - `testReceiveOrderMessageThrowsOnInvalidXml()`

---

## 2. Backend — Purchase Requisition Workflow

- [ ] 2.1 Create `lib/Service/RequisitionApprovalService.php` with:
  - `submit(string $reqId): ApprovalRequest` — resolves `ApprovalChain`, creates `ApprovalRequest` + `ApprovalTask` objects, dispatches notifications
  - `approve(string $taskId, string $approverId): void` — transitions task + requisition status
  - `reject(string $taskId, string $approverId, string $reason): void` — transitions status, cancels remaining tasks, notifies requester
  - SPDX header, constructor DI, `@spec` PHPDoc on each public method
- [ ] 2.2 Create `lib/Controller/RequisitionApprovalController.php` with:
  - `submit(string $reqId): JSONResponse`
  - `approve(string $taskId): JSONResponse`
  - `reject(string $taskId, Request $request): JSONResponse`
  - `#[NoAdminRequired]` on all methods; backend `IGroupManager` admin check on admin-only actions
- [ ] 2.3 Register approval routes in `appinfo/routes.php`:
  - `POST /api/purchase-requisitions/{reqId}/submit`
  - `POST /api/approval-tasks/{taskId}/approve`
  - `POST /api/approval-tasks/{taskId}/reject`
- [ ] 2.4 Write PHPUnit tests for `RequisitionApprovalService`:
  - `testSubmitCreatesApprovalRequestAndTasks()`
  - `testSubmitDispatchesNotificationToApprovers()`
  - `testSubmitWithNoChainConfiguredTransitionsStatusAndNotifiesAdmin()`
  - `testApproveTransitionsRequisitionStatusWhenAllTasksApproved()`
  - `testRejectCancelsRemainingTasks()`

---

## 3. Backend — Discount / Pricing Engine

- [ ] 3.1 Create `lib/Service/PricingRuleService.php` with:
  - `applyRules(array $orderLines, array $couponCodes, string $customerId): array` — evaluates active `PricingRule` objects in priority order, applies first-match or all-match logic, returns annotated order lines
  - `evaluateConditions(array $conditions, array $lineContext): bool` — evaluates a rule's condition array against a line context (supports operators: eq, neq, gte, lte, in, contains)
  - SPDX header, constructor DI via `ObjectService`, `@spec` PHPDoc
- [ ] 3.2 Write PHPUnit tests for `PricingRuleService`:
  - `testFirstMatchStopsAfterFirstMatchingRule()`
  - `testAllMatchAppliesAllMatchingRules()`
  - `testExpiredRuleIsExcluded()`
  - `testCouponCodeConditionMatches()`
  - `testVolumeThresholdConditionMatches()`
  - `testRulesEvaluatedInPriorityOrder()`

---

## 4. Backend — PO Matching Service

- [ ] 4.1 Create `lib/Service/PurchaseOrderMatchingService.php` with:
  - `match(string $poId, string $mode): MatchResult` — performs 2-way or 3-way matching, stores result as PO metadata
  - Value object `lib/ValueObject/MatchResult.php` with `status`, `discrepancies[]`, `mode`, `matchedAt`
  - Reads tolerance config from `Administration.poMatchingConfig` via `ObjectService.findObject()`
  - Dispatches `NotificationService` on discrepancy
  - SPDX header, constructor DI, `@spec` PHPDoc
- [ ] 4.2 Create `lib/Controller/PurchaseOrderMatchingController.php`:
  - `match(string $poId, Request $request): JSONResponse` — triggers matching, returns result
  - `#[NoAdminRequired]`; admin check via `IGroupManager` for configuring tolerance rules
- [ ] 4.3 Register matching route in `appinfo/routes.php`:
  - `POST /api/purchase-orders/{poId}/match`
- [ ] 4.4 Write PHPUnit tests for `PurchaseOrderMatchingService`:
  - `testTwoWayMatchSucceedsWithinTolerance()`
  - `testTwoWayMatchFailsOnPriceDiscrepancy()`
  - `testThreeWayMatchFailsOnQuantityDiscrepancy()`
  - `testToleranceRulesAppliedFromAdministrationConfig()`
  - `testNotificationDispatchedOnDiscrepancy()`

---

## 5. Backend — Sale Order Sequencing & Stock Sourcing

- [ ] 5.1 Create `lib/Service/SaleOrderSequenceService.php`:
  - `reorder(string $orderId, array $lineIdSequenceMap): void` — updates `sequence` field on each PO line item via `ObjectService.saveObject()`
  - SPDX header, `@spec` PHPDoc
- [ ] 5.2 Create `lib/Service/SaleStockSourcingService.php`:
  - `resolveSource(string $catalogItemId, int $quantity): SourcingResult` — looks up sourcing rules from `IAppConfig`, returns preferred supplier + lead time
  - Value object `lib/ValueObject/SourcingResult.php` with `supplier`, `warehouse`, `leadTimeDays`, `inStock`
  - SPDX header, `@spec` PHPDoc
- [ ] 5.3 Register routes:
  - `PUT /api/purchase-orders/{poId}/reorder-lines`
  - `GET /api/catalog-items/{itemId}/sourcing`
- [ ] 5.4 Write PHPUnit tests:
  - `testReorderUpdatesSequenceOnAllLines()`
  - `testResolveSourceReturnsSourcingResult()`
  - `testResolveSourceReturnsNullsWhenNoRuleConfigured()`

---

## 6. Backend — Quotation Management

- [ ] 6.1 Create `lib/Service/QuoteRevisionService.php`:
  - `createRevision(string $quoteId): array` — copies Quote object, increments `revisionNumber`, sets `previousVersion` relation, transitions original status to `herzien`
  - SPDX header, constructor DI via `ObjectService`, `@spec` PHPDoc
- [ ] 6.2 Create `lib/BackgroundJob/QuoteExpiryJob.php`:
  - Extends `\OC\BackgroundJob\TimedJob` (runs daily)
  - Finds all `Quote` objects with `status: "verzonden"` and `expiryDate < today`
  - Transitions each to `status: "verlopen"` and dispatches requester notification
  - SPDX header, `@spec` PHPDoc
- [ ] 6.3 Register `QuoteExpiryJob` in `lib/AppInfo/Application.php`
- [ ] 6.4 Create `lib/Controller/QuoteRevisionController.php`:
  - `createRevision(string $quoteId): JSONResponse`
  - `#[NoAdminRequired]`
- [ ] 6.5 Register route: `POST /api/quotes/{quoteId}/revise`
- [ ] 6.6 Write PHPUnit tests for `QuoteRevisionService`:
  - `testCreateRevisionIncrementsRevisionNumber()`
  - `testCreateRevisionSetsPreviousVersionRelation()`
  - `testCreateRevisionTransitionsOriginalStatusToHerzien()`
- [ ] 6.7 Write PHPUnit tests for `QuoteExpiryJob`:
  - `testExpiredQuotesTransitionedToVerlopen()`
  - `testNonExpiredQuotesNotAffected()`

---

## 7. Frontend — Catalog Views

- [ ] 7.1 Create `src/views/CatalogIndex.vue`:
  - Uses `CnIndexPage` with `useListView('catalog-item', { sidebarState, objectStore })`
  - `CnFacetSidebar` with facets: category, catalog, classificationScheme, stockStatus, price range
  - Inline pricing with discount badge via `CnCellRenderer`
  - "Toevoegen aan aanvraag" row action
  - SPDX header, all strings via `t(appName, 'key')`, `@nextcloud/axios` for mutations
- [ ] 7.2 Create `src/views/CatalogDetail.vue`:
  - Uses `CnDetailPage` + `CnDetailCard` sections: Productinformatie, Classificatie, Voorraad & Levering
  - Quantity input with `minimumOrderQuantity` validation
  - "Toevoegen aan aanvraag" + "Punch-out openen" (conditional on `punchoutUrl`) buttons
  - `CnObjectSidebar` with Files and Audit tabs
  - SPDX header, all strings via `t()`
- [ ] 7.3 Create `src/store/modules/basketStore.js`:
  - Transient Pinia store (not backed by OpenRegister): `items[]`, `costCenter`, `addItem()`, `removeItem()`, `clearBasket()`, `itemCount` (computed)
  - SPDX header
- [ ] 7.4 Create `src/views/BasketCheckout.vue`:
  - Lists basket items with quantity inputs and pricing
  - Cost center picker (dropdown from `CostCenter` object store)
  - Coupon code input — calls `POST /api/pricing-rules/apply` and updates displayed prices
  - "Aanvraag indienen" button — POSTs to create `PurchaseRequisition` and clears basket
  - SPDX header, all strings via `t()`
- [ ] 7.5 Register Catalog routes in `src/router/index.js`:
  - `/catalog` → `CatalogIndex` (named: `CatalogIndex`)
  - `/catalog/:id` → `CatalogDetail` (named: `CatalogDetail`, props: `catalogItemId`)
  - `/catalog/basket` → `BasketCheckout` (named: `BasketCheckout`)
  - Registered BEFORE `/:id` catch-all

---

## 8. Frontend — Purchase Requisition Views

- [ ] 8.1 Create `src/views/PurchaseRequisitionIndex.vue`:
  - `CnIndexPage` with `useListView('purchase-requisition')`
  - Status filter facet with `CnStatusBadge` per status
  - Row click → `PurchaseRequisitionDetail`
  - SPDX header, translations
- [ ] 8.2 Create `src/views/PurchaseRequisitionDetail.vue`:
  - View mode: `CnDetailPage` + `CnDetailCard` for: Aanvraaggegevens, Regelitems, Goedkeuringsstatus, Aangepaste velden
  - Edit mode: `CnAdvancedFormDialog` rendering custom fields from `ProcurementCategory.formTemplate`
  - Header actions: Indienen / Intrekken (status-conditional), Edit, Delete
  - `CnObjectSidebar` with Files, Notes, Tasks, Audit tabs
  - SPDX header, translations
- [ ] 8.3 Create `src/views/PurchaseOrderDetail.vue` (extended):
  - Add `CnTimelineStages` for PO status progression (Concept → Verzonden → Bevestigd → In Levering → Ontvangen → Gefactureerd → Betaald)
  - Add PO matching result section: status badge + discrepancy table
  - Add "Matching uitvoeren" button (2-way / 3-way selector)
  - Add stock sourcing indicator per line item
  - SPDX header, translations
- [ ] 8.4 Register Requisition routes: `/purchase-requisitions`, `/purchase-requisitions/:id`

---

## 9. Frontend — Discount Engine Admin UI

- [ ] 9.1 Create `src/views/PricingRuleIndex.vue`:
  - `CnIndexPage` with `useListView('pricing-rule')`
  - Columns: name, discountType, discountValue, executionMode, priority, status, validUntil
  - SPDX header, translations
- [ ] 9.2 Register route: `/pricing-rules`, `/pricing-rules/:id`
- [ ] 9.3 Add Kortingsregels navigation item to `src/components/MainMenu.vue`

---

## 10. Frontend — Quotation Views

- [ ] 10.1 Create `src/views/QuoteIndex.vue`:
  - `CnIndexPage` with `useListView('quote')`
  - `CnStatusBadge` per status with correct colour mapping: concept=grijs, verzonden=blauw, herzien=oranje, geaccepteerd=groen, afgewezen=rood, verlopen=grijs
  - SPDX header, translations
- [ ] 10.2 Create `src/views/QuoteDetail.vue`:
  - `CnDetailPage` + `CnDetailCard` sections: Offertegegevens, Regelitems, Revisiehistorie
  - `CnTimelineStages` for status progression
  - Header actions: "Nieuwe revisie maken" (hidden if status is terminal), "PDF genereren", Edit, Delete
  - Revisiehistorie `CnDetailCard` lists linked `previousVersion` chain with revision number and date
  - `CnObjectSidebar` with Files (PDF attachment visible here), Audit tabs
  - SPDX header, translations
- [ ] 10.3 Register routes: `/quotes`, `/quotes/:id`

---

## 11. Register Templates & Seed Data

- [ ] 11.1 Add seed data objects to `lib/Settings/shillinq_register.json` per design.md:
  - 3 `ProcurementCatalog` objects
  - 4 `CatalogItem` objects
  - 3 `PurchaseRequisition` objects
  - 4 `PricingRule` objects
  - 3 `Quote` objects
  - Use Dutch values, `@self` envelope with `register: "shillinq"`

---

## 12. Navigation & Settings

- [ ] 12.1 Add navigation items to `src/components/MainMenu.vue`:
  - Catalogus → `/catalog`
  - Inkoopverzoeken → `/purchase-requisitions`
  - Offertes → `/quotes`
  - Kortingsregels → `/pricing-rules`
- [ ] 12.2 Register all new object types in `src/store/store.js`:
  - `createObjectStore('catalog-item')` with `filesPlugin`, `auditTrailsPlugin`, `relationsPlugin`
  - `createObjectStore('procurement-catalog')` with `relationsPlugin`
  - `createObjectStore('purchase-requisition')` with `filesPlugin`, `auditTrailsPlugin`, `relationsPlugin`, `lifecyclePlugin`
  - `createObjectStore('pricing-rule')` with `auditTrailsPlugin`
  - `createObjectStore('quote')` with `filesPlugin`, `auditTrailsPlugin`, `relationsPlugin`, `lifecyclePlugin`
  - All type names: kebab-case, registered exactly ONCE

---

## 13. i18n Translations

- [ ] 13.1 Add all new user-visible strings to `l10n/nl.json` (Dutch) and `l10n/en.json` (English):
  - Navigation labels, button labels, status labels, error messages, notification texts
  - Verify no hardcoded Dutch or English strings remain in Vue templates

---

## 14. Tests — Integration & Browser

- [ ] 14.1 Add Newman/Postman API tests to `tests/integration/` for each new endpoint:
  - `catalog-punchout-setup`, `purchase-requisition-submit`, `approval-approve`, `po-match`, `quote-revise`
- [ ] 14.2 Add Playwright browser tests for each REQ scenario marked GIVEN/WHEN/THEN in specs:
  - At minimum: REQ-CAT-001 (browse catalog), REQ-REQ-001 (create requisition), REQ-DIS-001 (apply discount), REQ-POM-001 (2-way match), REQ-QUO-001 (create quote)

---

## 15. Pre-commit Verification

- [ ] 15.1 Run SPDX header check: `grep -rL 'SPDX-License-Identifier' src/ lib/ --include='*.php' --include='*.vue' --include='*.js'` — fix all missing headers
- [ ] 15.2 Run ObjectService call check: all `findObject`/`saveObject`/`findObjects` calls have 3 positional args
- [ ] 15.3 Run error response check: no `$e->getMessage()` in JSONResponse in `lib/Controller/`
- [ ] 15.4 Verify every POST/PUT/DELETE controller method has backend auth check
- [ ] 15.5 Run `npm run lint && npm run stylelint` — fix all ESLint and Stylelint errors
- [ ] 15.6 Run `composer check:strict` — all PHP lint, PHPCS, Psalm, PHPStan, and unit tests pass
- [ ] 15.7 Verify store registration: all entity types kebab-case, registered exactly once in `store.js`
- [ ] 15.8 Verify no imports from `@nextcloud/vue` directly — must be `@conduction/nextcloud-vue`
- [ ] 15.9 Verify all `await store.*` calls in Vue files are wrapped in `try/catch`
- [ ] 15.10 Verify no raw `fetch()` used for mutations — must use `@nextcloud/axios`
