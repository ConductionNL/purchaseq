# Spec: Purchase Order Management

Entities: `PurchaseOrder`, `PurchaseOrderChange`, `PurchaseOrderRevision`,
`BlanketPurchaseOrder`, `CallOffOrder`, `GoodsReceipt`, `VendorBill`,
`ProcurementOrder`, `Location`, `ProofOfDelivery`, `Supplier`

---

## REQ-PO-001: Purchase Order Creation with Receiving Workflow and Bill Matching

Procurement officers can create a `PurchaseOrder` from an approved `PurchaseRequisition`
or directly (free-text PO). The full receiving workflow links the PO to a `GoodsReceipt`
and a `VendorBill` for three-way matching.

#### Scenario 1: Create a purchase order from an approved requisition

- **GIVEN** a `PurchaseRequisition` has `status = approved`
- **WHEN** a procurement officer clicks "Inkooporder aanmaken" on the requisition detail
- **THEN** a `PurchaseOrder` is created pre-filled with supplier, line items, amounts,
  and delivery location from the requisition
- **AND** the PO has `status = draft` and `matchStatus = pending`
- **AND** the requisition detail page shows the linked PO number

#### Scenario 2: Send a purchase order to the supplier

- **GIVEN** a `PurchaseOrder` has `status = draft` and all required fields are complete
- **WHEN** a procurement officer clicks "Versturen" on the PO detail page
- **THEN** the PO status changes to `sent`
- **AND** a Nextcloud notification is sent to the responsible procurement officer
  (audit trail records the state change via `AuditTrailService`)

#### Scenario 3: Record goods receipt against a purchase order

- **GIVEN** a `PurchaseOrder` has `status = sent` or `confirmed`
- **WHEN** receiving staff click "Ontvangst registreren" on the PO detail page and
  enter received quantities per line item
- **THEN** a `GoodsReceipt` is created linked to the PO
- **AND** received quantities are compared to PO quantities line-by-line
- **AND** if all quantities match: PO `receiptStatus = complete`; if partial: `receiptStatus = partial`

#### Scenario 4: Three-way matching — full match

- **GIVEN** a `PurchaseOrder` has a linked `GoodsReceipt` (quantities confirmed) and
  a linked `VendorBill` (amounts entered)
- **WHEN** `ThreeWayMatchService.match()` is triggered (automatically on VendorBill link)
- **THEN** the service compares PO unit prices × received quantities against VendorBill line amounts
- **AND** if all lines match within a tolerance of 0.01 EUR: `PurchaseOrder.matchStatus = matched`
- **AND** the VendorBill is forwarded to accounts payable for payment processing

#### Scenario 5: Three-way matching — partial / disputed

- **GIVEN** a `VendorBill` amount for one or more lines differs from PO × receipt quantities
  by more than 0.01 EUR
- **WHEN** `ThreeWayMatchService.match()` runs
- **THEN** `PurchaseOrder.matchStatus = disputed`
- **AND** a notification is sent to the procurement officer and the accounts payable clerk
- **AND** the PO detail page shows a per-line match status indicating which lines differ

---

## REQ-PO-002: Purchase Order Management with Three-Way Matching Overview

The PO index provides a real-time overview of all purchase orders with their three-way match
status so procurement officers can prioritise follow-up actions.

#### Scenario 1: PO index shows match status at a glance

- **GIVEN** a procurement officer opens the purchase orders index page
- **WHEN** the `CnDataTable` renders
- **THEN** each row shows order number, supplier, total amount, status, and `matchStatus`
  rendered as a `CnStatusBadge` (matched=green, partial=orange, disputed=red, pending=grey)
- **AND** the table is sortable by `matchStatus` to surface disputed orders first

#### Scenario 2: Filter POs by match status

- **GIVEN** a procurement officer is on the purchase orders index
- **WHEN** they select `matchStatus = disputed` in the `CnFilterBar`
- **THEN** only disputed POs are shown
- **AND** the count of disputed POs is displayed in the `CnActionsBar` filter summary

---

## REQ-PO-003: Blanket PO Management with Release Scheduling and Consumption Monitoring

