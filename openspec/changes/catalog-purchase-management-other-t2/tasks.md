# Tasks: Catalog & Purchase Management — Shillinq — Other T2

## Deduplication Check (ADR-012)

- [ ] Verify `ObjectService`, `RegisterService`, `SchemaService` do not already expose catalog/requisition/PO CRUD — they provide generic CRUD; custom controllers are needed for business logic endpoints only.
- [ ] Verify `@conduction/nextcloud-vue` components cover list/detail/form UI — confirmed: `CnIndexPage`, `CnDetailPage`, `CnFormDialog`, `CnDashboardPage` are used throughout; no custom UI framework needed.
- [ ] Verify `WorkflowEngineController` can express procurement automation rules — confirm via OpenRegister source before building custom rule engine.
- [ ] Verify no duplicate schema exists for `ProcurementCatalog`, `CatalogItem`, `ProcurementCategory`, `PurchaseRequisition`, `PurchaseOrder`, `BlanketPurchaseOrder`, `ProcurementQuote`, `SavingsOpportunity` in the shillinq register.

---

## Phase 1: Register & Schema Setup

- [ ] **TASK-1** Create `lib/Settings/shillinq_register.json` with OpenAPI 3.0 + x-openregister format containing schemas for: `ProcurementCatalog`, `CatalogItem`, `ProcurementCategory`, `PurchaseRequisition`, `PurchaseOrder`, `PurchaseOrderRevision`, `BlanketPurchaseOrder`, `ProcurementQuote`, `SavingsOpportunity`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-1`
  - All property names use schema.org vocabulary where equivalent exists (ADR-011)
  - All required fields and enum values defined
  - Relations reference register + schema + objectId (no foreign keys)

- [ ] **TASK-2** Create `IRepairStep` class `lib/Migration/RegisterSchemaRepair.php` to register the `shillinq-procurement` register and import schemas on app install/upgrade
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-2`
  - Uses `RegisterService` to create/update register
  - Uses `SchemaService` to import all schemas from `shillinq_register.json`
  - Does NOT modify existing migrations

