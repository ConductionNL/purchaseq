# Spec: Quotation Management

**Change:** catalog-purchase-management-other-t3
**Capability:** quotation-management

---

## ADDED Requirements

### REQ-QUO-001: Create Professional Quote

Users must be able to create a professional `Quote` object with line items, pricing (including discounts from `PricingRuleService`), validity date, and customer details — and generate a professional PDF.

#### Scenario: Create new quote

- **GIVEN** a user navigates to Offertes → Nieuw
- **WHEN** the user fills in customer, title, line items, validity date, and notes
- **THEN** a new `Quote` object is created with `status: "concept"` and `revisionNumber: 1`
- **AND** `PricingRuleService::applyRules()` is called to apply any applicable discounts to the quote lines
- **AND** the total amount (including discounts) is calculated and stored

#### Scenario: Generate professional PDF quote

- **GIVEN** a `Quote` object exists with all required fields populated
- **WHEN** a user clicks "PDF genereren" on the quote detail page
- **THEN** the `docudesk` PDF generation service is called with the Shillinq `shillinq-quote-v1` template
- **AND** the PDF includes: quote number, customer name/address, issue date, expiry date, line items table with quantities/prices/discounts, totals summary, and payment/delivery terms
- **AND** the generated PDF is attached to the `Quote` object via `FileService`
- **AND** the PDF is available for download from the Files tab in `CnObjectSidebar`

#### Scenario: Quote missing required fields — validation

- **GIVEN** a user attempts to save a `Quote` without a customer or expiry date
- **WHEN** the form is submitted
- **THEN** validation errors are shown for the missing fields
- **AND** the `Quote` object is not created
- **AND** the user remains on the form with error highlighting (WCAG AA accessible)

### REQ-QUO-002: Quotation Revision

Users must be able to create a new revision of an existing `Quote`, maintaining a linked revision chain for audit and negotiation tracking.

#### Scenario: Create quote revision

- **GIVEN** a `Quote` exists with `status: "verzonden"` or `status: "herzien"`
- **WHEN** a user clicks "Nieuwe revisie maken"
- **THEN** `QuoteRevisionService::createRevision()` is called
- **AND** a new `Quote` object is created with all fields copied from the original
- **AND** the new quote's `revisionNumber` is the original's `revisionNumber` + 1
- **AND** the new quote's `previousVersion` OpenRegister relation points to the original quote
- **AND** the original quote's `status` transitions to `herzien`
- **AND** the new revision's `status` is `concept`

#### Scenario: View revision chain on quote detail

- **GIVEN** a `Quote` has one or more revisions linked via `previousVersion`
- **WHEN** a user views any quote in the chain
- **THEN** a "Revisiehistorie" section in `CnDetailCard` lists all revisions with revision number, issue date, and status
- **AND** each revision is a clickable link navigating to that revision's detail page

#### Scenario: Revise quote that is already accepted or rejected

- **GIVEN** a `Quote` with `status: "geaccepteerd"` or `status: "afgewezen"`
- **WHEN** a user attempts to create a revision
- **THEN** the "Nieuwe revisie maken" button is not displayed
- **AND** the quote is in a terminal state — no further revision is possible

### REQ-QUO-003: Track Quote Status

Users must be able to track the lifecycle status of a `Quote` through defined transitions and see the current status at a glance.

#### Scenario: View quote status in list

- **GIVEN** multiple `Quote` objects with different statuses exist
- **WHEN** a user views the Offertes index page
- **THEN** each quote displays a `CnStatusBadge` with the current status: concept (grijs), verzonden (blauw), herzien (oranje), geaccepteerd (groen), afgewezen (rood), verlopen (grijs)

#### Scenario: Status timeline on quote detail

- **GIVEN** a user opens a `Quote` detail page
- **WHEN** the detail view renders
- **THEN** a `CnTimelineStages` component shows the status progression: Concept → Verzonden → Geaccepteerd / Afgewezen / Verlopen
- **AND** the current status stage is highlighted
- **AND** completed stages show the timestamp of the transition

#### Scenario: Quote expires automatically

- **GIVEN** a `Quote` with `status: "verzonden"` has an `expiryDate` in the past
- **WHEN** the Shillinq background job `QuoteExpiryJob` runs (daily)
- **THEN** the `Quote` status transitions to `verlopen`
- **AND** the requester receives a notification: "Offerte [quoteNumber] is verlopen"

#### Scenario: Filter quotes by status

- **GIVEN** a user is on the Offertes index page
- **WHEN** the user selects "Geaccepteerd" in the status filter
- **THEN** only `Quote` objects with `status: "geaccepteerd"` are shown
- **AND** the total count shown matches the filtered result count
