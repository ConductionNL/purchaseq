# Spec: Purchase Requisition Management

**Change:** catalog-purchase-management-other-t3
**Capability:** purchase-requisition

---

## ADDED Requirements

### REQ-REQ-001: Create Purchase Requisition

Users must be able to create a `PurchaseRequisition` with a customizable intake form that adapts to the selected `ProcurementCategory`, improving adoption and reducing form friction.

#### Scenario: Create requisition from scratch

- **GIVEN** a user navigates to Inkoopverzoeken → Nieuw
- **WHEN** the user selects a `ProcurementCategory` from the dropdown
- **THEN** the intake form dynamically renders standard fields (title, justification, cost center, urgency, requestDate) plus any custom fields defined in `ProcurementCategory.formTemplate`
- **AND** required custom fields are validated before submission is allowed
- **AND** all labels and field descriptions are displayed in the user's locale (via `t(appName, 'key')`)

#### Scenario: Create requisition from catalog basket

- **GIVEN** a user has checked out a basket from the catalog
- **WHEN** the requisition creation form opens
- **THEN** line items from the basket are pre-populated in the requisition
- **AND** the cost center selected during basket checkout is pre-filled
- **AND** the user can add, remove, or modify line items before submission

#### Scenario: Customizable form fields per category

- **GIVEN** a `ProcurementCategory` has a non-empty `formTemplate` JSON Schema
- **WHEN** a user creates a requisition for that category
- **THEN** additional form fields defined in the template are rendered via `CnAdvancedFormDialog`
- **AND** custom field values are stored in `PurchaseRequisition.customFields` as JSON
- **AND** custom fields are displayed in the requisition detail view

### REQ-REQ-002: Submit and Route Requisition for Approval

Submitted requisitions must be routed automatically to the correct approvers via `ApprovalChain` based on cost center and amount thresholds.

#### Scenario: Submit requisition

- **GIVEN** a user has completed the requisition form
- **WHEN** the user clicks "Indienen"
- **THEN** `RequisitionApprovalService::submit()` is called
- **AND** the appropriate `ApprovalChain` is resolved based on `CostCenter` and `totalAmount`
- **AND** an `ApprovalRequest` is created linked to the requisition
- **AND** `ApprovalTask` objects are created for each approval step
- **AND** approvers receive a Nextcloud notification via `NotificationService`
- **AND** the requisition status transitions from `concept` to `ingediend`

#### Scenario: No approval chain configured

- **GIVEN** no `ApprovalChain` is configured for the cost center and amount of the requisition
- **WHEN** the user submits the requisition
- **THEN** a warning notification is shown: "Geen goedkeuringsroute gevonden — verzoek staat open voor handmatige afhandeling"
- **AND** the requisition status transitions to `ingediend` without creating `ApprovalTask` objects
- **AND** an admin notification is dispatched via `NotificationService`

### REQ-REQ-003: Approve or Reject Requisition

Approvers must be able to review and approve or reject `PurchaseRequisition` objects via their `ApprovalTask` inbox.

#### Scenario: Approve requisition

- **GIVEN** an approver has a pending `ApprovalTask` linked to a `PurchaseRequisition`
- **WHEN** the approver opens the task and clicks "Goedkeuren"
- **THEN** the `ApprovalTask` status transitions to `goedgekeurd`
- **AND** if all tasks in the `ApprovalChain` are approved, the requisition status transitions to `goedgekeurd`
- **AND** the requester receives a Nextcloud notification: "Uw inkoopverzoek [title] is goedgekeurd"

#### Scenario: Reject requisition

- **GIVEN** an approver has a pending `ApprovalTask` linked to a `PurchaseRequisition`
- **WHEN** the approver enters a rejection reason and clicks "Afwijzen"
- **THEN** the `ApprovalTask` status transitions to `afgewezen`
- **AND** the requisition status transitions to `afgewezen`
- **AND** the requester receives a notification with the rejection reason
- **AND** subsequent approval tasks in the chain are cancelled

### REQ-REQ-004: Track Purchase Order Status

Users must be able to track the status of a `PurchaseOrder` through its lifecycle from creation to delivery and payment.

#### Scenario: View PO status timeline

- **GIVEN** a `PurchaseOrder` exists with a current status
- **WHEN** a user opens the PO detail page
- **THEN** a `CnTimelineStages` component displays the status progression: Concept → Verzonden → Bevestigd → In Levering → Ontvangen → Gefactureerd → Betaald
- **AND** the current status is highlighted
- **AND** timestamps for completed stages are shown where available

#### Scenario: Filter POs by status

- **GIVEN** a user is on the PurchaseOrders index page
- **WHEN** the user selects a status filter from the filter bar
- **THEN** only `PurchaseOrder` objects with the matching status are displayed
- **AND** the total count per status is shown as filter facet counts
