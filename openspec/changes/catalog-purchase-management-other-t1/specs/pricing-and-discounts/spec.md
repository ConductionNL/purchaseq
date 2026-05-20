# Spec: Pricing and Discounts

Entities: `PricingRule`, `RateCard`, `CatalogItem`, `SpendTransaction`, `SpendCategory`,
`ProcurementCategory`

---

## REQ-PRC-001: Product Pricing Catalog Management

Catalog administrators can define and manage pricing rules per catalog item, category,
supplier, or volume bracket. Pricing rules are applied at requisition and PO creation time.

#### Scenario 1: Create a percentage-based volume discount rule

- **GIVEN** a catalog administrator opens the pricing rules index and clicks "Toevoegen"
- **WHEN** they fill in: name, `discountType = percentage`, `discountValue = 5.0`,
  `minimumOrderAmount = 10000`, and `category = ict-apparatuur`
- **THEN** a `PricingRule` is saved and linked to the specified `ProcurementCategory`
- **AND** it appears in the pricing rules index with the category badge and effective date

#### Scenario 2: Create a fixed-price rule for a supplier framework agreement

- **GIVEN** a catalog administrator creates a `PricingRule` with `discountType = fixed_price`,
  `fixedUnitPrice = 38.50`, and `supplier = supplier-iss-nederland`
- **WHEN** a user adds a `CatalogItem` from that supplier to a requisition
- **THEN** `CatalogController.resolvePrice()` detects the applicable `PricingRule`
  and uses the fixed unit price instead of the catalog item's default `unitPrice`
- **AND** the applied rule name is shown next to the price on the requisition line item

#### Scenario 3: Pricing rule precedence — most specific wins

- **GIVEN** a `CatalogItem` has an applicable `PricingRule` at both category level
  and supplier level
- **WHEN** `CatalogController.resolvePrice()` is called for that item
- **THEN** the supplier-level rule takes precedence over the category-level rule
  (supplier > category > global)
- **AND** the applied rule is recorded in the line item for audit purposes

#### Scenario 4: View all pricing rules for a catalog item

- **GIVEN** a user opens a `CatalogItem` detail page
- **WHEN** the page renders
- **THEN** a "Prijsregels" section lists all `PricingRule` objects that could apply to
  this item (matched by category, supplier, or global)
- **AND** the currently active rule is highlighted with the effective price shown

---

## REQ-PRC-002: Centralized Pricing Terms Management with Rate Enforcement

`RateCard` objects define contracted rates for services or products negotiated with a
supplier. Rates are enforced when a procurement officer creates a requisition or PO
line referencing a supplier with an active `RateCard`.

#### Scenario 1: Create a rate card for a service supplier

- **GIVEN** a catalog administrator creates a `RateCard` with: supplier, effective date,
  and one or more rate lines (service type, unit, rate amount, currency)
- **WHEN** the rate card is saved
- **THEN** it is linked to the `Supplier` object and appears on the supplier's detail page
- **AND** `CatalogController` uses this rate card when resolving prices for line items
  from that supplier in the applicable time period

#### Scenario 2: Rate enforcement on requisition line item

- **GIVEN** a `RateCard` is active for a supplier in the current period
- **WHEN** a user sets a unit price on a requisition line for that supplier that deviates
  from the contracted rate by more than 5%
- **THEN** `CatalogController.resolvePrice()` returns a validation warning
- **AND** the UI shows a warning: "Prijs wijkt af van raamovereenkomst (gecontracteerd: €X)"
- **AND** the user can override with a reason, which is stored in the audit trail

#### Scenario 3: Export rate card as PDF

- **GIVEN** a user opens a `RateCard` detail page
- **WHEN** they click "Exporteren als PDF"
- **THEN** a PDF is generated containing all rate lines, the supplier name, and validity period
- **AND** the PDF is stored via `FileService` attached to the `RateCard` object

---

## REQ-PRC-003: Discount Management — Coupons and Promotion Codes for Subscriptions

For subscription-type catalog items (software, SaaS, recurring services), promotional
codes and coupons can be applied to reduce the invoiced price.

