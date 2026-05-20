# Spec: Purchase Order Matching & Sequencing

**Change:** catalog-purchase-management-other-t3
**Capability:** po-matching, sale-order-sequencing, sale-stock-sourcing

---

## ADDED Requirements

### REQ-POM-001: 2-way Purchase Order Matching

The system must compare `PurchaseOrder` line items against linked `Invoice` line items for quantity and amount discrepancies within configurable tolerance rules.

#### Scenario: 2-way match succeeds within tolerance

- **GIVEN** a `PurchaseOrder` is linked to an `Invoice`, both with identical line item quantities and amounts within the configured `priceVariancePct`
- **WHEN** `PurchaseOrderMatchingService::match(poId, '2-way')` is called
- **THEN** the match result `status` is `matched`
- **AND** the match result is stored as metadata on the `PurchaseOrder` object
- **AND** no notification is dispatched

#### Scenario: 2-way match fails with price discrepancy

- **GIVEN** a `PurchaseOrder` line item amount differs from the linked `Invoice` amount by more than the configured `priceVariancePct`
- **WHEN** `PurchaseOrderMatchingService::match(poId, '2-way')` is called
- **THEN** the match result `status` is `discrepancy`
- **AND** the `discrepancies` array contains an entry with field `unitPrice`, expected value, and actual value
- **AND** `NotificationService` dispatches a notification to the assigned buyer: "Afwijking gevonden bij inkooporder [PO number]"

#### Scenario: Invoice not yet linked to PO

- **GIVEN** a `PurchaseOrder` has no linked `Invoice`
- **WHEN** `match()` is called
- **THEN** an exception is thrown with message "Geen factuur gekoppeld aan inkooporder"
- **AND** the PO matching status remains unchanged

### REQ-POM-002: 3-way Purchase Order Matching

The system must extend 2-way matching with `GoodsReceipt` comparison — verifying goods received match both what was ordered and what was invoiced.

#### Scenario: 3-way match succeeds

- **GIVEN** a `PurchaseOrder` is linked to both an `Invoice` and a `GoodsReceipt`, with quantities and amounts consistent across all three within tolerance
- **WHEN** `PurchaseOrderMatchingService::match(poId, '3-way')` is called
- **THEN** the match result `status` is `matched`
- **AND** the result indicates all three documents (PO, Invoice, GoodsReceipt) were compared

#### Scenario: 3-way match fails — goods receipt quantity differs

- **GIVEN** the `GoodsReceipt` records fewer units delivered than the `PurchaseOrder` and `Invoice` line
- **WHEN** `match(poId, '3-way')` is called
- **THEN** the match result `status` is `discrepancy`
- **AND** the discrepancy entry identifies field `quantity` with the PO quantity as expected and the GoodsReceipt quantity as actual
- **AND** a buyer notification is dispatched

#### Scenario: Configurable tolerance rules per Administration

- **GIVEN** `Administration.poMatchingConfig` is set to `{ "priceVariancePct": 2, "quantityVarianceUnits": 0 }`
- **WHEN** a 3-way match is performed with a 1.5% price variance and 0 quantity variance
- **THEN** the match result `status` is `matched` (within configured tolerance)
- **WHEN** the price variance is 3%
- **THEN** the match result `status` is `discrepancy` (exceeds tolerance)

### REQ-POM-003: Sale Order Line Sequence Management

Users must be able to reorder line items within a `PurchaseOrder` for display and export purposes.

#### Scenario: Reorder line items by drag-and-drop

- **GIVEN** a `PurchaseOrder` detail page is open in edit mode
- **WHEN** a user drags a line item row to a new position
- **THEN** `SaleOrderSequenceService::reorder()` is called with the updated sequence map
- **AND** the `sequence` field of each affected line item is updated via `ObjectService.saveObject()`
- **AND** the line items are re-rendered in the new order without a full page reload

#### Scenario: Line items exported in sequence order

- **GIVEN** a `PurchaseOrder` with line items that have non-default sequence values
- **WHEN** a user exports the PO to CSV or PDF
- **THEN** line items appear in ascending `sequence` order

### REQ-POM-004: Sale Stock Sourcing

The system must resolve preferred supplier and warehouse for catalog items to support sourcing decisions on purchase orders.

#### Scenario: Resolve stock source for catalog item

- **GIVEN** a `CatalogItem` has a configured stock sourcing rule in app config
- **WHEN** `SaleStockSourcingService::resolveSource(catalogItemId, quantity)` is called
- **THEN** the service returns a `SourcingResult` with preferred supplier name, warehouse location, and estimated lead time in days

#### Scenario: No sourcing rule configured

- **GIVEN** a `CatalogItem` has no matching sourcing rule in app config
- **WHEN** `resolveSource()` is called
- **THEN** the service returns a `SourcingResult` with `supplier: null` and `leadTimeDays: null`
- **AND** the PO detail page shows "Geen voorkeursleverancier geconfigureerd" for that line

#### Scenario: Stock sourcing indicator on PO detail

- **GIVEN** a `PurchaseOrder` line item references a `CatalogItem` with a resolved sourcing result
- **WHEN** a user views the PO detail page
- **THEN** the preferred supplier and estimated lead time are displayed next to the line item
- **AND** a green indicator shows if the item is in stock; orange if on-order
