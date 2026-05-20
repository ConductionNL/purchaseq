# Tasks: Catalog & Purchase Management — Analytics

**Change:** catalog-purchase-management-analytics
**Artifact sequence:** proposal → design → specs → tasks ✓

---

## Deduplication Check (ADR-012)

- [x] `ObjectService`, `RegisterService`, `SchemaService` — no overlap: all analytics
      aggregation logic is domain-specific to Shillinq procurement data.
- [x] `@conduction/nextcloud-vue` — `CnDashboardPage`, `CnChartWidget`, `CnStatsBlock`,
      `CnKpiGrid`, `CnTableWidget`, `CnTimelineStages` used as-is; no rebuild.
- [x] `ExportService`/`CnMassExportDialog` — used for spend and catalog exports; no custom
      export controller built.
- [x] `AuditTrailService` — automatic via OpenRegister; no custom audit logging.
- [x] `NotificationService` — used for low-stock and bill-mismatch alerts; no custom system.
- [x] No shared spec overlap found with other Shillinq changes.

---

## 1. Backend — Analytics Controller & Aggregation Service

- [ ] 1.1 Create `lib/Controller/AnalyticsController.php` with `@spec` tags linking to
      `tasks.md#task-1`. Implement four thin endpoints (< 10 lines each):
      - `GET /api/analytics/spend-variance` → delegates to `SpendAggregationService`
      - `GET /api/analytics/co2` → delegates to `SpendAggregationService`
      - `GET /api/analytics/catalog-usage` → reads `CatalogItem` + `ProcurementOrder` via ObjectService
      - `GET /api/analytics/annual-stats` → reads latest annual `Report` object
- [ ] 1.2 Register routes in `appinfo/routes.php` (specific before wildcard `{slug}`):
      `analytics-spend-variance`, `analytics-co2`, `analytics-catalog-usage`, `analytics-annual-stats`
- [ ] 1.3 Create `lib/Service/SpendAggregationService.php`:
      - `getSpendVariance(string $from, string $to): array` — groups `SpendTransaction` by
        `ProcurementCategory`, joins `BudgetAllocation` for estimated amounts, returns
        `[{ category, estimated, actual, variance, variancePct }]`
      - `getCo2Totals(string $from, string $to): array` — multiplies actual spend by
        `co2EmissionFactor` per category, skips null/zero factors, returns
        `[{ category, spend, co2Kg, co2Factor }]`
      - All methods: stateless, constructor-injected `ObjectService`, `@spec` tags
- [ ] 1.4 Write PHPUnit tests in `tests/Unit/Service/SpendAggregationServiceTest.php`:
      - `testGetSpendVarianceReturnsStructuredArray()`
      - `testCo2SkipsCategoriesWithoutFactor()`
      - `testGetSpendVarianceWithDateRangeFilter()`

---

## 2. Backend — Delivery Note Service

- [ ] 2.1 Create `lib/Service/DeliveryNoteService.php` with `@spec` tags (REQ-CAT-004):
      - `generateDeliveryNote(string $goodsReceiptId): void`
      - Resolves `GoodsReceipt` → linked `PurchaseOrder`, `Supplier`, `ProofOfDelivery`
        via `ObjectService` relations
      - Renders docudesk Twig template `templates/pdf/delivery_note.html.twig`
      - Attaches generated PDF to `GoodsReceipt` via `FileService`
      - Throws `\RuntimeException` (not exposes stack trace) if required relations are missing
- [ ] 2.2 Create `templates/pdf/delivery_note.html.twig` with fields:
      pakbon-nummer, datum, leverancier, inkooporderreferentie, ontvangen goederen, handtekening
- [ ] 2.3 Add "Pakbon genereren" action to `AnalyticsController` or a dedicated
      `DeliveryNoteController`: `POST /api/delivery-notes/{goodsReceiptId}/generate`
- [ ] 2.4 Write PHPUnit tests in `tests/Unit/Service/DeliveryNoteServiceTest.php`:
      - `testGenerateDeliveryNoteAttachesToGoodsReceipt()`
      - `testGenerateThrowsWhenSupplierMissing()`

---

## 3. Backend — Annual Statistics Background Job

- [ ] 3.1 Create `lib/BackgroundJob/AnnualStatisticsJob.php` extending `QueuedJob` with `@spec`
      tags (REQ-CAT-006):
      - `run(array $argument): void` — aggregates `SpendTransaction`, `PurchaseOrder`,
        `GoodsReceipt` for prior fiscal year via `ObjectService.findAll()` with year filter
      - Saves a `Report` object to OpenRegister with totals and category breakdown
      - Admin-only manual trigger via `POST /api/analytics/annual-stats/trigger` (checks
        `IGroupManager::isAdmin()` server-side — ADR-005)