- [ ] **TASK-3** Add seed data to repair step: 3–5 Dutch-valued objects per schema (per design.md seed section)
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-3`
  - Seed objects use `@self` envelope: `register`, `schema`, `slug`
  - Only inserted if register is empty (idempotent)

---

## Phase 2: Backend — Controllers & Services

### Procurement Catalog

- [ ] **TASK-4** Create `lib/Service/ProcurementCatalogService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-4`
  - `getCatalogItems(string $catalogSlug, array $filters): array` — filtered item list
  - `activateCatalog(string $catalogSlug): array` — transitions status draft→active, validates required fields
  - `syncHostedBuyerCatalog(string $catalogSlug): void` — fetches external catalog for hosted buyer catalogs
  - Uses `ObjectService->findObjects($register, $schema, $params)` (3 positional args)
  - Stateless — no instance state between requests

- [ ] **TASK-5** Create `lib/Controller/ProcurementCatalogController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-5`
  - Thin controller (<10 lines/method): route + validate + delegate to service
  - Register routes in `appinfo/routes.php` (specific routes before wildcard)
  - Auth: all mutation methods check `IGroupManager::isAdmin()` or `AuthorizationService`
  - Error responses: static message strings only, no `$e->getMessage()`

### Purchase Requisition

- [ ] **TASK-6** Create `lib/Service/PurchaseRequisitionService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-6`
  - `submitRequisition(string $id): array` — draft→submitted, validates required fields
  - `approveRequisition(string $id, string $approverId): array` — pending_approval→approved
  - `rejectRequisition(string $id, string $reason): array` — pending_approval→rejected
  - `convertToPurchaseOrder(string $id): array` — creates PurchaseOrder, sets relation, transitions to converted_to_po
  - `checkAutoApproval(array $requisition): bool` — reads `IAppConfig` threshold, evaluates category overrides
  - Uses `WorkflowEngineController` for `ApprovalChain` routing
  - Sends notifications via `NotificationService` on approve, reject, auto-approve

- [ ] **TASK-7** Create `lib/Controller/PurchaseRequisitionController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-7`
  - Endpoints: `POST /api/procurement/requisitions/{id}/submit`, `/approve`, `/reject`, `/convert-to-po`
  - Validate requester identity from `IUserSession` — never trust client-sent user IDs
  - Auth: approve/reject require `AuthorizationService` role check (Budgethouder or higher)

### Purchase Order

- [ ] **TASK-8** Create `lib/Service/PurchaseOrderService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-8`
  - `generateOrderNumber(int $year): string` — returns `PO-{YYYY}-{00001}` sequential number
  - `signOrder(string $id, string $userId): array` — sets signedBy, signedAt, transitions draft→sent, generates PDF
  - `reviseOrder(string $id, array $changes, string $reason): array` — creates PurchaseOrderRevision, updates PO
  - `confirmDelivery(string $id, string $type): array` — transitions to partially_delivered or delivered
  - `checkDossierCompleteness(string $id): array` — returns list of present/missing required documents
  - PDF generation: uses `docudesk` or equivalent PDF service; business-specific template
  - Sends `NotificationService` notification on revision to supplier

- [ ] **TASK-9** Create `lib/Controller/PurchaseOrderController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-9`
  - Endpoints: `POST /api/procurement/purchase-orders/{id}/sign`, `/revise`, `/confirm-delivery`, `GET /{id}/dossier-check`
  - Auth: sign requires signatory role; revision requires Inkoper role

### Blanket Purchase Orders

- [ ] **TASK-10** Create `lib/Service/BlanketPurchaseOrderService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-10`
  - `releaseCallOff(string $blanketOrderId): array` — creates PurchaseOrder call-off, updates consumedValue/Quantity, checks exhaustion
  - `scheduleNextRelease(array $blanketOrder): void` — computes nextReleaseDate from releaseSchedule
  - `checkExpiry(array $blanketOrder): void` — transitions status to expired or exhausted as appropriate
  - Called by `BlanketOrderReleaseJob`

- [ ] **TASK-11** Create `lib/Controller/BlanketPurchaseOrderController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-11`
  - Endpoint: `POST /api/procurement/blanket-orders/{id}/release`

### Procurement Quotes

- [ ] **TASK-12** Create `lib/Service/ProcurementQuoteService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-12`
  - `acceptQuote(string $id): array` — transitions to accepted, creates PurchaseOrder, sets purchaseOrder relation
  - `rejectQuote(string $id, string $reason): array` — transitions to rejected, stores reason in notes
  - `checkExpiryReminders(): void` — called by `QuoteExpiryReminderJob`; queries quotes expiring within reminder window
  - `expireOverdueQuotes(): void` — transitions received/under_review quotes past expiryDate to expired

- [ ] **TASK-13** Create `lib/Controller/ProcurementQuoteController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-13`
  - Endpoints: `POST /api/procurement/quotes/{id}/accept`, `/reject`

### Procurement Analytics

- [ ] **TASK-14** Create `lib/Service/ProcurementAnalyticsService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-14`
  - `getKpis(): array` — returns: openOrdersCount, openRequisitionsCount, totalOrderValueYtd, realisedSavingsYtd
  - `getCategorySpendTrend(int $months): array` — spend by category over last N months
  - `getDemandSuggestions(): array` — identifies categories below historical average (demand-based procurement)
  - Results cached in `IAppConfig` and refreshed by `ProcurementKpiJob`

- [ ] **TASK-15** Create `lib/Controller/ProcurementAnalyticsController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-15`
  - Endpoint: `GET /api/procurement/analytics/kpis`
  - Auth: `#[AuthorizedAdminSetting]` or Financieel medewerker role

### Procurement Automation

- [ ] **TASK-16** Create `lib/Service/ProcurementAutomationService.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-16`
  - `evaluateRules(string $objectType, array $object): array` — evaluates active `PolicyRule` objects against the given procurement object
  - `applyAutoActions(string $objectType, array $object, array $matchedRules): void` — executes matched rule actions (auto-approve, escalate, etc.)
  - Delegates to `WorkflowEngineController` for complex workflow actions
  - Rules retrieved via `ObjectService` from `PolicyRule` schema

### Metrics & Health

