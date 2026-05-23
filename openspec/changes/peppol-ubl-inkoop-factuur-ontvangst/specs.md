---
status: specs
app: purchaseq
spec: peppol-ubl-inkoop-factuur-ontvangst
created: 2026-05-23
---

# Specifications: Peppol UBL Inkoopfactuur Ontvangst en Drie-weg-match

## Functional Requirements

---

### REQ-001-010: Inbound Receipt & Schema Validation

**Acceptance Criteria:**

GIVEN a Peppol AS4 message arrives via openconnector peppol-e-invoicing-adapter  
WHEN the message is addressed to a Peppol Participant ID registered in this organization  
THEN:
- The system stores the original UBL document in `InboundPeppolFactuur.ubl_document_raw` (immutable, never overwritten)
- The system parses and extracts core metadata (document_type, invoice_number, date, totals, vendor Peppol ID)
- The system validates the UBL document against **Peppol BIS Billing 3.0** XML Schema
- The system validates against **NLCIUS conformance rules** (Dutch mandatory requirements)
- The system records validation results in `InboundPeppolFactuur.validation_status` and `validation_errors`
- IF all validations pass: `validation_status = "valide"`, no errors, invoice proceeds to vendor matching (REQ-002)
- IF any validation fails: `validation_status = "ongeldig"`, validation error objects are populated, a Peppol MLR (Message Level Response) negative is generated and sent back to the sender via the AP

**Non-functional:**
- Validation must complete within 5 seconds
- MLR response must be sent within 30 seconds
- UBL payload must be stored exactly as received (no encoding conversion, no whitespace normalization)

---

### REQ-002-010: Vendor Matching (Peppol ID → KvK → Proposal)

**Acceptance Criteria:**

GIVEN a validated invoice with `validation_status = "valide"`  
WHEN the matching service starts processing the invoice  
THEN:
- The system attempts to match the vendor using the Peppol Participant Identifier (sender_peppol_id from BT-7 CompanyID)
- IF an exact match is found: vendor is linked, `FactuurHeader.vendor_id` is set, proceed to PO matching (REQ-003)
- IF no exact match on Peppol ID: the system attempts secondary match using the KvK number extracted from the EndpointID field (BT-XX-EndpointID-scheme="0192")
- IF KvK match found: vendor is linked, `FactuurHeader.vendor_id` is set, proceed to PO matching (REQ-003)
- IF no match on either Peppol ID or KvK: the system creates an `ApprovalTaak` record with type="voorstel-nieuwe-leverancier" assigned to a krediteuren-administrateur role, populated with:
  - Extracted vendor name (BT-7-Name)
  - VAT number (BT-7-TaxID, if present)
  - IBAN (BT-84, if present)
  - Contact email (BT-7-Contact, if present)
  - Full UBL extract as JSON for review
- The invoice remains in `status = "in-behandeling"` until the krediteuren-administrateur approves the new vendor
- ONCE the new vendor is approved: the system re-triggers vendor matching (now succeeds) and proceeds to PO matching
- ONCE the new vendor is rejected: a Peppol MLR negative is generated and sent back; invoice is marked `status = "ongeldig"`

**Non-functional:**
- Vendor matching must complete within 3 seconds
- New vendor proposal must appear in the krediteuren-administrateur's task list within 10 seconds

---

### REQ-003-010: PO Lookup & Validation via BT-13 Reference

**Acceptance Criteria:**

GIVEN an invoice with a matched `vendor_id` (from REQ-002)  
WHEN the PO matching service processes the invoice  
THEN:
- The system extracts the PO reference from the invoice header field **BT-13 (PurchaseOrderReference)**
- IF BT-13 is null or empty: the invoice is moved to a "no-PO" queue; a krediteuren-administrateur can manually classify it as a "kostenfactuur zonder PO" or attach a PO retroactively; if classified as no-PO, the invoice proceeds to line-level validation only (2-way match skipped, GR not required)
- IF BT-13 is present: the system queries the purchaseq PO registry by the reference string
- IF no PO is found: the system creates an `ApprovalTaak` with type="po-niet-gevonden" assigned to a krediteuren-administrateur, displaying the PO reference and asking for manual resolution (attach correct PO, or reclassify as no-PO)
- IF a PO is found: the system validates that the vendor on the PO matches the vendor on the invoice (`FactuurHeader.vendor_id` == `PO.vendor_id`)
  - IF vendors match: `FactuurHeader.matched_po_id` is set, invoice proceeds to GR lookup (REQ-004)
  - IF vendors do NOT match: a critical error is logged, the invoice is escalated to decidesk with reason "vendor-mismatch-po-invoice", and processing stops