`BlanketPurchaseOrder` objects represent standing agreements with a supplier up to a
maximum amount. `CallOffOrder` records represent individual releases against the blanket PO.

#### Scenario 1: Create a blanket purchase order

- **GIVEN** a procurement officer opens the blanket PO index and clicks "Toevoegen"
- **WHEN** they fill in supplier, title, `validFrom`, `validUntil`, and `maxAmount`
  in the `CnFormDialog`
- **THEN** a `BlanketPurchaseOrder` is saved with `releasedAmount = 0`, `consumedAmount = 0`
- **AND** it appears on the index with a consumption progress bar at 0%

#### Scenario 2: Release a call-off order against a blanket PO

- **GIVEN** a `BlanketPurchaseOrder` is active with remaining headroom
- **WHEN** a procurement officer creates a `CallOffOrder` linked to the blanket PO
  with an amount and expected delivery date
- **THEN** `BlanketPurchaseOrder.releasedAmount` is incremented by the call-off amount
- **AND** if `releasedAmount` would exceed `maxAmount`, the save is rejected with HTTP 422
  and message "Vrijgegeven bedrag overschrijdt maximum van raambestelling"

#### Scenario 3: Consumption monitoring on the detail page

- **GIVEN** a user opens a `BlanketPurchaseOrder` detail page
- **WHEN** the page renders
- **THEN** a `CnProgressBar` shows `consumedAmount / maxAmount` as a percentage
- **AND** below it a table lists all `CallOffOrder` records with their status and amount
- **AND** when consumption exceeds 80% of `maxAmount`, an orange warning badge appears

#### Scenario 4: Limit purchase orders for service-based procurement with periodic invoicing

- **GIVEN** a `BlanketPurchaseOrder` is of type `service` (periodic invoicing)
- **WHEN** a `VendorBill` is matched to a `CallOffOrder` linked to that blanket PO
- **THEN** `consumedAmount` on the blanket PO is incremented by the matched bill amount
- **AND** the remaining headroom (`maxAmount − consumedAmount`) is shown on the detail page

---

## REQ-PO-004: Multi-Location Purchase Orders with Location-Specific Delivery Addresses

A single `PurchaseOrder` or `BlanketPurchaseOrder` can specify different `Location` entities
as delivery addresses per line item, supporting organisations with 30+ sites.

#### Scenario 1: Set a delivery location per PO line item

- **GIVEN** a procurement officer is creating or editing a `PurchaseOrder`
- **WHEN** they expand a line item and select a `Location` from the delivery location picker
- **THEN** the selected `Location` (name, address) is stored on the PO line item
- **AND** if no line-level location is set, the PO-level `deliveryLocation` is used

#### Scenario 2: PO confirmation shows per-line delivery addresses

- **GIVEN** a `PurchaseOrder` has line items with different delivery locations
- **WHEN** the PO detail page renders
- **THEN** each line item row shows its delivery location name
- **AND** the PO summary section shows "Meerdere leverlocaties" instead of a single address

#### Scenario 3: Multi-entity procurement for shared service centers

- **GIVEN** a shared service center (SSC) procurement officer creates a `PurchaseOrder`
  on behalf of multiple client organisations
- **WHEN** they set `organisation` to the SSC entity and add multiple line items each
  with a different `organisation` field referencing client entities
- **THEN** the PO is saved with cross-entity line items
- **AND** each line item's cost allocation is attributed to the correct client entity
- **AND** `AuthorizationService` verifies the procurement officer has SSC-level permission

---

## REQ-PO-005: Receiving Logs for Confirming Delivery Against PO Line Items

Receiving staff can confirm delivery against specific PO line items and record discrepancies,
generating a `GoodsReceipt` record linked to the PO.

#### Scenario 1: Confirm partial delivery against a PO

- **GIVEN** a `PurchaseOrder` has `status = sent` and a supplier delivers part of the order
- **WHEN** receiving staff click "Ontvangst registreren" and enter received quantities
  that are less than ordered for one or more lines
- **THEN** a `GoodsReceipt` is created with `type = partial`
- **AND** the PO line items show remaining quantity outstanding
- **AND** `PurchaseOrder.receiptStatus = partial`

