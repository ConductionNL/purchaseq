---
kind: code
depends_on: []
---

# Catalog & Purchase Management — Shillinq

## Why

Procurement teams using Shillinq on Nextcloud lack a unified catalog and purchase management
system. Staff navigate external tools for product catalog browsing, purchase requisition
creation, PO management, approval routing, three-way matching, and spend analysis — creating
data silos, audit gaps, and inefficient workflows. Procurement management is the highest-demand
feature cluster in this spec (demand score 4,150; 1,356 tender mentions). Catalog management
(1,715), requisition management (1,713), and PO management with approval workflows (1,679)
round out the top tier. Dutch public-sector tenders repeatedly cite source-to-pay completeness
and EU procurement threshold compliance as hard selection criteria.

## What Changes

- **Catalog management**: Index, search, and detail views for `ProcurementCatalog` and
  `CatalogItem` entities; bulk import via `CnMassImportDialog`; user-specific product views
  via `CnFacetSidebar`; composite item (kit/bundle) support linking parent `CatalogItem`
  to component items.
- **Purchase requisition**: Multi-line `PurchaseRequisition` creation with custom fields and
  `CostAllocation` per line; automatic routing through `ApprovalChain` / `ApprovalRoute`
  based on category, amount, and entity; legal deadline tracking on the requisition object.
- **Purchase order management**: `PurchaseOrder` creation from approved `PurchaseRequisition`
  records; three-way matching workflow linking `PurchaseOrder` → `GoodsReceipt` →
  `VendorBill`; blanket PO (`BlanketPurchaseOrder`) management with `CallOffOrder` release
  scheduling and consumption monitoring; free-text POs for non-catalog items with
  category-based approval routing; multi-location delivery with `Location` references.
- **Pricing and discount management**: `PricingRule` and `RateCard` management for catalog
  items; volume, category, and supplier-based discount structures; centralized rate
  enforcement; coupon and promotion code support for subscriptions; EU procurement threshold
  reference table for procedure type selection.
- **Spend analysis dashboard**: Category-level spend visibility using `SpendTransaction` and
  `SpendCategory` records aggregated in `CnDashboardPage` `CnChartWidget` widgets; cross-P2P
  cycle analysis linking sourcing decisions to downstream procurement outcomes.
- **Seed data**: 3–5 realistic Dutch seed objects per entity via `lib/Settings/shillinq_register.json`.

## Capabilities

### New Capabilities

- `catalog-management`: Browse and manage internal procurement catalogs and catalog items with
  negotiated pricing, bulk import/export, user-specific views, and composite item (kit/bundle)
  management.
- `purchase-requisition`: Create multi-line purchase requisitions with custom fields, cost
  allocation, legal deadlines, and multi-step approval routing through configurable approval
  chains.
- `purchase-order-management`: Create purchase orders from approved requisitions or free-text;
  three-way matching against goods receipts and vendor bills; blanket PO management with
  call-off releases and consumption monitoring; multi-location delivery; receiving logs;
  PO templates for recurring procurement.
- `pricing-and-discounts`: Manage pricing rules, rate cards, volume/category-based discounts,
  and EU procurement threshold reference data for procedure type compliance.
- `spend-analysis`: Category-level spend visibility dashboard with KPI widgets, trend charts,
  and cross-P2P-cycle analytics.

## Impact

- `src/views/ProcurementCatalogs.vue`: Index page for procurement catalogs using `CnIndexPage`
- `src/views/CatalogItemsIndex.vue`: Index page for catalog items with `CnFacetSidebar`
- `src/views/CatalogItemDetail.vue`: Detail page for a single catalog item
- `src/views/PurchaseRequisitionsIndex.vue`: Index page with approval status badges
- `src/views/PurchaseRequisitionDetail.vue`: Detail page with multi-line item table and
  approval timeline (`CnTimelineStages`)
- `src/views/PurchaseOrdersIndex.vue`: Index page with three-way match status column
- `src/views/PurchaseOrderDetail.vue`: Detail page with linked `GoodsReceipt` and `VendorBill`
- `src/views/BlanketPurchaseOrdersIndex.vue`: Index page with consumption progress bar
- `src/views/SpendAnalysisDashboard.vue`: Dashboard with `CnChartWidget` + `CnStatsBlock`
- `src/router/index.js`: New named routes for all catalog and procurement pages
- `src/store/modules/`: Object stores for each entity via `createObjectStore`
- `lib/Controller/CatalogController.php`: Pricing lookup, bulk validation, composite item resolution
- `lib/Controller/ProcurementController.php`: Requisition-to-PO workflow, three-way match trigger
- `lib/Service/ThreeWayMatchService.php`: PO ↔ GoodsReceipt ↔ VendorBill matching logic
- `lib/Service/ApprovalRoutingService.php`: Requisition routing via `ApprovalChain` / `ApprovalRoute`
- `lib/Service/ProcurementThresholdService.php`: EU procurement threshold lookup and procedure type validation
- `lib/Settings/shillinq_register.json`: Seed data for catalog and procurement entities
- `appinfo/routes.php`: New API routes under `/api/catalog-items`, `/api/procurement-catalogs`,
  `/api/purchase-requisitions`, `/api/purchase-orders`, `/api/blanket-purchase-orders`
- `tests/Unit/Service/ThreeWayMatchServiceTest.php`: Unit tests for matching service
- `tests/Unit/Service/ApprovalRoutingServiceTest.php`: Unit tests for approval routing service
- `tests/Unit/Service/ProcurementThresholdServiceTest.php`: Unit tests for threshold validation
- `tests/Unit/Controller/CatalogControllerTest.php`: Unit tests for catalog controller
- `tests/Unit/Controller/ProcurementControllerTest.php`: Unit tests for procurement controller