**Non-functional:**
- PO lookup must complete within 2 seconds
- Manual resolution task must appear within 10 seconds

---

### REQ-004-010: GR (Goederen-Ontvangst) Lookup

**Acceptance Criteria:**

GIVEN an invoice with a matched `matched_po_id` (from REQ-003)  
WHEN the GR lookup service processes the invoice  
THEN:
- The system queries the purchaseq GR registry for registrations linked to the matched PO
- IF one or more GR records exist for the PO: the system attempts to match each invoice line to a GR line by purchase order line reference
  - For each matched pair, the system records the match in `MatchingPoging.matched_gr_line_id`
  - GR is optional; if some lines have no GR match, those lines proceed to 2-way match (invoice ↔ PO, skipping GR-qty)
- IF no GR exists for the PO: the invoice proceeds to 2-way or 3-way matching using available PO data (GR comparison is skipped)
- IF GR exists but is dated in the future: the system logs a warning and treats the GR as "not yet complete"; line matching uses PO qty only

**Non-functional:**
- GR lookup and matching must complete within 3 seconds

---

### REQ-005-010: Three-Way Match with Configurable Tolerances

**Acceptance Criteria:**

GIVEN an invoice with a matched `matched_po_id` and optional `matched_gr_line_id`  
WHEN the line-level matching service processes each invoice line  
THEN for **each FactuurRegel**:
- The system finds the corresponding PO line (matched by line_number or article_code, configurable)
- The system compares:
  - **Quantity:** `invoice_qty` vs. `po_qty` (and GR qty if present)
  - **Unit Price:** `invoice_unit_price` vs. `po_unit_price`
- The system loads the applicable `MatchingTolerantie` rule (organization-wide, or by cost_category if configured)
- The system calculates:
  - `price_difference_amount = invoice_unit_price - po_unit_price`
  - `price_difference_percentage = (price_difference_amount / po_unit_price) * 100`
  - `quantity_difference = invoice_qty - po_qty`
- The system applies tolerance thresholds:
  - `within_tolerance = (ABS(price_diff_pct) <= tolerance_pct OR ABS(price_diff_amt) <= tolerance_abs) AND (ABS(qty_diff) <= qty_tolerance)`
- The system creates a `MatchingPoging` record with:
  - `match_status = "volledig"` if qty_diff=0 and price_diff=0
  - `match_status = "gedeeltelijk"` if one of qty/price differs but both within tolerance
  - `match_status = "geen"` if no PO line is found for this invoice line
- **Conclusion logic:**
  - IF `match_status = "volledig" AND within_tolerance = true` → `conclusion = "auto-akkoord"` (no manual approval needed)
  - IF `match_status = "gedeeltelijk" AND within_tolerance = true` → `conclusion = "wacht-op-accordering"` (create ApprovalTaak)
  - ELSE → `conclusion = "wacht-op-accordering"` (create ApprovalTaak)
- The system records the tolerance check details in `MatchingPoging.tolerance_check_details` for audit
- Once all invoice lines are processed:
  - IF all lines have `conclusion = "auto-akkoord"` → invoice proceeds to "ready for payment" (REQ-006)
  - IF any line has `conclusion = "wacht-op-accordering"` → ApprovalTaak records are created for PO-eigenaar review (REQ-005-bis)

**Non-functional:**
- Per-line matching must complete within 500 ms per line (100 lines in 50 seconds)
- Tolerance configuration must allow per-cost-category customization
- Tolerance ranges: 0.0% to 10.0% (price %), 0 to 999 units (qty)

---

### REQ-005-bis-010: Guided Exception Workflow for Mismatches

**Acceptance Criteria:**