#### Scenario 2: Record a delivery discrepancy

- **GIVEN** receiving staff are recording a goods receipt
- **WHEN** they mark a line item as "afwijkend" (discrepant) and enter a reason
- **THEN** the discrepancy is stored in the `GoodsReceipt.discrepancies` array
- **AND** a notification is sent to the procurement officer
- **AND** the PO `matchStatus` is set to `disputed` pending investigation

#### Scenario 3: Generate proof of delivery

- **GIVEN** a `GoodsReceipt` has been saved with `type = complete`
- **WHEN** a user clicks "Afleveringsbewijs" on the goods receipt detail page
- **THEN** a `ProofOfDelivery` record is created referencing the `GoodsReceipt`
- **AND** it is accessible in the Files tab of the PO's `CnObjectSidebar`

---

## REQ-PO-006: Purchase Order Templates for Standardised and Recurring Procurement

Procurement officers can save a `PurchaseOrder` as a template and use it to create new
orders with pre-filled supplier, line items, and delivery location.

#### Scenario 1: Save a PO as a template

- **GIVEN** a procurement officer is on a `PurchaseOrder` detail page
- **WHEN** they click "Opslaan als sjabloon" and provide a template name
- **THEN** a copy of the PO is saved as a template (linked via `PurchaseOrderRevision`)
  with `isTemplate = true`
- **AND** the template appears in a "Sjablonen" filter tab on the PO index page

#### Scenario 2: Create a new PO from a template

- **GIVEN** a procurement officer opens the PO index and selects a template
- **WHEN** they click "Bestelling aanmaken"
- **THEN** a new `PurchaseOrder` is created pre-filled with the template's supplier,
  line items, and delivery location, with a new order number and `status = draft`
- **AND** the new PO is not linked to any requisition unless the user explicitly links one

---

## REQ-PO-007: Free-Text Purchase Orders for Non-Catalog Items

Procurement officers can create purchase orders for items not in the internal catalog,
with automatic routing to the appropriate approval chain based on category.

#### Scenario 1: Create a free-text PO for a non-catalog service

- **GIVEN** a procurement officer clicks "Vrije-tekst inkooporder" on the PO index
- **WHEN** they enter supplier, description, `ProcurementCategory`, quantity, and price
  without selecting a `CatalogItem`
- **THEN** a `PurchaseOrder` is saved with `isFreeText = true`
- **AND** `ApprovalRoutingService` assigns an approval chain based on the selected category

#### Scenario 2: Free-text PO triggers category-based approval

- **GIVEN** a free-text `PurchaseOrder` has `status = draft`
- **WHEN** the procurement officer submits it for approval
- **THEN** `ApprovalRoutingService` resolves the appropriate `ApprovalChain` by category
- **AND** `ApprovalRequest` records are created and approvers are notified

---

## REQ-PO-008: Transparent and Competitive Procurement Process — Procedure Type Selection

When a requisition or PO exceeds an EU procurement threshold, the system enforces selection
of the correct procurement procedure type and maintains a reference table of thresholds.

#### Scenario 1: Select procurement procedure type for a purchase order

- **GIVEN** a `PurchaseOrder` has `totalAmount` set
- **WHEN** the procurement officer selects a `ProcurementProcedure` from the dropdown
- **THEN** the selected procedure type is stored on the PO
- **AND** `ProcurementThresholdService` validates that the selected procedure is
  appropriate for the order value and category (returns HTTP 422 if not)

#### Scenario 2: Maintain EU procurement threshold reference table

- **GIVEN** an administrator navigates to the Procurement settings page
- **WHEN** they open the "EU drempelwaarden" section
- **THEN** the current EU procurement thresholds per procedure type and category are shown
  as a read-only table sourced from `ProcurementProcedure` seed objects
- **AND** thresholds are updated by re-importing the `shillinq_register.json` seed data
  (no manual edit required in the UI)

#### Scenario 3: Track procurement procedure type usage

- **GIVEN** `PurchaseOrder` objects carry a `procurementProcedure` field
- **WHEN** a finance controller opens the Spend Analysis dashboard
- **THEN** a procedure type distribution chart shows the count and total value of orders
  per procedure type for the current year