- [ ] 3.2 Register job in `lib/AppInfo/Application.php` via `IJobList::add()`
- [ ] 3.3 Write PHPUnit test in `tests/Unit/BackgroundJob/AnnualStatisticsJobTest.php`:
      - `testRunCreatesReportObject()`
      - `testAdminTriggerEndpointRequiresAdminRole()`

---

## 4. Backend — Bill Matching & Maverick Spend

- [ ] 4.1 Create `lib/Service/BillMatchingService.php` with `@spec` tags (REQ-CAT-005):
      - `matchBillToPurchaseOrder(string $billId, string $poId): void`
      - Creates OpenRegister relation from `VendorBill` → `PurchaseOrder`
      - Updates `PurchaseOrder.status` to "Factuur ontvangen"
      - If amount difference > 1%: creates `MaverickSpendAlert` object and notifies
        procurement officer via `NotificationService`
- [ ] 4.2 Add thin controller method in a `BillController` or `AnalyticsController`:
      `POST /api/vendor-bills/{billId}/match-po`
- [ ] 4.3 Write PHPUnit test in `tests/Unit/Service/BillMatchingServiceTest.php`:
      - `testMatchCreatesMaverickAlertOnAmountMismatch()`
      - `testMatchUpdatesPoStatus()`

---

## 5. Frontend — Pinia Stores

- [ ] 5.1 Create `src/store/modules/spendTransactions.js` using `createObjectStore('SpendTransaction')`
      with plugins: `auditTrails`, `files`, `relations`, `search`
- [ ] 5.2 Create `src/store/modules/catalogItems.js` using `createObjectStore('CatalogItem')`
- [ ] 5.3 Create `src/store/modules/procurementCategories.js` using `createObjectStore('ProcurementCategory')`
- [ ] 5.4 Create `src/store/modules/statementOfWorks.js` using `createObjectStore('StatementOfWork')`
      with `relations` plugin (for ContractMilestone relation)
- [ ] 5.5 Create `src/store/modules/supplierKpis.js` using `createObjectStore('SupplierKPI')`
- [ ] 5.6 Create `src/store/modules/inventoryItems.js` using `createObjectStore('InventoryItem')`
- [ ] 5.7 Register all stores in `src/store/store.js` → `initializeStores()` via
      `objectStore.registerObjectType(name, schemaSlug, registerSlug)` for each

---

## 6. Frontend — Dashboard & Widgets

- [ ] 6.1 Create `src/views/Dashboard.vue` using `CnDashboardPage` (GridStack drag-drop).
      Include 6 default widget slots: SpendVariance, Sustainability, CatalogUsage, KpiScorecard,
      MilestoneProgress, AnnualStats. Save layout to `UserPreference` on change.
- [ ] 6.2 Create `src/components/widgets/SpendVarianceWidget.vue`:
      - Fetches `GET /api/analytics/spend-variance` on mount
      - Renders bar chart via `CnChartWidget` (type: bar, categories on X, estimated+actual on Y)
      - Highlights over-budget categories using NL Design token (not hardcoded hex, ADR-010)
      - Shows tooltip with estimated, actual, variance, variancePct on hover
- [ ] 6.3 Create `src/components/widgets/SustainabilityWidget.vue`:
      - Fetches `GET /api/analytics/co2` on mount
      - Renders donut chart via `CnChartWidget` (type: donut, CO2 kg per category)
      - Shows total CO2 as `CnStatsBlock` above the chart
      - Footnote showing count of categories excluded (no CO2 factor)
- [ ] 6.4 Create `src/components/widgets/CatalogUsageWidget.vue`:
      - Fetches `GET /api/analytics/catalog-usage` on mount
      - Renders top-20 items via `CnTableWidget` (sortable columns)
      - Row click navigates to `CatalogItem` detail page
- [ ] 6.5 Create `src/components/widgets/KpiScorecardWidget.vue`:
      - Reads `SupplierKPI` objects via `supplierKpis` store
      - Renders scorecards via `CnKpiGrid`
      - Below-target KPIs use NL Design token for warning state
- [ ] 6.6 Create `src/components/widgets/MilestoneProgressWidget.vue`:
      - Reads `StatementOfWork` objects with related `ContractMilestone` via `statementOfWorks` store
      - Renders milestones via `CnTimelineStages`
      - Overdue milestones (due < today, status ≠ Afgerond) highlighted