GIVEN an invoice with one or more lines in `conclusion = "wacht-op-accordering"` state  
WHEN a PO-eigenaar (Inkoper or Budgethouder) opens their Approval task list  
THEN:
- An `ApprovalTaak` is created for each mismatched line, assigned to the PO-eigenaar associated with the PO cost_center
- The task includes a deadline of **5 business days** from creation (configurable per organization)
- When the PO-eigenaar opens the task detail view, the system displays a **three-column side-by-side comparison:**
  - **Column 1 (PO):** PO line number, qty, unit_price, line_amount, cost_center
  - **Column 2 (GR):** GR receipt date, received_qty, received_date (if exists)
  - **Column 3 (Invoice):** Invoice line number, qty, unit_price, line_amount, VAT, [difference percentages highlighted in red]
- The system displays exactly **three action buttons:**
  1. **"Accordeer"** → Opens a text area for the PO-eigenaar to enter a reason (e.g., "Leverancier bevestigd prijsaanpassung via e-mail"), saves the approval, sets `ApprovalTaak.status = "geaccordeerd"` and `action_taken = "accordeer"`
  2. **"PO bijwerken"** → Opens an inline edit form to modify the PO line unit_price or qty, saves the change to purchaseq base, automatically re-triggers line matching for that invoice, and if all lines now pass, skips further approval (auto-advances invoice to "ready for payment")
  3. **"Betwisten"** → Opens a dispute form where the PO-eigenaar selects a reason (duplicate | invalid_charge | qty_mismatch | other) and enters a message; on submission, creates a `FactuurDispuut` record and triggers Peppol response generation (REQ-007)
- Once a decision is made, `ApprovalTaak.treated_at`, `treated_by`, and `action_taken` are recorded
- If ALL lines in an invoice are now approved (either auto-akkoord or geaccordeerd), the invoice automatically proceeds to "ready for payment" (REQ-006)

**Non-functional:**
- Task detail page must load within 2 seconds
- Re-matching after PO update must complete within 3 seconds
- Approval action must save and trigger next step within 1 second

---

### REQ-006-010: Submission to shillinq for Payment

**Acceptance Criteria:**

GIVEN an invoice with all `FactuurRegel` lines in either `conclusion = "auto-akkoord"` or `"geaccordeerd"` state  
WHEN the invoice status transitions to "ready-for-payment"  
THEN:
- The system verifies that every line has been approved (no pending ApprovalTaak without a decision)
- The system locks the `FactuurHeader` and all `FactuurRegel` records for write-only (reading is allowed, modification is blocked)
- The system extracts and validates the payment information:
  - `payment_account_iban` from `FactuurHeader` (BT-84)
  - `total_incl_vat` from `FactuurHeader`
  - `due_date` from `FactuurHeader` (BT-9)
  - `early_payment_discount_percent` if present (BT-20)
  - `payment_reference` (vendor invoice number for reconciliation)
- IF any required payment field is missing or invalid: the invoice is escalated to decidesk with reason "payment-data-incomplete"
- ELSE: The system creates an `InvoiceReadyForPayment` event and publishes it to the shillinq event stream with the complete payment DTO:
  ```json
  {
    "factuur_header_id": "uuid",
    "vendor_id": "uuid",
    "vendor_name": "string",
    "vendor_iban": "string",
    "invoice_amount_incl_vat": 1815.00,
    "invoice_currency": "EUR",
    "due_date": "2026-06-10",
    "payment_reference": "F-2026-004521",
    "early_payment_discount_percent": 2.0,
    "early_payment_deadline": "2026-05-25",
    "verified": true,
    "audit_trail": {
      "po_reference": "PO-2026-001234",
      "matched_at": "2026-05-15T14:32:30Z",
      "all_lines_approved": true
    }
  }
  ```
- The system sets `FactuurHeader.status = "submitted-to-payment"`
- shillinq receives the event and creates a payment instruction scheduled for `due_date - X days` (configurable per organization, typically 2-5 days before due date)
- The system publishes a `FactuurSubmittedToPayment` event for audit and archival (REQ-009)

