# Proposal: Catalog & Purchase Management — Shillinq — Analytics

## Why

Procurement and purchasing managers working with Shillinq currently lack integrated analytics
to turn raw purchasing data into actionable decisions. Market research across 644 tender mentions
(demand score 1934) shows that service procurement with SOW-based PO creation and milestone
tracking alone is the single most requested capability in the procurement space. Competitors
cover these areas at 1–11% — leaving a high-value, low-competition gap that Shillinq can claim
by surfacing spend, delivery, catalog, KPI, and sustainability analytics natively in Nextcloud.

## What Changes

- **Spend analytics:** real-time expense dashboards, estimated-vs-actual variance tracking,
  and strategic procurement insights derived from existing `SpendTransaction`, `Expense`,
  and `ProcurementCategory` objects in OpenRegister.
- **SOW-based procurement:** associate `StatementOfWork` objects with `PurchaseOrder` records;
  surface milestone progress on the procurement dashboard.
- **Delivery operations:** generate delivery notes (PDF) from `GoodsReceipt`/`ProofOfDelivery`
  objects; track online delivery status and bill-matching against `VendorBill`.
- **Catalog analytics:** catalog-item usage statistics and inventory tracking widgets for
  `CatalogItem` and `InventoryItem` objects; order confirmation tracking per `ProcurementOrder`.
- **Sustainability reporting:** CO2 emission data linked to `ProcurementCategory`; spend
  breakdowns with sustainability labels on the dashboard.
- **KPI & executive dashboards:** configurable KPI scorecards (`SupplierKPI`,
  `SupplierPerformanceScore`), annual procurement statistics, and drill-down dashboards
  built on `CnDashboardPage` + `CnChartWidget`.

## Capabilities

### New Capabilities

- `spend-analytics`: Real-time expense tracking dashboard, estimated-vs-actual spend tracking,
  and strategic analytics widgets consuming `SpendTransaction`, `Expense`, `ExpenseReport`,
  and `ProcurementCategory` objects.
- `sow-procurement`: SOW-based purchase order creation with milestone tracking, linking
  `StatementOfWork` → `PurchaseOrder` → `ContractMilestone`.
- `delivery-operations`: Delivery note PDF generation from `GoodsReceipt`/`ProofOfDelivery`
  objects; online delivery status tracking and bill matching against `VendorBill`.
- `catalog-analytics`: Catalog item usage statistics and inventory tracking widgets for
  `CatalogItem`, `ProcurementCatalog`, and `InventoryItem`; order confirmation tracking.
- `sustainability-analytics`: CO2 emission spend tracking linked to `ProcurementCategory`;
  sustainability labels on spend charts.
- `procurement-kpis`: Configurable KPI scorecards, executive dashboards, annual procurement
  statistics; built on `SupplierKPI`, `SupplierPerformanceScore`, `SupplierPerformanceReport`.

### Modified Capabilities

_(none — all capabilities are net-new analytics on existing OpenRegister entities)_

## Impact

- `src/views/Dashboard.vue`: new `CnDashboardPage` layout with spend, KPI, and catalog widgets
- `src/store/modules/`: stores for `SpendTransaction`, `CatalogItem`, `ProcurementCategory`,
  `StatementOfWork`, `GoodsReceipt`, `InventoryItem`, `SupplierKPI` via `createObjectStore`
- `lib/Controller/AnalyticsController.php`: aggregation endpoints for spend variance,
  CO2 totals, and annual statistics (data not derivable from raw object lists)
- `lib/Service/DeliveryNoteService.php`: PDF generation for delivery notes (docudesk template)
- `lib/Service/SpendAggregationService.php`: estimated-vs-actual spend computation
- `lib/BackgroundJob/AnnualStatisticsJob.php`: scheduled annual procurement statistics rollup
- `appinfo/routes.php`: analytics API routes
- `l10n/nl.js` + `l10n/en.js`: all new translatable strings
- `lib/Settings/shillinq_register.json`: seed data additions (no schema changes —
  all entities already defined in OpenRegister)