- [ ] 6.7 Create `src/components/widgets/AnnualStatsWidget.vue`:
      - Fetches `GET /api/analytics/annual-stats` on mount
      - Renders 4 `CnStatsBlock` cards: Totale inkoop, Aantal PO's, Actieve leveranciers,
        Grootste categorie
      - "Volledig rapport bekijken" link navigates to `Report` detail page

---

## 7. Frontend — Analytics Views

- [ ] 7.1 Create `src/views/SpendAnalytics.vue` using `CnIndexPage` + `useListView`:
      - Entity: `SpendTransaction`
      - Columns from `columnsFromSchema()`; add date filter via `CnFilterBar`
      - Export button triggers `CnMassExportDialog`
- [ ] 7.2 Create `src/views/CatalogAnalytics.vue` using `CnIndexPage` + `useListView`:
      - Entity: `CatalogItem`
      - Usage stats tab (catalog-usage data) + Inventory tab (`InventoryItem` objects)
      - Row click → `CatalogItem` detail with order timeline chart
- [ ] 7.3 Create `src/views/ProcurementKpi.vue` using `CnDetailPage`:
      - Renders `SupplierPerformanceReport` detail with `CnDetailCard` sections
      - Includes benchmarking bar chart (`CnChartWidget`) and trend line chart

---

## 8. Frontend — Router & Navigation

- [ ] 8.1 Add routes to `src/router/index.js`:
      - `/` → `Dashboard`
      - `/spend-analytics` → `SpendAnalytics`
      - `/catalog-analytics` → `CatalogAnalytics`
      - `/procurement-kpis/:id` → `ProcurementKpi` (props via arrow function)
      - `/settings` → `Settings`
      - `*` → redirect to `/`
- [ ] 8.2 Add navigation items to `src/components/MainMenu.vue` using `NcAppNavigationItem`:
      - Dashboard (mdi-view-dashboard)
      - Uitgavenanalyse (mdi-chart-bar)
      - Catalogusanalyse (mdi-package-variant)
      - Inkoop-KPI's (mdi-gauge)

---

## 9. Seed Data & Register Configuration

- [ ] 9.1 Add seed data to `lib/Settings/shillinq_register.json` (or equivalent register
      template) for `CatalogItem` (5 objects), `ProcurementCategory` (4 objects),
      `SpendTransaction` (5 objects), `StatementOfWork` (3 objects) — Dutch values as
      specified in `design.md`. All objects use `@self` envelope (ADR-001).
- [ ] 9.2 Verify all seed objects are importable via `POST /api/settings/load` without errors.

---

## 10. Translations (ADR-007)

- [ ] 10.1 Add Dutch translations to `l10n/nl.js` for all new user-visible strings:
      "Uitgavenanalyse", "Catalogusanalyse", "Inkoop-KPI's", "Pakbon genereren",
      "Voorraadwaarschuwing", "Koppelen aan PO", "Jaarstatistieken herberekenen", etc.
- [ ] 10.2 Add English translations to `l10n/en.js` as primary language counterparts.
- [ ] 10.3 Verify no hardcoded user-visible strings remain in Vue components
      (all must use `t(appName, 'key')`).

---

## 11. Metrics & Health (ADR-006)

- [ ] 11.1 Extend `GET /api/metrics` to include:
      - `shillinq_spend_transactions_total` — count of `SpendTransaction` objects
      - `shillinq_purchase_orders_total` — count of `PurchaseOrder` objects
      - `shillinq_annual_report_last_run_timestamp` — epoch of last `AnnualStatisticsJob` run
- [ ] 11.2 Verify `GET /api/health` still returns `{ status: "ok" }` with OpenRegister
      connectivity check.

---

## 12. Tests & Documentation (ADR-008, ADR-009)

- [ ] 12.1 Write Newman/Postman collection in `tests/integration/analytics.json` covering:
      `GET /api/analytics/spend-variance`, `GET /api/analytics/co2`,
      `GET /api/analytics/catalog-usage`, `GET /api/analytics/annual-stats`
- [ ] 12.2 Write Playwright browser tests for GIVEN/WHEN/THEN scenarios:
      - REQ-SPA-001 Scenario 1.1 (dashboard loads spend summary)
      - REQ-DSH-001 Scenario 1.3 (drill-down to category)
      - REQ-CAT-004 Scenario 4.1 (delivery note PDF generated)
- [ ] 12.3 Add feature documentation in `docs/features/analytics.md` with screenshots
      of: spend variance chart, sustainability donut, KPI scorecard, annual stats widget.
- [ ] 12.4 Run `composer check:strict` — all tests must pass before marking tasks complete.