**Non-functional:**
- Transition to "ready-for-payment" must complete within 2 seconds
- Event publishing must occur within 5 seconds
- shillinq must receive the event and acknowledge within 60 seconds; if not acknowledged, escalate to decidesk

---

### REQ-007-010: Dispute Workflow & Peppol Response

**Acceptance Criteria:**

GIVEN an invoice that is disputed (via ApprovalTaak action "Betwisten" or manual escalation)  
WHEN the PO-eigenaar or krediteuren-administrateur confirms the dispute  
THEN:
- A `FactuurDispuut` record is created with:
  - `dispute_reason` (enum: duplicate | invalid_charge | qty_mismatch | other)
  - `dispute_message` (user-entered reason text)
  - `status = "gemeld"`
  - `created_by` (current user)
  - `created_at` (current timestamp)
- The system generates a **Peppol UBL ApplicationResponse** with:
  - Root element: `ApplicationResponse` (UBL 2.1)
  - Response code: "RE" (Rejected) or "AP" (Accepted with conditions, if partial)
  - Reason code: mapped from `dispute_reason` (e.g., "3" for duplicate, "ZZZ" for other)
  - Response message: populated with `dispute_message`
  - Response timestamp
  - Reference to the original invoice header
- The system sends the ApplicationResponse via the Peppol Access Point (openconnector peppol-e-invoicing-adapter):
  - Routes to the sender's Peppol endpoint
  - Captures the AS4 Message ID in `FactuurDispuut.peppol_response_message_id`
  - Logs the transmission in the audit trail
- The system sets `FactuurHeader.status = "in-dispuut"`
- The system removes the invoice from the payment queue (prevents any shillinq submission)
- The system creates a dashboard alert visible to krediteuren-administrateur: "Dispute on [invoice] sent to [vendor]; awaiting response"
- **Resolution path 1 (Credit Note):** IF the vendor sends a credit note UBL that references this dispute:
  - The system matches the credit note to the dispute
  - Once the credit note is processed, sets `FactuurDispuut.status = "afgehandeld"` and `FactuurHeader.status = "credit-note-received"`
- **Resolution path 2 (Correction Invoice):** IF the vendor sends a corrected invoice:
  - The system processes the new invoice as normal (REQ-001 through REQ-006)
  - Sets `FactuurDispuut.status = "afgehandeld"` and links the new invoice to the original dispute

**Non-functional:**
- ApplicationResponse generation must complete within 2 seconds
- AS4 transmission must complete within 30 seconds
- Message ID capture must be logged for audit trail

---

### REQ-008-010: Automatic Duplicate Detection

**Acceptance Criteria:**

