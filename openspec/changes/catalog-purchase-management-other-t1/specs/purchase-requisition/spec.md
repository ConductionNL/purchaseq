# Spec: Purchase Requisition

Entities: `PurchaseRequisition`, `ApprovalChain`, `ApprovalRoute`, `ApprovalRequest`,
`ApprovalTask`, `CostAllocation`, `ProcurementCategory`, `ProcurementProcedure`

---

## REQ-REQ-001: Purchase Requisition Creation with Multi-Line Items and Custom Fields

Procurement users can create a `PurchaseRequisition` containing multiple line items, each
referencing a `CatalogItem` or entered as free text, with quantity, unit price, cost
allocation, and procurement category per line.

#### Scenario 1: Create a multi-line purchase requisition from catalog items

- **GIVEN** a user clicks "Inkoopverzoek toevoegen" on the requisitions index page
- **WHEN** they fill in the `CnFormDialog` with title, description, required-by date,
  and add at least one line item referencing a `CatalogItem`
- **THEN** a `PurchaseRequisition` is saved with `status = draft`
- **AND** line items carry `catalogItem`, `quantity`, `unitPrice`, `totalLineAmount`,
  and `costAllocation` fields
- **AND** `totalAmount` on the requisition equals the sum of all `totalLineAmount` values

#### Scenario 2: Add a free-text line item for a non-catalog product

- **GIVEN** a user is creating a `PurchaseRequisition`
- **WHEN** they add a line item without selecting a `CatalogItem` and enter description,
  quantity, and unit price manually
- **THEN** the line item is saved with `catalogItem = null` and the user-entered values
- **AND** the requisition is tagged as requiring category-based approval routing
  because no pre-approved catalog item is referenced

#### Scenario 3: Set cost allocation per requisition line

- **GIVEN** a user is editing a requisition line item
- **WHEN** they fill in the `CostAllocation` fields (cost center, project, percentage)
- **THEN** the cost allocation is stored as a nested object on the line item
- **AND** total allocated percentages across all cost centers on that line must equal 100%
  (validated by `CatalogController` before save; returns HTTP 422 otherwise)

#### Scenario 4: Requisition total triggers EU procurement threshold check

- **GIVEN** a user submits a `PurchaseRequisition` with `totalAmount > €0`
- **WHEN** the requisition status changes from `draft` to `submitted`
- **THEN** `ProcurementThresholdService` compares `totalAmount` to the applicable
  EU procurement threshold for the selected `ProcurementCategory`
- **AND** the applicable `ProcurementProcedure` type is stored on the requisition
  (e.g. `enkelvoudig onderhands`, `meervoudig onderhands`, `openbaar`)
- **AND** if the amount exceeds the European threshold for that category, a warning
  notification is sent to the procurement officer via `NotificationService`

---

## REQ-REQ-002: Requisition Approval Workflow Routing

Submitted `PurchaseRequisition` records are automatically routed through an `ApprovalChain`
selected by `ApprovalRoutingService` based on category, total amount, and organisation.
Approvers receive Nextcloud notifications and can approve or reject from the requisition detail.

#### Scenario 1: Automatic approval chain selection on submit

- **GIVEN** a user sets a requisition status to `submitted`
- **WHEN** `ApprovalRoutingService.resolveChain()` evaluates the requisition
- **THEN** it selects the `ApprovalChain` whose `procurementCategory`, `minAmount`,
  `maxAmount`, and `organisation` fields match the requisition
- **AND** `ApprovalRequest` records are created for each step in the chain
- **AND** the first approver receives a Nextcloud notification via `NotificationService`

#### Scenario 2: Approver approves a requisition step

- **GIVEN** an approver opens a `PurchaseRequisition` detail page where they have a
  pending `ApprovalRequest`
- **WHEN** they click "Goedkeuren" and optionally add a comment
- **THEN** the `ApprovalRequest` status changes to `approved`
- **AND** if more steps remain in the chain, the next approver is notified
- **AND** if this was the final step, the requisition status changes to `approved`
  and the requestor is notified

#### Scenario 3: Approver rejects a requisition

- **GIVEN** an approver has a pending `ApprovalRequest` for a requisition
- **WHEN** they click "Afwijzen" and enter a rejection reason
- **THEN** the `ApprovalRequest` status changes to `rejected`
- **AND** the requisition status changes to `rejected`
- **AND** the requestor receives a notification containing the rejection reason

#### Scenario 4: Approval chain progress is visible on the detail page

- **GIVEN** a user opens a requisition that is in `pending_approval` status
- **WHEN** the detail page renders
- **THEN** `CnTimelineStages` shows each approval step with status colour:
  green (approved), orange (pending), grey (not yet reached), red (rejected)
- **AND** the name of the assigned approver for each step is shown

---

## REQ-REQ-003: Procurement Timeline with Legal Deadlines

`PurchaseRequisition` objects can carry a procurement timeline with legal deadlines per
procedure type. The system warns when a deadline is at risk.

#### Scenario 1: Set legal deadline on a requisition

- **GIVEN** a user is editing a `PurchaseRequisition`
- **WHEN** they enter a `legalDeadline` date in the timeline section
- **THEN** the date is stored on the requisition and displayed on the detail page
- **AND** if `legalDeadline` is within 14 days from today, a warning badge is shown on
  the index page row

#### Scenario 2: Deadline warning notification

- **GIVEN** a `PurchaseRequisition` with status `pending_approval` has a `legalDeadline`
- **WHEN** the date is 7 days away and the requisition is not yet approved
- **THEN** `NotificationService` sends a warning to the procurement officer and the
  current approver
- **AND** the notification links directly to the requisition detail page

#### Scenario 3: Track EU procurement procedure type usage trends

- **GIVEN** requisitions carry a `procurementProcedure` field (set by REQ-REQ-001, Scenario 4)
- **WHEN** a finance controller opens the Spend Analysis dashboard
- **THEN** a `CnChartWidget` (donut chart) shows the distribution of procedure types
  used (openbaar / meervoudig onderhands / enkelvoudig onderhands) for the current fiscal year
- **AND** the chart is filterable by `ProcurementCategory`

---

## REQ-REQ-004: Search and Select Pre-Approved Catalog Items in a Requisition

When a user creates a requisition line item, they can search the internal catalog to find
pre-approved items with negotiated pricing, avoiding ad-hoc purchases.

#### Scenario 1: Search catalog from within a requisition form

- **GIVEN** a user is adding a line item to a `PurchaseRequisition`
- **WHEN** they type in the catalog item search field on the line item form
- **THEN** `IndexService` queries `CatalogItem` records filtered to active, non-expired items
- **AND** results appear as a dropdown showing name, SKU, unit price, and supplier
- **AND** selecting an item auto-fills `unitPrice`, `unit`, and `supplier` on the line

#### Scenario 2: Only pre-approved items appear in catalog search

- **GIVEN** a `CatalogItem` has `status = inactive` or its parent catalog has expired
- **WHEN** a user searches for that item in the requisition form
- **THEN** the item does not appear in the search results
- **AND** a free-text line item must be used instead