- [ ] **TASK-17** Create `lib/Controller/MetricsController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-17`
  - `GET /api/metrics` — Prometheus text format, admin auth
  - Metrics: `shillinq_health_status`, `shillinq_info`, `shillinq_open_orders_total`, `shillinq_open_requisitions_total`, `shillinq_spend_ytd`

- [ ] **TASK-18** Create `lib/Controller/HealthController.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-18`
  - `GET /api/health` — JSON, `#[PublicPage]` + `#[NoCSRFRequired]`
  - Verifies OpenRegister connectivity

---

## Phase 3: Background Jobs

- [ ] **TASK-19** Create `lib/BackgroundJob/QuoteExpiryReminderJob.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-19`
  - Extends `TimedJob`, runs daily
  - Calls `ProcurementQuoteService->checkExpiryReminders()` and `->expireOverdueQuotes()`
  - Registered in `lib/AppInfo/Application.php`

- [ ] **TASK-20** Create `lib/BackgroundJob/BlanketOrderReleaseJob.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-20`
  - Extends `TimedJob`, runs daily
  - Calls `BlanketPurchaseOrderService->releaseCallOff()` for all active blanket orders where `nextReleaseDate` ≤ today
  - Updates `nextReleaseDate` and consumption totals

- [ ] **TASK-21** Create `lib/BackgroundJob/ProcurementKpiJob.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-21`
  - Extends `TimedJob`, runs weekly
  - Calls `ProcurementAnalyticsService->getKpis()` and caches results in `IAppConfig`

---

## Phase 4: Frontend — Stores

- [ ] **TASK-22** Register entity stores in `src/store/store.js`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-22`
  - Register with `createObjectStore` + plugins (files, auditTrails, relations):
    - `procurement-catalog` → `ProcurementCatalog`
    - `catalog-item` → `CatalogItem`
    - `procurement-category` → `ProcurementCategory`
    - `purchase-requisition` → `PurchaseRequisition`
    - `purchase-order` → `PurchaseOrder`
    - `purchase-order-revision` → `PurchaseOrderRevision`
    - `blanket-purchase-order` → `BlanketPurchaseOrder`
    - `procurement-quote` → `ProcurementQuote`
    - `savings-opportunity` → `SavingsOpportunity`
  - All slugs kebab-case; each registered exactly once (not in both OBJECT_TYPES and ENTITY_STORES)
  - `initializeStores()` fetches settings first, then registers each type

---

## Phase 5: Frontend — Pages & Components

- [ ] **TASK-23** Create `src/views/ProcurementDashboard.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-23`
  - Uses `CnDashboardPage` with `useDashboardView`
  - Four `CnStatsBlock` KPI cards: openstaande bestellingen, openstaande aanvragen, bestelwaarde YTD, besparingen YTD
  - One `CnChartWidget` (bar): order volume per month
  - Fetches KPIs from `GET /api/procurement/analytics/kpis`
  - `Promise.all` for parallel data fetching
  - SPDX header: `<!-- SPDX-License-Identifier: EUPL-1.2 -->`

- [ ] **TASK-24** Add navigation items to `src/components/MainMenu.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-24`
  - Items: Dashboard, Catalogi, Aanvragen, Inkooporders, Raambestellingen, Offertes, Besparingen
  - Uses `NcAppNavigationItem` with `to` prop per route
  - All labels via `t(appName, 'label')` — no hardcoded Dutch strings

- [ ] **TASK-25** Add routes to `src/router/index.js`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-25`
  - Flat named routes (no nesting): all routes listed in design.md route table
  - History mode, base `generateUrl('shillinq', '')`
  - Catch-all `*` redirects to `/procurement`
  - Route params via arrow function props

- [ ] **TASK-26** Create catalog index page `src/views/ProcurementCatalogIndex.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-26`
  - Uses `CnIndexPage` with `useListView('procurement-catalog', { sidebarState, objectStore })`
  - `CnFacetSidebar` filters: status, isHostedBuyer, validUntil range
  - Row click → `$router.push({ name: 'ProcurementCatalogDetail', params: { id } })`
  - Add button → `/procurement/catalogs/new`