GIVEN a new incoming invoice  
WHEN the system processes the FactuurHeader  
THEN:
- The system queries the `FactuurHeader` table for any existing records with the same:
  - `vendor_id` (matched vendor)
  - `invoice_number_vendor` (BT-1, vendor's own invoice number)
  - `invoice_date` (BT-2)
- IF a matching triple is found:
  - An alert is created and assigned to the krediteuren-administrateur
  - The new invoice is flagged with `status = "duplicate-suspected"`
  - The new invoice is removed from the automatic matching queue
  - A comparison view is displayed: original invoice ↔ suspected duplicate, side-by-side totals and line counts
  - The krediteuren-administrateur can click "Confirm duplicate" → creates a dispute (REQ-007), or "Different invoice" → proceeds to normal matching
- IF no duplicate is found: processing continues normally

**Non-functional:**
- Duplicate check must complete within 1 second
- Alert must appear in krediteuren-administrateur's queue within 10 seconds

---

### REQ-009-010: Archival for Compliance (7-Year Retention)

**Acceptance Criteria:**

GIVEN an invoice that is either:
- Successfully validated (REQ-001), OR
- Submitted to payment (REQ-006), OR
- Disputed (REQ-007)

WHEN the invoice reaches any terminal state, the archival service is triggered  
THEN:
- The system retrieves the original UBL document from `InboundPeppolFactuur.ubl_document_raw`
- The system calculates a **SHA-256 hash** of the UBL content and stores it in `ArchiefRecord.hash_checksum_sha256`
- The system extracts the **Peppol signature chain** (AS4 signature metadata, timestamps, signer certificates) and stores metadata in `ArchiefRecord.authenticity_proof`
- The system generates a **PDF/A-3 document** containing:
  - A human-readable rendering of the invoice (formatted as a standard invoice view)
  - The original UBL XML embedded as an attachment (for machine-readability)
- The system stores all three artifacts (UBL, PDF/A-3, metadata) in **docudesk** via an integration call:
  - `docudesk.store_invoice_archive({ubl, pdfa3, metadata, retention_days: 2920})`
- The system creates an `ArchiefRecord` with:
  - `archival_location` (docudesk URI returned from storage)
  - `hash_checksum_sha256` (calculated hash)
  - `authenticity_proof` (signature chain metadata)
  - `readability_version_pdfa3` (reference or binary blob, depending on implementation)
  - `retention_deadline` = `invoice_date + 7 years + 1 month` (adds 1 month buffer to ensure 7-year minimum)
  - `created_at` (timestamp)
- **Audit & Retrieval:** When an auditor requests the archived invoice:
  - The system retrieves the `ArchiefRecord` by `factuur_header_id`
  - Provides download links for: original UBL, PDF/A-3, signature chain metadata
  - Displays calculated hash for integrity verification
  - Shows retention deadline and destruction schedule
- **Destruction Schedule:** After `retention_deadline`, docudesk automatically schedules destruction; purchaseq monitors via event stream:
  - If destruction is blocked (regulatory hold), purchaseq extends `retention_deadline`
  - Once destruction occurs, `ArchiefRecord.destroyed_at` is updated

**Non-functional:**
- Archival must complete within 10 seconds of invoice reaching terminal state
- PDF/A-3 generation must produce valid PDF/A-3 format (verifiable by external tools)
- Hash calculation must use SHA-256 algorithm (no alternatives)
- Retention deadline must be immutable once set
- Retrieval of archived invoice must complete within 3 seconds

---

### REQ-010-010: Dashboard for STP Rate & Monitoring

**Acceptance Criteria:**

GIVEN a Controller or Financieel Beheerder who opens the invoice dashboard  
WHEN the dashboard loads  
THEN the system displays:

**Section 1: STP Rate & Cycle Time KPIs**
- **STP Rate (%):** Percentage of invoices that auto-passed all 3-way matching without manual approval
  - Configurable time window: last 7/30/90/365 days
  - Formula: `(count where all_lines_auto_akkoord) / (count total_invoices) * 100`
  - Trend line showing 30-day rolling average
- **Average Cycle Time (days):** From invoice receipt to payment submission
  - Formula: `avg(submitted_to_payment_timestamp - received_at_timestamp)` (in days)
  - Trend line
- **Total Invoice Volume & Value** (configurable period):
  - Count of invoices processed
  - Total value (sum of all invoices incl VAT)

**Section 2: Invoice Distribution by Status**
- Pie chart or stacked bar chart:
  - Auto-akkoord: count & %
  - Awaiting approval: count & %
  - In-dispuut: count & %
  - Submitted to payment: count & %

**Section 3: Exception Analysis**
- Top 10 vendors ranked by exception rate (count of approval_tasks / count of invoices from vendor)
- Table columns: Vendor name | Invoice count | Exception count | Exception rate (%)
- Click vendor → drills into their invoices and reasons for exceptions

**Section 4: Open Approval Tasks**
- Table: User | # Open tasks | Oldest deadline | Overdue count
- Overdue tasks highlighted in red (deadline < today)
- Click user → opens their open approval tasks in a list view

**Section 5: STP Trend (optional, for data-driven orgs)**
- Line chart: STP rate over the selected period
- Shows correlation with vendor count, invoice volume, tolerance adjustments (if any)

**Non-functional:**
- Dashboard must load within 5 seconds
- Charts must support 12-month data without performance degradation
- Drill-down from any chart element must complete within 2 seconds

---

## Non-Functional Requirements

---

### NFR-001: API Contract (Event-Driven)

The system communicates via internal event streams with the following event types:

**Inbound events (subscribed by this spec):**
- `InboundPeppolMessage` (from openconnector peppol-e-invoicing-adapter)
  - Fields: `peppol_message_id`, `sender_peppol_id`, `receiver_peppol_id`, `ubl_payload_xml`, `received_at`, `message_metadata`

**Outbound events (published by this spec):**
- `FactuurValidated` (schema validation passed)
- `VendorMatched` (vendor found or proposed)
- `POMatched` (PO found and validated)
- `GRMatched` (GR lookup complete)
- `LineMatched` (3-way match complete for a line)
- `FactuurReadyForPayment` (all lines approved, locked for payment)
- `FactuurSubmittedToPayment` (payment DTO sent to shillinq)
- `FactuurArchived` (archival complete)
- `FactuurDisputed` (dispute created)
- `FactuurDuplicateSuspected` (duplicate flagged)

**Example: FactuurReadyForPayment event**
```json
{
  "event_type": "FactuurReadyForPayment",
  "event_id": "uuid",
  "timestamp": "2026-05-15T14:33:00Z",
  "factuur_header_id": "uuid",
  "payment_data": {
    "vendor_id": "uuid",
    "vendor_name": "IT Services B.V.",
    "vendor_iban": "NL91ABNA0417164300",
    "invoice_amount_incl_vat": 1815.00,
    "currency": "EUR",
    "due_date": "2026-06-10",
    "payment_reference": "F-2026-004521",
    "early_payment_discount_percent": 2.0,
    "early_payment_deadline": "2026-05-25"
  },
  "audit_trail": {
    "po_reference": "PO-2026-001234",
    "all_lines_approved": true,
    "matched_at": "2026-05-15T14:32:30Z"
  }
}
```

---

### NFR-002: Concurrency & Async Processing

- All multi-second operations (validation, matching, archival) must run asynchronously
- Concurrent invoice processing: support at least 100 invoices matched in parallel
- Async job queue must track state and allow retry on transient failure
- Dead-letter queue for permanently failed invoices (escalated to decidesk)

---

### NFR-003: Data Immutability & Audit Trail

- `InboundPeppolFactuur.ubl_document_raw` is immutable (no updates after creation)
- `FactuurHeader`, `FactuurRegel`, `MatchingPoging` records are immutable after creation
- Modifications (PO updates, approvals, disputes) create new records, never overwrite existing ones
- Every state change is logged with `(user_id, timestamp, old_value, new_value)` in an audit log
- Audit log must support 7-year retention and querying

---

### NFR-004: Error Handling & Escalation

- Transient errors (network timeouts, temporary service unavailable): automatic retry with exponential backoff (3 retries max, 1s → 5s → 30s)
- Permanent errors (invalid PO reference, schema validation failure): log error, escalate to appropriate human (krediteuren-administrateur or decidesk)
- Unhandled exceptions: log stack trace, create incident alert for ops team

---

### NFR-005: Localization (Dutch)

- All UI text in Dutch (NL)
- Date format: DD-MM-YYYY
- Number format: 1.234,56 (European)
- Currency: EUR (with symbol €)
- All error messages and validation errors in Dutch

---

### NFR-006: Security & Authorization

- Invoice data is sensitive; access restricted by role:
  - `role:inkoper` can view & approve invoices for their cost centers
  - `role:crediteuren-admin` can view all invoices, manage vendors, configure tolerances
  - `role:controller` can view dashboard and export KPIs (read-only)
  - `role:auditor` can view archived invoices and audit trails (read-only)
- All API calls must be authenticated (OAuth2 or equivalent) and authorized (role-based)
- Audit trail must include user_id for all state changes
- PII (IBAN, VAT numbers) is masked in UI except for authorized roles

---

### NFR-007: Performance

- Invoice receipt to validation: <5 seconds
- 3-way match completion (100 lines): <50 seconds
- Dashboard load: <5 seconds
- API response time (p99): <2 seconds
- Concurrent user load: support 100+ concurrent dashboard users without degradation

---

### NFR-008: Reliability & Availability

- Service availability: 99.5% uptime (allow <4 hours downtime/month)
- Invoice processing: once an invoice is received, it must be processed to terminal state (approved, archived, or error escalated) within 7 days
- Event publishing must be idempotent (no duplicate payments if event is re-processed)
- Database backups: daily, with 30-day retention

---

### NFR-009: Compliance & Standards

- **Peppol BIS Billing 3.0:** strict schema validation, no deviations
- **EN 16931:** semantic validation for all mandatory/conditional fields
- **NLCIUS:** Dutch implementation rules from Forum Standaardisatie
- **Btw-richtlijn 2010/45/EU:** authenticity (signature chain) + integrity (hash) + readability (PDF/A-3)
- **Archiefwet:** 7-year retention with immutability proof
- **GDPR:** invoices contain vendor data; data retention policy must align with contract terms

---

### NFR-010: Integration Contracts

**Upstream dependency: openconnector peppol-e-invoicing-adapter**
- Must deliver Peppol AS4 messages as events
- Must accept Peppol ApplicationResponse messages from this spec
- Must provide SMP registration status and Peppol endpoint for outbound responses

**Downstream dependency: shillinq**
- Must subscribe to `FactuurReadyForPayment` events
- Must acknowledge receipt of payment instruction within 60 seconds
- Must NOT book invoice if returned error code (escalate to decidesk)

**Downstream dependency: purchaseq base**
- Must provide PO and GR APIs for matching queries
- Must support PO line price/qty updates from this spec

**Downstream dependency: docudesk**
- Must store and retrieve invoice archives with 7-year retention guarantee
- Must provide hash verification endpoints

---

## Use Cases & Scenarios

---

### Scenario 1: Happy Path (70%+ of invoices)

```
Timeline:
T0:   Peppol AS4 message arrives
T5s:  FactuurValidated event (schema OK, NLCIUS OK)
T8s:  VendorMatched event (Peppol ID match found)
T11s: POMatched event (BT-13 found, vendor verified)
T14s: GRMatched event (GR found, 3-way match prepared)
T18s: LineMatched event (all lines auto-akkoord, within tolerance)
T20s: FactuurReadyForPayment event published to shillinq
T22s: FactuurArchived event (UBL + hash + PDF/A-3 stored in docudesk)

Outcome: Invoice automatically submitted to payment, no human involvement.
Dashboard STP rate incremented by 1.
```

---

### Scenario 2: Price Mismatch (5% of invoices)

```
Timeline:
T0-T18s: Same as Scenario 1, but line matching detects 2.5% price difference
T18s:    LineMatched event (line status = gedeeltelijk, within tolerance)
         ApprovalTaak created, assigned to PO-eigenaar
         FactuurAwaitingApproval event published (not submitted to payment yet)

PO-eigenaar action:
T1d:     PO-eigenaar opens approval task
T1d5m:   Sees side-by-side: PO €1500 | Invoice €1537.50 (2.5% higher)
T1d10m:  Clicks "Accordeer" with reason "Leverancier bevestigd prijsaanpassung via e-mail"
T1d11m:  ApprovalTaak.status = geaccordeerd
         LineMatched event (now line status = geaccordeerd)
         FactuurReadyForPayment event published to shillinq
T1d12m:  FactuurArchived event (completed)

Outcome: Invoice submitted to payment after manual review. Cycle time ~1 day.
Dashboard shows 1 exception task resolved.
```

---

### Scenario 3: New Vendor (2-3% of invoices)

```
Timeline:
T0-T5s:  FactuurValidated event (schema OK)
T8s:     VendorMatched service runs, no match found on Peppol ID or KvK
         ApprovalTaak created with type=voorstel-nieuwe-leverancier
         Assigned to krediteuren-administrateur
         Status = "in-behandeling" (paused)

Krediteuren-administrateur action:
T1h:     Task appears in approval list
T1h5m:   Opens task, reviews extracted vendor data:
         - Name: "New IT Services Ltd"
         - KvK: (empty, not found)
         - VAT: "GB..." (UK company)
         - IBAN: "GB..."
T1h10m:  Clicks "Goedkeuren", vendor is added to Leveranciers master
         System re-triggers vendor matching (now succeeds, links vendor_id)
         Enqueues invoice for PO matching (REQ-003)
T1h15m:  PO matching, GR matching, line matching proceed as normal
T2h:     FactuurReadyForPayment event published

Outcome: Invoice submitted to payment after vendor approval. Cycle time ~2 hours.
```

---

### Scenario 4: Duplicate Detection (0.5% of invoices)

```
Timeline:
T0:      New invoice arrives from vendor ABC, invoice# F-2026-004521, date 2026-05-10
T3s:     FactuurValidated event
T6s:     VendorMatched event
T9s:     In duplicate check service: query for (vendor_id, invoice_number_vendor, invoice_date)
         Found: same invoice already processed on 2026-05-15
T12s:    DuplicateSuspected event
         New invoice status = "duplicate-suspected"
         Alert created for krediteuren-administrateur

Krediteuren-administrateur action:
T1d:     Opens alert, sees side-by-side comparison:
         Original (2026-05-15): 10 lines, €1815 total
         Suspected duplicate (2026-05-15): 10 lines, €1815 total
T1d5m:   Clicks "Confirm duplicate"
         ApprovalTaak created with type=dispute
T1d6m:   Enters reason: "Leverancier heeft twee keer dezelfde factuur verstuurd"
         FactuurDispuut created
         Peppol ApplicationResponse (rejected, reason=duplicate) generated
T1d7m:   ApplicationResponse sent to vendor via AP
         FactuurDispuut.status = "gemeld"

Outcome: Duplicate blocked, dispute sent to vendor. Original invoice processed, duplicate rejected.
```

---

### Scenario 5: Missing PO (3-5% of invoices)

```
Timeline:
T0-T8s:  FactuurValidated + VendorMatched events
T11s:    POMatched service runs, BT-13 is empty (no PO reference)
         Invoice moved to "no-PO" queue
         ApprovalTaak created with type=po-het-gevonden (or "no-PO classification")
         Assigned to krediteuren-administrateur
         Status = "in-behandeling" (paused)

Krediteuren-administrateur action (option A):
T2h:     Opens task, reviews invoice (vendor, amount, description)
T2h5m:   Decides it's a legitimate expense without PO (consulting, utilities, etc.)
T2h6m:   Clicks "Kostenfactuur zonder PO" classification
         Invoice continues to 2-way match (invoice ↔ GR only, no PO required)
         Enqueues for archival

Krediteuren-administrateur action (option B):
T2h:     Opens task
T2h5m:   Finds the matching PO in purchaseq (BT-13 was incomplete)
T2h6m:   Attaches PO via "PO toewijzen" form
         Re-triggers full 3-way match as normal

Outcome: Either processed as expense (fast-track), or matched to PO (normal path).
```

---

## Summary Table

| REQ | Title | GIVEN | WHEN | THEN | Actors |
|-----|-------|-------|------|------|--------|
| 001 | Validation | Peppol message arrives | Schema check runs | Valid → proceed; Invalid → MLR negative | peppol-receiver, AP |
| 002 | Vendor Matching | Validated invoice | Lookup vendor | Match → link; No match → proposal task | matcher, krediteuren-admin |
| 003 | PO Lookup | Vendor linked | BT-13 present | PO found → link; Not found → task | po-matcher |
| 004 | GR Lookup | PO linked | Query GR | GR found → optional match; Not found → 2-way | gr-matcher |
| 005 | 3-Way Match | Lines ready | Compare qty/price | Within tolerance → auto-akkoord or wacht; Out → task | line-matcher |
| 005b | Exception Workflow | Line mismatch | PO owner opens task | Show diff, 3 actions (approve / adjust PO / dispute) | po-owner |
| 006 | Payment Submission | All lines approved | Lock invoice | Publish FactuurReadyForPayment to shillinq | payment-submitter |
| 007 | Dispute | Invoice disputed | Generate response | Send ApplicationResponse, update status | dispute-handler, AP |
| 008 | Duplicate Detection | New invoice | Check (vendor, invnum, date) | Found → alert; Not found → proceed | duplicate-checker, krediteuren-admin |
| 009 | Archival | Invoice terminal | Store UBL + hash + PDF | docudesk stores, 7-year retention | archival-service |
| 010 | Dashboard | Controller opens | Load KPIs | Show STP rate, cycle time, exceptions | controller |

