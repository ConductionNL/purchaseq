---
kind: code
depends_on: []
---

# Proposal: catalog-purchase-management-other-t3

## Why

Shillinq lacks guided catalog browsing, structured purchase requisition workflows, and a configurable discount engine — three capabilities present in nearly every competing procurement solution. Procurement staff at SMBs and municipalities must coordinate catalog orders manually outside the system, submit requisitions by email, and apply discounts by hand on each invoice. This creates maverick spending, approval bottlenecks, and pricing inconsistencies. Demand analysis of 25 features across competitors confirms the gap: guided buying (demand 4), purchase community integrations (demand 3), and a discount/promotion engine (demand 2) are the top unmet needs. This change closes these gaps by adding catalog-based ordering, a requisition approval lifecycle, a rule-based discount engine, 2-way/3-way PO matching, and quotation revision management to Shillinq.

## What Changes

- Guided catalog browsing UI with enriched `CatalogItem` attributes, UNSPSC classification codes, and pricing display
- Punch-out catalog integration (cXML/OCI) for external supplier catalogs (`Catalogusbeheer`)
- Purchase requisition lifecycle: create with customizable intake forms → submit → approve/reject via `ApprovalChain`
- Discount and promotion engine using `PricingRule` — coupon codes, per-item discounts, volume discounts, dynamic negotiated pricing, with configurable first-match or all-match execution modes and rule priority ordering
- 2-way and 3-way `PurchaseOrder` matching against `GoodsReceipt` and `Invoice` with configurable tolerance rules
- Quotation management: create professional `Quote`, track status, create revisions with `previousVersion` link
- Cost center assignment on catalog orders and purchase requisitions
- Sale order line sequence management and sale stock sourcing rules
- `PurchaseOrder` status tracking dashboard

## Capabilities

### New Capabilities

- `catalog-browsing`: Guided buying experience — browse `ProcurementCatalog` and `CatalogItem` objects with enriched attributes, UNSPSC codes, pricing display, faceted search, and basket flow
- `catalog-punchout`: Punch-out catalog integration (cXML/OCI) — generate PunchOutSetupRequest to external supplier URL, receive PunchOutOrderMessage, convert to local `CatalogItem` basket
- `purchase-requisition`: Full `PurchaseRequisition` lifecycle — customizable intake forms per `ProcurementCategory`, submit, route via `ApprovalChain`, approve or reject with `ApprovalTask`
- `discount-engine`: Rule-based pricing engine on `PricingRule` — coupon codes, per-item and percentage discounts, volume thresholds, `first-match` and `all-match` execution modes, priority ordering
- `po-matching`: 2-way and 3-way `PurchaseOrder` matching against `GoodsReceipt` and `Invoice` with configurable variance tolerance per `Administration`
- `quotation-management`: `Quote` lifecycle — create, revise (with `previousVersion` relation and `revisionNumber`), track status (concept → verzonden → herzien → geaccepteerd/afgewezen/verlopen), generate professional PDF
- `cost-center-assignment`: Assign `CostCenter` to catalog orders and `PurchaseRequisition` objects
- `sale-order-sequencing`: Manage line item `sequence` integer on `PurchaseOrder` for display ordering
- `sale-stock-sourcing`: Stock sourcing rule lookup — associate products with preferred supplier/warehouse combinations

### Modified Capabilities

- `purchase-order-view`: Extend existing PO detail view with matching result display, status timeline (`CnTimelineStages`), and stock sourcing indicator

## Impact

- Apps affected: Shillinq
- Breaking changes: none
- Migration needed: no — all entities are pre-existing in the OpenRegister schema; this change adds custom services and UI views on top of existing data model