#### Scenario 1: Apply a coupon code on a subscription requisition line

- **GIVEN** a `CatalogItem` has `itemType = subscription`
- **WHEN** a user adds it to a `PurchaseRequisition` and enters a promotion code
- **THEN** `CatalogController.applyPromoCode()` validates the code against active
  `PricingRule` records with `promoCode` set
- **AND** if valid: the discount is applied to the line item total and the promo code
  is stored on the line
- **AND** if invalid: an error message is shown "Promotiecode niet geldig of verlopen"

#### Scenario 2: Promo code expires and is no longer accepted

- **GIVEN** a `PricingRule` with `promoCode` has a `validUntil` date that has passed
- **WHEN** a user enters that promo code on a requisition line
- **THEN** `CatalogController.applyPromoCode()` returns HTTP 422
- **AND** the error message states "Promotiecode verlopen op [datum]"

---

## REQ-PRC-004: Spend Analysis with Category-Level Visibility

Finance controllers and CFOs can view spend distribution by `SpendCategory` across all
matched purchase orders, enabling cross-P2P-cycle analysis.

#### Scenario 1: View category spend on the dashboard

- **GIVEN** a finance controller opens the Spend Analysis dashboard
- **WHEN** the `CnDashboardPage` loads
- **THEN** a bar chart (`CnChartWidget`) shows total `SpendTransaction.amount` grouped
  by `SpendCategory.name` for the selected fiscal year
- **AND** four `CnStatsBlock` KPI cards show:
  - Total spend YTD
  - Number of matched POs
  - Largest single PO
  - Average PO value

#### Scenario 2: Drill down into a spend category

- **GIVEN** a finance controller clicks a bar in the spend-by-category chart
- **WHEN** the drill-down view opens
- **THEN** it shows all `SpendTransaction` records in that category for the period
- **AND** each row links to the originating `PurchaseOrder` and `VendorBill`

#### Scenario 3: SpendTransaction created automatically on bill match

- **GIVEN** `ThreeWayMatchService` sets a `PurchaseOrder.matchStatus = matched`
- **WHEN** the match is confirmed
- **THEN** `ProcurementController` creates a `SpendTransaction` with:
  - `amount` = matched `VendorBill` total
  - `category` = `SpendCategory` derived from the PO's `ProcurementCategory`
  - `purchaseOrder` = the matched PO
  - `vendorBill` = the matched bill
  - `transactionDate` = today
- **AND** the transaction is immediately visible in the Spend Analysis dashboard

#### Scenario 4: Cross-P2P-cycle analysis — sourcing to outcome

- **GIVEN** a finance controller opens a `SpendCategory` detail view
- **WHEN** they select "Inkoopcyclus analyse"
- **THEN** a table shows for each `PurchaseOrder` in that category:
  the originating `PurchaseRequisition` date, PO creation date, receipt date, bill match date,
  and the duration in days for each phase (requisition → PO, PO → receipt, receipt → match)
- **AND** average cycle times per phase are shown in the `CnStatsPanel` header

---

## REQ-PRC-005: High Procurement Visibility for Controllers and Management

Management and finance controllers have a real-time view of all procurement activity,
providing spend control and compliance oversight.

#### Scenario 1: Procurement overview KPI dashboard

- **GIVEN** a finance controller or CFO opens the Shillinq main dashboard
- **WHEN** the procurement widgets load
- **THEN** four `CnStatsBlock` cards show:
  - Open inkoopverzoeken (count and total value)
  - Openstaande inkooporders (count and total value)
  - Niet-gematchte facturen (disputed count)
  - Spend YTD vs budget (percentage bar)

#### Scenario 2: Export spend analysis data

- **GIVEN** a finance controller is on the Spend Analysis dashboard
- **WHEN** they click "Exporteren" and select "Excel" via `CnMassExportDialog`
- **THEN** `ExportService` generates a workbook with `SpendTransaction` data
  grouped by `SpendCategory` and `ProcurementCategory`
- **AND** the export includes: transaction date, PO number, supplier, category, amount, VAT,
  and match status