- [ ] **TASK-27** Create catalog detail page `src/views/ProcurementCatalogDetail.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-27`
  - Uses `CnDetailPage` with `useDetailView`
  - `CnDetailCard` sections: Catalogusgegevens, Artikelen (table of CatalogItem), Prijsregels
  - Edit button opens `CnFormDialog`; Delete opens `CnDeleteDialog`
  - `CnObjectSidebar` with Files, Audit tabs

- [ ] **TASK-28** Create catalog item index page `src/views/CatalogItemIndex.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-28`
  - `CnIndexPage` + `CnFacetSidebar` (category, catalog, cpvCode, price range, isActive)
  - "Toevoegen aan mand" row action → creates draft PurchaseRequisition with item pre-filled
  - Full-text search via `IndexService`

- [ ] **TASK-29** Create purchase requisition index and detail pages
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-29`
  - `src/views/PurchaseRequisitionIndex.vue`: `CnIndexPage`, `CnFacetSidebar` (status, category, requester)
  - `src/views/PurchaseRequisitionDetail.vue`:
    - `CnTimelineStages` component showing requisition lifecycle stages
    - Action buttons: Indienen, Goedkeuren, Afwijzen, Omzetten naar PO (shown per status + role)
    - `CnDetailCard` for line items, attachments, approval status
    - `CnObjectSidebar` with Files tab (supporting documents)

- [ ] **TASK-30** Create purchase order index and detail pages
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-30`
  - `src/views/PurchaseOrderIndex.vue`: `CnIndexPage`, `CnFacetSidebar` (status, supplier, date range)
  - `src/views/PurchaseOrderDetail.vue`:
    - Action buttons: Ondertekenen, Revisie aanmaken, Ontvangst bevestigen (per status)
    - `CnDetailCard` sections: Bestelgegevens, Regelitems, Revisies, Dossiercompleteness check
    - Dossier completeness indicator shows present/missing required documents
    - `CnObjectSidebar` with Files, Audit tabs

- [ ] **TASK-31** Create blanket purchase order pages
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-31`
  - `src/views/BlanketPurchaseOrderIndex.vue`: `CnIndexPage` with status, consumedValue/maxValue progress
  - `src/views/BlanketPurchaseOrderDetail.vue`:
    - Consumed vs. max value/quantity progress bars (`CnProgressBar`)
    - Call-off orders table in `CnDetailCard`
    - "Afroep aanmaken" button → `POST /api/procurement/blanket-orders/{id}/release`

- [ ] **TASK-32** Create procurement quote index and detail pages
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-32`
  - `src/views/ProcurementQuoteIndex.vue`: `CnIndexPage` with expiry date column, `CnStatusBadge` per status
  - Expired and expiring-soon quotes highlighted visually (status badge color, not sole indicator)
  - `src/views/ProcurementQuoteDetail.vue`:
    - Action buttons: Accepteren, Afwijzen (per status)
    - Expiry countdown indicator
    - Accept → `POST /api/procurement/quotes/{id}/accept` → redirects to new PO

- [ ] **TASK-33** Create savings opportunity index page `src/views/SavingsOpportunityIndex.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-33`
  - `CnIndexPage` + `CnFacetSidebar` (status, savingsType, category)
  - Realised vs. target savings visible in list view
  - `CnStatsBlock` above list showing total target and total realised YTD

- [ ] **TASK-34** Create procurement settings page `src/views/ProcurementSettings.vue`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-34`
  - `CnVersionInfoCard` FIRST (mandatory per ADR-004)
  - `CnRegisterMapping` for shillinq-procurement register
  - `CnSettingsSection`: Drempelwaarden (autoApprovalThreshold per category)
  - `CnSettingsSection`: Inkoopautomatisering (procurement automation rules)
  - `CnSettingsSection`: Inkoopteam (team and role assignments)
  - Load settings from `GET /api/settings`; save via `POST /api/settings`
  - Re-import button calls `POST /api/settings/load`

---

## Phase 6: i18n

- [ ] **TASK-35** Add Dutch translations `l10n/nl.json`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-35`
  - All user-visible strings from new Vue components
  - Keys in English; values in Dutch
  - Example entries: `"New catalog": "Nieuwe catalogus"`, `"Submit requisition": "Aanvraag indienen"`, `"Sign order": "Ondertekenen"`, `"Accept quote": "Offerte accepteren"`

- [ ] **TASK-36** Add English translations `l10n/en.json`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-36`
  - English source strings mirroring nl.json keys

---

## Phase 7: Tests

- [ ] **TASK-37** PHPUnit tests `tests/Unit/Service/ProcurementCatalogServiceTest.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-37`
  - ≥ 3 test methods: `testActivateCatalogTransitionsStatus`, `testGetCatalogItemsAppliesFilters`, `testSyncHostedBuyerCatalogFallsBackOnApiFailure`

- [ ] **TASK-38** PHPUnit tests `tests/Unit/Service/PurchaseRequisitionServiceTest.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-38`
  - ≥ 3 test methods: `testAutoApprovalBelowThreshold`, `testRequisitionAboveThresholdRoutesToApprovalChain`, `testConvertToPurchaseOrderCreatesPoAndSetsRelation`

- [ ] **TASK-39** PHPUnit tests `tests/Unit/Service/PurchaseOrderServiceTest.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-39`
  - ≥ 3 test methods: `testSignOrderSetsSignedByAndTransitionsToDraft`, `testReviseOrderCreatesRevisionRecord`, `testCheckDossierCompletenessReturnsMissingDocuments`

- [ ] **TASK-40** PHPUnit tests `tests/Unit/Service/ProcurementQuoteServiceTest.php`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-40`
  - ≥ 3 test methods: `testAcceptQuoteCreatesPurchaseOrder`, `testExpiredQuoteTransitionsStatus`, `testReminderJobSendsNotificationInWindow`

- [ ] **TASK-41** Newman/Postman integration test collection `tests/integration/procurement_api.json`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-41`
  - Covers: create catalog, create catalog item, create requisition, submit, approve, convert-to-po, sign PO, create blanket order, release call-off, create quote, accept quote

- [ ] **TASK-42** Browser tests (Playwright) `tests/browser/procurement.spec.js`
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-42`
  - GIVEN/WHEN/THEN scenarios from specs.md:
    - REQ-REQ-004: auto-approval below threshold
    - REQ-PO-005: sign PO and verify signature
    - REQ-QTE-001: quote expiry reminder
    - REQ-ANA-001: savings opportunity realised
  - All tests must pass in `composer check:strict`

---

## Phase 8: Documentation

- [ ] **TASK-43** Create `docs/procurement-catalog.md` with screenshots of catalog browser, item detail, and add-to-basket flow
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-43`

- [ ] **TASK-44** Create `docs/purchase-requisition.md` with screenshots of requisition form, approval timeline, and auto-approval flow
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-44`

- [ ] **TASK-45** Create `docs/purchase-orders.md` with screenshots of PO creation, signing, revision, and dossier completeness
  - `@spec` tag: `openspec/changes/catalog-purchase-management-other-t2/tasks.md#TASK-45`

---

## Pre-commit Checklist (must pass before every commit)

- [ ] SPDX headers present on ALL new PHP, Vue, and JS files
- [ ] All `ObjectService` calls use 3 positional args: `($register, $schema, $idOrParams)`
- [ ] No `$e->getMessage()` in any `JSONResponse` — static error strings only
- [ ] Every POST/PUT/DELETE controller method has a backend auth check
- [ ] Each entity type registered exactly once in `store.js`, kebab-case slug
- [ ] `npm run lint` passes (catches missing package.json entries)
- [ ] All user-visible strings use `t(appName, 'key')` — no hardcoded Dutch/English
- [ ] Every `await objectStore.action()` is wrapped in `try/catch`
- [ ] No raw `fetch()` for mutations — `@nextcloud/axios` only
- [ ] No `from '@nextcloud/vue'` imports — use `@conduction/nextcloud-vue`
- [ ] Every `<NcFoo>` / `<CnFoo>` in templates is imported AND listed in `components: {}`
- [ ] All entity type strings are kebab-case across store, search, routes, and views
- [ ] All `t()` keys are English; Dutch translations go in `l10n/nl.json`
- [ ] Every named route in the router has a matching navigation item or component reference
- [ ] All tasks marked `[x]` are fully implemented — no stubs or empty method bodies
