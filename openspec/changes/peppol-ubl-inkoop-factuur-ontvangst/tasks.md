---
status: tasks
app: purchaseq
spec: peppol-ubl-inkoop-factuur-ontvangst
created: 2026-05-23
---

# Implementation Tasks: Peppol UBL Inkoopfactuur Ontvangst en Drie-weg-match

## Phase 1: Data Layer & Schema

### Database Schema

- [ ] Create `InboundPeppolFactuur` table with fields: id (UUID PK), peppol_message_id, received_at, sender_peppol_id, receiver_peppol_id, ubl_document_raw (XML), document_type (enum), validation_status (enum), validation_errors (jsonb), nlcius_compliant (bool), created_at, created_by
- [ ] Create `FactuurHeader` table with fields: id, inbound_peppol_factuur_id (FK), vendor_id (FK, nullable), vendor_peppol_id, vendor_name, vendor_btw_number, invoice_number_vendor, invoice_date, due_date, po_reference, currency_code, total_excl_vat, total_vat, total_incl_vat, payment_terms_text, payment_account_iban, discount_early_payment, matched_po_id (FK, nullable), status (enum), created_at
- [ ] Create `FactuurRegel` table with fields: id, factuur_header_id (FK), line_number, article_code, description, quantity, unit_code, unit_price, line_amount_excl_vat, vat_category_code, vat_percentage, vat_amount, po_line_reference, cost_center_suggestion, created_at
- [ ] Create `MatchingPoging` table with fields: id, factuur_regel_id (FK), matched_po_line_id (FK, nullable), matched_gr_line_id (FK, nullable), match_status (enum: volledig|gedeeltelijk|geen), price_difference_amount, price_difference_percentage, quantity_difference, within_tolerance (bool), conclusion (enum: auto-akkoord|wacht-op-accordering|auto-afkeur), tolerance_check_details (jsonb), created_at, created_by
- [ ] Create `ApprovalTaak` table with fields: id, factuur_regel_id (FK), matching_poging_id (FK), assigned_to (user_id FK), created_at, deadline, status (enum: open|geaccordeerd|afgekeurd|escalatie), action_taken (enum, nullable), action_reason (text, nullable), treated_at (timestamp, nullable), treated_by (user_id, nullable)
- [ ] Create `MatchingTolerantie` table with fields: id, organization_id (FK), cost_category (string, nullable), price_tolerance_percentage, price_tolerance_absolute, quantity_tolerance_percentage, quantity_tolerance_absolute, is_default (bool), created_at, created_by
- [ ] Create `FactuurDispuut` table with fields: id, factuur_header_id (FK), dispute_reason (enum), dispute_message (text), status (enum: gemeld|afgehandeld|escalatie), peppol_response_message_id, credit_note_received_id (FK, nullable), created_at, created_by, resolved_at, resolved_by
- [ ] Create `ArchiefRecord` table with fields: id, factuur_header_id (FK), archival_location (string), hash_checksum_sha256, authenticity_proof (jsonb), readability_version_pdfa3 (blob, nullable), retention_deadline (date), destruction_scheduled_at, destroyed_at, created_at
- [ ] Create audit log table: `AuditLog` with fields: id, table_name, record_id (UUID), action (enum: INSERT|UPDATE|DELETE), old_value (jsonb), new_value (jsonb), changed_by (user_id), changed_at (timestamp), reason (text, nullable)
- [ ] Add database indexes on:
  - `InboundPeppolFactuur(peppol_message_id)` (unique)
  - `FactuurHeader(vendor_id, invoice_number_vendor, invoice_date)` (for duplicate detection)
  - `FactuurHeader(matched_po_id)`
  - `FactuurHeader(status)`
  - `ApprovalTaak(assigned_to, status)`
  - `ApprovalTaak(deadline)`
  - `MatchingPoging(factuur_regel_id)`
- [ ] Document schema changes in migration file with description of each entity's purpose and role in the workflow

---

## Phase 2: Backend Services (Async)

### Validation Service

- [ ] Create `PeppolValidationService` class with method `validateInboundUBL(InboundPeppolFactuur)`
  - Parse UBL XML using standard XML library
  - Validate against Peppol BIS Billing 3.0 XSD schema (download from Openpeppol registry)
  - Validate against NLCIUS rules from Forum Standaardisatie (check mandatory fields per NL rules)
  - Extract core metadata: document_type, invoice_number, date, totals, vendor Peppol ID
  - Store extracted metadata in FactuurHeader
  - Create FactuurRegel records (one per UBL InvoiceLine)
  - If any validation error: populate validation_errors array, set validation_status=ongeldig, return error list
  - If all validations pass: set validation_status=valide, publish `FactuurValidated` event
  - Log validation result (success/failure) to audit trail
- [ ] Create unit tests for validation service covering: valid BIS 3.0 invoice, invalid schema, NLCIUS violations, edge cases (empty lines, large amounts, special characters)
- [ ] Integrate with UBL parser library (e.g., Zugferd, Peppol UBL). Document choice and fallback options.

### Vendor Matching Service

- [ ] Create `VendorMatchingService` class with method `matchVendor(FactuurHeader)`
  - Primary match: query Leverancier table by `sender_peppol_id` (exact match)
  - If not found, secondary match: extract KvK from UBL EndpointID field, query Leverancier by KvK
  - If match found: update FactuurHeader.vendor_id, publish `VendorMatched` event, return success
  - If no match: extract vendor data from UBL (name, VAT, IBAN, contact), create ApprovalTaak with type=voorstel-nieuwe-leverancier, assign to krediteuren-administrateur, publish `VendorProposalCreated` event, return pending
  - Publish event to event stream
- [ ] Implement approval task handler: when krediteuren-administrateur approves the proposed vendor, create Leverancier record and re-trigger vendor matching
- [ ] Implement rejection handler: generate Peppol MLR negative and send back via AP
- [ ] Add logging at each matching step for debugging
- [ ] Write integration tests with mock Leverancier data

### PO Matching Service

- [ ] Create `POMatchingService` class with method `matchPO(FactuurHeader)`
  - Extract BT-13 (po_reference) from FactuurHeader
  - If empty/null: move invoice to "no-PO" queue, create task for krediteuren-administrateur, publish `PONotFound` event, return no-match
  - Query purchaseq PO registry by po_reference (e.g., "PO-2026-001234")
  - If found: validate vendor on PO matches invoice vendor (FactuurHeader.vendor_id == PO.vendor_id)
    - If match: update FactuurHeader.matched_po_id, publish `POMatched` event, return success
    - If vendor mismatch: log error, escalate to decidesk, publish `POVendorMismatch` event, return error
  - If not found: create ApprovalTaak with type=po-niet-gevonden, assign to krediteuren-administrateur, publish event
- [ ] Implement manual PO attachment: krediteuren-administrateur can manually link a PO via task action
- [ ] Implement "Kostenfactuur zonder PO" classification: skip PO matching for invoices explicitly marked as expenses
- [ ] Call purchaseq base API (REST or event-based) to fetch PO data
- [ ] Document PO reference format expectations and edge cases (spaces, case sensitivity, special chars)

### GR (Goods Receipt) Matching Service

- [ ] Create `GRMatchingService` class with method `matchGR(FactuurHeader)`
  - Query purchaseq GR registry for GR records linked to FactuurHeader.matched_po_id
  - For each FactuurRegel, attempt to find matching GR line by po_line_reference or article_code
  - Update MatchingPoging.matched_gr_line_id for matched pairs
  - If no GR found for a line: leave matched_gr_line_id null, proceed with 2-way match (invoice ↔ PO)
  - If GR exists but receipt_date is in the future: log warning, treat as "not yet complete"
  - Publish `GRMatched` event with all GR line matches
  - Return matched pairs for downstream line-matching service
- [ ] Handle optional GR (not all invoices have GR, or GR arrives late)
- [ ] Call purchaseq base API to fetch GR data
- [ ] Log GR matching details for audit trail

### Line-Level Matching Service

- [ ] Create `LineMatchingService` class with method `matchLines(FactuurHeader, List<FactuurRegel>, List<MatchingPoging>)`
  - For each FactuurRegel:
    - Find matching PO line (by line_number or article_code)
    - If no PO line: create MatchingPoging with match_status=geen, conclusion=wacht-op-accordering
    - If PO line found:
      - Compare qty: invoice_qty vs po_qty, calculate difference
      - Compare unit_price: invoice_unit_price vs po_unit_price, calculate difference (absolute & %)
      - Load MatchingTolerantie rules (org-wide or by cost_category)
      - Check: ABS(price_diff_pct) <= tolerance_pct AND ABS(qty_diff) <= qty_tolerance
      - Determine match_status: volledig (no diffs) | gedeeltelijk (diff but within tolerance) | geen (no PO)
      - Determine conclusion:
        - If match_status=volledig AND within_tolerance: conclusion=auto-akkoord
        - If match_status=gedeeltelijk AND within_tolerance: conclusion=wacht-op-accordering
        - Else: conclusion=wacht-op-accordering
    - Create MatchingPoging record with all calculations
    - Store tolerance_check_details as JSON for audit
  - Check overall invoice status:
    - If all lines have conclusion=auto-akkoord: invoice proceeds to "ready-for-payment"
    - If any line has conclusion=wacht-op-accordering: create ApprovalTaak for each mismatched line
  - Publish `LineMatched` or `AllLinesMatched` event
- [ ] Write comprehensive unit tests:
  - Exact match (0% diff, 0 qty diff)
  - Price within 2% tolerance
  - Qty within 3 units tolerance
  - Mix of matches and mismatches in same invoice
  - Different cost categories with different tolerance rules
- [ ] Ensure tolerance logic is transparent (log why a line passed/failed)

### Approval Task Management Service

- [ ] Create `ApprovalTaskService` class to manage task lifecycle:
  - `createTask(factuur_regel_id, matching_poging_id, assigned_to_user_id)` → create ApprovalTaak with deadline = today + 5 business days
  - `approveTask(task_id, reason_text)` → update status=geaccordeerd, action_taken=accordeer, record user & timestamp
  - `updatePOandRethink(task_id, new_price, new_qty)` → update PO line in purchaseq base, re-trigger line matching for affected invoice
  - `disputeTask(task_id, reason_enum, message_text)` → create FactuurDispuut, trigger Peppol response generation
  - `escalateTask(task_id, reason_text)` → mark status=escalatie, send alert to decidesk
- [ ] Notify assigned user when task is created (email + in-app notification)
- [ ] Implement deadline monitoring: mark as red if deadline < today
- [ ] Log all task actions to audit trail
- [ ] Handle task reassignment (PO owner changes role or leaves)

### Duplicate Detection Service

- [ ] Create `DuplicateDetectionService` class with method `checkDuplicate(FactuurHeader)`
  - Query FactuurHeader table for existing records matching (vendor_id, invoice_number_vendor, invoice_date)
  - If match found: create alert, set invoice status=duplicate-suspected, publish `DuplicateSuspected` event
  - Krediteuren-administrateur reviews and confirms duplicate (creates dispute) or marks as "Different invoice" (proceeds normally)
- [ ] Create dashboard view for duplicate alerts: side-by-side comparison of original ↔ suspected duplicate
- [ ] Write tests with mock data: duplicate case, similar-but-different case

### Payment Submission Service

- [ ] Create `PaymentSubmissionService` class with method `submitToPayment(FactuurHeader)`
  - Verify all FactuurRegel lines have conclusion=auto-akkoord or geaccordeerd (no pending approval tasks)
  - Lock FactuurHeader for writes (set status=locked-for-payment)
  - Extract & validate payment data:
    - vendor_iban from FactuurHeader.payment_account_iban
    - total_incl_vat from FactuurHeader
    - due_date from FactuurHeader
    - early_payment_discount_percent if present
    - payment_reference = invoice_number_vendor for reconciliation
  - If any required field missing/invalid: escalate to decidesk, return error
  - Create InvoiceReadyForPayment DTO with audit_trail metadata (PO reference, matched_at, all_lines_approved)
  - Publish InvoiceReadyForPayment event to shillinq event stream
  - Set FactuurHeader.status=submitted-to-payment
  - Publish `FactuurSubmittedToPayment` event for archival service
  - Log submission to audit trail
- [ ] Implement idempotent event publishing (no duplicate payments if event is re-published)
- [ ] Handle shillinq acknowledgment timeout (>60 sec = escalate to decidesk)

### Dispute Handling Service

- [ ] Create `DisputeService` class with method `createDispute(FactuurHeader, reason_enum, message_text)`
  - Create FactuurDispuut record with reason, message, status=gemeld
  - Lock invoice (remove from payment queue)
  - Call PeppolResponseGenerator to create UBL ApplicationResponse
  - Send ApplicationResponse via openconnector peppol-e-invoicing-adapter
  - Capture returned message_id in FactuurDispuut.peppol_response_message_id
  - Publish `FactuurDisputed` event
  - Create dashboard alert for krediteuren-administrateur: "Dispute sent to [vendor]; awaiting response"
- [ ] Implement dispute resolution paths:
  - Credit note from vendor: match to dispute by references, auto-close dispute
  - Correction invoice: process as new invoice, link to original dispute, close dispute
  - Timeout (>30 days): escalate to decidesk for manual follow-up
- [ ] Log all dispute communications to audit trail

### Peppol Response Generator

- [ ] Create `PeppolResponseGenerator` class with method `generateApplicationResponse(FactuurHeader, reason_code, message_text)`
  - Create valid UBL 2.1 ApplicationResponse XML:
    - Root element: `<ApplicationResponse xmlns="...UBL 2.1...">`
    - Response code: "RE" (Rejected) or "AP" (Accepted with conditions)
    - Reason code: mapped from dispute_reason (e.g., "3" for duplicate)
    - Response message with user text
    - References to original invoice ID, date, amount
    - Timestamp
  - Validate generated XML against UBL schema
  - Return serialized XML for transport
- [ ] Document reason code mapping (duplicate → 3, qty_mismatch → 5, etc.)
- [ ] Write tests for valid/invalid responses

### Archival Service

- [ ] Create `ArchivalService` class with method `archiveInvoice(InboundPeppolFactuur, FactuurHeader, ArchiefRecord)`
  - Retrieve original UBL from InboundPeppolFactuur.ubl_document_raw
  - Calculate SHA-256 hash of UBL bytes
  - Extract Peppol signature chain metadata (from AS4 message envelope)
  - Generate PDF/A-3 rendering:
    - Human-readable invoice layout (vendor, date, lines, totals, payment info)
    - UBL XML embedded as attachment
    - File format must pass PDF/A-3 validation
  - Call docudesk API to store all artifacts:
    - UBL XML
    - PDF/A-3 file
    - Signature chain metadata
    - Hash checksum
  - Receive back archival_location URI from docudesk
  - Create ArchiefRecord with all metadata
  - Set retention_deadline = invoice_date + 7 years + 1 month
  - Publish `FactuurArchived` event
  - Log archival completion to audit trail
- [ ] Implement archival retrieval: query ArchiefRecord by factuur_header_id, return download URIs for UBL + PDF + metadata
- [ ] Implement hash verification: allow auditor to verify downloaded UBL against stored hash
- [ ] Monitor docudesk destruction events: update destroyed_at when archival expires and is destroyed
- [ ] Document PDF/A-3 generation tool/library used (e.g., iText, Apache PDFBox)

---

## Phase 3: Backend APIs & Event Handling

### Event Subscriptions & Handlers

- [ ] Subscribe to `InboundPeppolMessage` event from openconnector peppol-e-invoicing-adapter
  - Handler: create InboundPeppolFactuur record, immediately trigger validation service
- [ ] Subscribe to `VendorApprovalDecision` event (krediteuren-administrateur approves/rejects proposed vendor)
  - Handler: on approval, create Leverancier, re-trigger vendor matching; on rejection, generate MLR negative
- [ ] Subscribe to `POApprovalDecision` event (krediteuren-administrateur finds/attaches PO)
  - Handler: on success, update FactuurHeader.matched_po_id, re-trigger PO matching; on failure, log escalation
- [ ] Subscribe to `ApprovalTaskDecision` event (PO owner takes action: approve / adjust PO / dispute)
  - Handlers:
    - On "approve": update ApprovalTaak.status=geaccordeerd, check if all lines now approved, if so publish FactuurReadyForPayment
    - On "adjust PO": call PO update API, re-trigger line matching
    - On "dispute": trigger dispute service

### REST APIs for Frontend

- [ ] `GET /api/invoices` (list invoices with filters)
  - Query params: status, date_range, vendor_id, sort_by
  - Returns: paginated list of FactuurHeader with summary info (vendor, date, total, status, action_count)
  - Auth: require role:inkoper or role:krediteuren-admin
  - Response time: <2 sec
- [ ] `GET /api/invoices/{id}` (invoice detail)
  - Returns: full FactuurHeader + FactuurRegel lines + MatchingPoging results + ArchiefRecord link
  - Shows side-by-side comparison if pending approval
  - Auth: require role:inkoper (own cost center) or role:krediteuren-admin (all)
- [ ] `GET /api/invoices/{id}/approval-tasks` (approval tasks for this invoice)
  - Returns: list of ApprovalTaak with assigned user, deadline, action options
  - Auth: require role:inkoper or role:krediteuren-admin
- [ ] `POST /api/approval-tasks/{id}/approve` (PO owner approves)
  - Body: { reason_text }
  - Updates ApprovalTaak.status=geaccordeerd, publishes event
  - Returns: success status + next action (if all lines now approved, shows "voorgezet naar betaling")
  - Auth: require role:inkoper (assigned task) or role:krediteuren-admin
- [ ] `POST /api/approval-tasks/{id}/adjust-po` (PO owner adjusts PO)
  - Body: { new_price, new_qty }
  - Calls purchaseq base API to update PO line
  - Re-triggers line matching
  - Returns: re-match result (all lines now pass? or still mismatches?)
  - Auth: require role:inkoper (PO owner)
- [ ] `POST /api/approval-tasks/{id}/dispute` (PO owner / krediteuren-admin disputes)
  - Body: { reason: enum, message_text }
  - Creates FactuurDispuut, generates Peppol response
  - Returns: dispute_id + message_id sent to vendor
  - Auth: require role:inkoper or role:krediteuren-admin
- [ ] `GET /api/invoices/{id}/archive` (auditor retrieves archived invoice)
  - Returns: links to download UBL + PDF/A-3 + signature metadata + hash
  - Auth: require role:auditor or role:krediteuren-admin
- [ ] `GET /api/invoices/{id}/audit-log` (view all state changes)
  - Returns: audit log entries for this invoice (created, validated, matched, approved, submitted, archived)
  - Auth: require role:auditor or role:krediteuren-admin
- [ ] `GET /api/dashboard` (controller dashboard KPIs)
  - Returns:
    - STP rate (%) for selected period
    - Avg cycle time (days)
    - Invoice volume & value
    - Status distribution (chart data)
    - Top vendors by exception rate
    - Open approval tasks by user
  - Auth: require role:controller or role:crediteuren-admin
- [ ] `GET /api/config/tolerances` (view tolerance configuration)
  - Returns: list of MatchingTolerantie records for organization
  - Auth: require role:krediteuren-admin
- [ ] `POST /api/config/tolerances` (update tolerances)
  - Body: { cost_category, price_tolerance_pct, price_tolerance_abs, qty_tolerance_pct, qty_tolerance_abs }
  - Updates or creates MatchingTolerantie record
  - Returns: success + affected invoice count (how many pending matches will be re-evaluated)
  - Auth: require role:krediteuren-admin
- [ ] Rate limiting: implement on all endpoints (100 req/min per user)
- [ ] Input validation: validate all request parameters, return 400 for invalid input
- [ ] Error responses: return 500 with error_code and error_message for all errors
- [ ] API documentation: generate OpenAPI/Swagger spec from code

### Authentication & Authorization

- [ ] Implement role-based access control (RBAC) checks in middleware
  - Roles: inkoper, krediteuren-admin, controller, auditor, admin
  - Check user role before allowing action
  - Scope access to cost centers (inkoper can only see own cost center invoices)
- [ ] Implement audit logging: log all API calls with user_id, action, resource, timestamp
- [ ] Implement PII masking: mask IBAN, VAT numbers in API responses for non-admin roles

---

## Phase 4: Frontend (UI Pages & Components)

### Invoice List Page (`/invoices`)

- [ ] Create page component in Vue 3 (or React, as per app conventions)
- [ ] Implement quick stats widgets:
  - STP rate (%) last 30 days
  - Avg cycle time (days)
  - Open approval tasks count
- [ ] Implement filters:
  - Status dropdown: All | In Ontvangst | Matching | Awaiting Approval | Approved | Disputed
  - Date range picker (last 7/30/90 days or custom)
  - Vendor search/autocomplete
  - Sort: date desc | amount desc | status
- [ ] Implement invoice table:
  - Columns: invoice_date | vendor_name | invoice_number | total_incl_vat | status [badge] | actions
  - Status badges (styled): 🟢 auto-akkoord | 🟡 awaiting-approval | 🔴 disputed
  - Row click → navigate to detail page
  - Inline actions: View | Print | Download PDF
- [ ] Implement pagination: 25/50/100 rows per page
- [ ] Implement export: CSV/Excel with invoice data
- [ ] Performance: load table within 2 seconds, lazy-load details on row click
- [ ] Mobile responsive: show key columns (date, vendor, amount, status) on small screens
- [ ] Accessibility: semantic HTML, ARIA labels, keyboard navigation

### Invoice Detail Page (`/invoices/{id}`)

- [ ] Create detail page component
- [ ] Header section:
  - Vendor name & Peppol ID
  - Invoice number (vendor) | Date | Due date | Currency
  - Status badge + alerts (if duplicate/disputed)
  - Action buttons (Dispute | Print | Download PDF | Archive)
- [ ] Financial summary:
  - Subtotal excl VAT | VAT amount | Total incl VAT (breakdown by VAT category)
  - Payment info: IBAN (masked) | Due date | Early payment discount
- [ ] Matching summary card:
  - PO reference [link to PO] | GR link [if matched] | Match status (volledig/gedeeltelijk)
- [ ] Line-by-line detail table (expandable rows):
  - Columns: Line # | Description | Qty | Unit Price | Line Amount | Match Status | [expand]
  - On expand row:
    - Show side-by-side: PO line | GR line (if exists) | Invoice line
    - Show MatchingPoging results: price_diff, qty_diff, tolerance_check
    - Show action buttons (Approve | Adjust PO | Dispute) if awaiting approval
- [ ] Tab navigation (if needed):
  - Details | Matching Results | Audit Log | Archive
- [ ] Responsive design for mobile (stack columns, horizontal scroll)

### Approval Task Modal / Page

- [ ] Create reusable approval task component (can be modal or dedicated page)
- [ ] Header:
  - Task ID | Assigned to: [name] | Deadline: [date] [red if overdue]
  - Invoice: [vendor] [date] [amount]
- [ ] Three-column side-by-side comparison:
  - Column 1 (PO): line #, qty, unit_price, line_amount, cost_center
  - Column 2 (GR, if exists): receipt_date, received_qty
  - Column 3 (Invoice): line #, qty, unit_price, line_amount, VAT%
  - Highlight differences in red (price %, qty delta)
  - Show tolerance thresholds (e.g., "2% ± €25")
- [ ] Three action buttons:
  - "Accordeer" → text area for reason, save button
  - "PO bijwerken" → inline edit form for PO price/qty, save button
  - "Betwisten" → dropdown for reason (duplicate|qty_mismatch|invalid|other), text area, save button
- [ ] Confirmation modal before action (prevent accidental clicks)
- [ ] Loading state during submission (disable buttons, show spinner)
- [ ] Success/error toast after action completes
- [ ] Auto-close modal on success, show next pending task (if any)

### Dashboard Page (`/invoices/dashboard`)

- [ ] KPI card widgets (4-5 top cards):
  - STP Rate (%) [with trend indicator ↑↓]
  - Avg Cycle Time (days)
  - Invoice Volume (count) & Value (EUR)
  - Exception Rate (%)
- [ ] Period selector: Last 7/30/90/365 days (default 30)
- [ ] Chart 1: STP Trend (line chart, 30-day rolling average)
  - X-axis: date (daily)
  - Y-axis: STP % (0-100)
  - Hover tooltip shows STP % + volume for that day
- [ ] Chart 2: Invoice Status Distribution (pie chart or stacked bar)
  - Categories: auto-akkoord | awaiting-approval | disputed | submitted
  - Show count & % for each
  - Click category → filters main invoice list
- [ ] Chart 3: Top 10 Vendors by Exception Rate (horizontal bar chart)
  - X-axis: exception rate (%)
  - Y-axis: vendor name (sorted by exception rate descending)
  - Bar color: gradient (green for low, red for high)
  - Click vendor → filters main invoice list by vendor
- [ ] Table: Open Approval Tasks
  - Columns: User | # Open tasks | Oldest deadline | # Overdue
  - Overdue tasks highlighted in red
  - Click user → filters approval tasks by user
- [ ] All charts interactive: click to drill-down, filter invoice list
- [ ] Performance: dashboard loads within 5 seconds, charts render within 3 seconds
- [ ] Export: download dashboard data as CSV/PDF report

### Vendor Proposal Modal

- [ ] Modal appears when krediteuren-administrateur reviews new vendor proposal
- [ ] Display extracted vendor data (from UBL):
  - Name
  - VAT/BTW number (if present)
  - Peppol ID
  - IBAN (masked)
  - Contact email
- [ ] Action buttons:
  - "Goedkeuren" → vendor added to Leveranciers, invoice re-queued for matching
  - "Weigeren" → Peppol MLR negative sent, invoice marked invalid
- [ ] Confirmation message before action

### Duplicate Alert Modal

- [ ] Modal shows when duplicate is detected
- [ ] Side-by-side comparison:
  - Original invoice (left): date, vendor, lines, total, status
  - Suspected duplicate (right): same fields
- [ ] Highlight differences (if any)
- [ ] Action buttons:
  - "Bevestig dubbel" → creates dispute, sends Peppol response
  - "Ander factuur" → proceeds to normal matching, closes alert

---

## Phase 5: Integrations & Event Stream

### openconnector peppol-e-invoicing-adapter Integration

- [ ] Verify adapter can publish `InboundPeppolMessage` events (or implement if missing)
  - Event fields: peppol_message_id, sender_peppol_id, receiver_peppol_id, ubl_payload_xml, received_at, message_metadata
- [ ] Subscribe to events (implement event listener in async job queue)
- [ ] Consume Peppol AS4 message, create InboundPeppolFactuur record
- [ ] Send Peppol MLR responses:
  - Call adapter API: `POST /peppol/send-response` with { recipient_peppol_id, message_id, response_type, reason_code, message_text }
  - Adapter routes response to vendor's Peppol endpoint
- [ ] Handle AS4 transport errors (timeout, invalid endpoint)
- [ ] Log all messages received/sent for audit trail

### shillinq Integration

- [ ] Publish `InvoiceReadyForPayment` event to shared event stream
  - Event format: { factuur_header_id, vendor_id, vendor_name, vendor_iban, total_incl_vat, currency, due_date, payment_reference, early_payment_discount, audit_trail }
  - Publish timing: once invoice transitions to "submitted-to-payment" status
- [ ] Subscribe to `PaymentScheduled` event (acknowledgment from shillinq)
  - Handler: log acknowledgment in audit trail, do not re-publish same event
- [ ] Subscribe to `PaymentFailed` event (if shillinq cannot process)
  - Handler: escalate to decidesk with error details, unlock invoice (allow re-submission)
- [ ] Define event schema/contract with shillinq team

### purchaseq base Integration

- [ ] Call purchaseq PO API to fetch PO data:
  - `GET /api/po/{po_id}` → return PO header + lines
  - `GET /api/po/search?reference={reference}` → search PO by reference string
  - `PUT /api/po/{po_id}/lines/{line_id}` → update PO line price/qty
- [ ] Call purchaseq GR API to fetch GR data:
  - `GET /api/gr?po_id={po_id}` → return GR records for PO
  - `GET /api/gr/{gr_id}/lines` → return GR line details
- [ ] Handle API errors:
  - PO not found: return 404, trigger manual matching task
  - PO API timeout: retry with exponential backoff, escalate if persistent
- [ ] Document API contracts with purchaseq team

### docudesk Integration

- [ ] Call docudesk API to store invoice archive:
  - `POST /api/archives/store` with multipart form: { ubl_xml, pdfa3_file, metadata_json }
  - Returns: { archival_location, storage_timestamp }
- [ ] Call docudesk API to retrieve archive:
  - `GET /api/archives/{archival_location}` → download original UBL
  - `GET /api/archives/{archival_location}/pdfa3` → download PDF/A-3
- [ ] Monitor docudesk destruction events:
  - Subscribe to `ArchiveDestroyed` event → update ArchiefRecord.destroyed_at
- [ ] Document API contracts with docudesk team

### decidesk Integration (Escalations)

- [ ] Publish escalation events for:
  - `POVendorMismatch` → escalate with invoice details + PO + vendor info
  - `PaymentDataIncomplete` → escalate with missing fields
  - `DisputeTimeout` (>30 days no response) → escalate with dispute details
  - `HighValueDispute` (amount > EUR 10,000) → escalate automatically
- [ ] Define escalation event schema with decidesk team
- [ ] Log all escalations to audit trail

### openconnector kvk-adapter Integration

- [ ] Call kvk-adapter API to verify new suppliers:
  - `POST /api/kvk/verify` with { kvk_number, company_name }
  - Returns: { valid, registration_data, warnings }
- [ ] Store verification result in Leverancier record (for audit)
- [ ] Handle unverified suppliers: allow krediteuren-administrateur to override (with logged reason)

---

## Phase 6: Configuration & Administration

### Tolerance Configuration UI

- [ ] Create admin page at `/admin/invoice-config/tolerances`
- [ ] Display current tolerance rules (org-wide default + per-category overrides)
- [ ] Form to add/edit rule:
  - Cost category (optional, or "default")
  - Price tolerance %
  - Price tolerance absolute (EUR)
  - Qty tolerance %
  - Qty tolerance absolute (units)
- [ ] Show estimated impact: "Changing price tolerance to 3% will affect X pending invoices (list them)"
- [ ] Save button → create new MatchingTolerantie record, re-evaluate pending matches
- [ ] Audit trail: log all tolerance configuration changes

### Vendor Proposal Review Queue

- [ ] Create admin page at `/admin/vendor-proposals`
- [ ] List of proposed new vendors (from ApprovalTaak type=voorstel-nieuwe-leverancier)
- [ ] Display: vendor name, Peppol ID, first invoice from vendor, status
- [ ] Click vendor → modal shows:
  - Extracted data from UBL (name, VAT, IBAN, contact)
  - First invoice referencing this vendor (for context)
  - KVK verification status (if attempted)
- [ ] Action buttons: "Approve" | "Reject with reason"
- [ ] Audit log: who approved/rejected and when

### Manual PO Attachment UI

- [ ] Create admin page at `/admin/manual-tasks`
- [ ] List: invoices waiting for manual PO attachment (ApprovalTaak type=po-niet-gevonden)
- [ ] For each: show invoice + suggested PO references (fuzzy search based on amount, date, vendor)
- [ ] Action:
  - Select a PO from suggestions OR search manually
  - Attach PO → re-trigger matching
  - Mark as "Kostenfactuur" → skip PO matching, proceed to archival
- [ ] Audit log: who attached PO and when

---

## Phase 7: Testing & QA

### Unit Tests

- [ ] Validation service tests: valid/invalid BIS 3.0, NLCIUS rules
- [ ] Vendor matching tests: Peppol ID match, KvK match, no match
- [ ] PO matching tests: PO found, PO not found, vendor mismatch
- [ ] Line matching tests: exact match, price within tolerance, qty out of tolerance, no PO line
- [ ] Tolerance logic tests: various tolerance combinations, cost-category overrides
- [ ] Approval task lifecycle tests: create → approve/adjust/dispute → close
- [ ] Duplicate detection tests: exact duplicate, false positives
- [ ] Peppol response generation tests: valid XML, all reason codes
- [ ] PDF/A-3 generation tests: valid PDF/A-3 format, UBL embedded, readable

### Integration Tests

- [ ] End-to-end workflow: Peppol message → validated → matched → approved → submitted to shillinq → archived
- [ ] Vendor proposal flow: new vendor → proposal task → approval → re-queued → matched
- [ ] Exception handling: PO not found → manual task → attach PO → re-match
- [ ] Duplicate detection: duplicate blocked → dispute → sent to vendor
- [ ] Dispute resolution: credit note received → matched to dispute → closed
- [ ] Integration with purchaseq base: PO lookup, GR lookup, PO update
- [ ] Integration with shillinq: payment submission, event acknowledgment
- [ ] Integration with docudesk: archival storage, retrieval, hash verification
- [ ] Event stream publishing: all events published with correct schema

### Frontend Tests

- [ ] Component tests (Vue/React):
  - Invoice list filters, pagination, sorting
  - Approval task modal: side-by-side comparison, action buttons
  - Dashboard: KPI loading, chart rendering, drill-down filtering
- [ ] E2E tests (Playwright/Cypress):
  - User flow: login → view invoice list → open invoice detail → create approval task → approve → submit to payment
  - Admin flow: view vendor proposals → approve new vendor → invoice re-queues
  - Exception flow: detect duplicate → confirm dispute → verify MLR sent
- [ ] Performance tests: page load time, API response time, concurrent user load
- [ ] Accessibility tests: WCAG 2.1 AA compliance, keyboard navigation, screen reader

### Manual Testing (QA)

- [ ] Smoke tests: basic flows with real Peppol test messages
- [ ] User acceptance tests: krediteuren-admin, inkoper, controller personas
- [ ] Edge case tests: large invoices (500+ lines), special characters, non-EUR currency, round-off issues
- [ ] Error scenario tests: network failures, service timeouts, invalid data
- [ ] Compliance tests: UBL schema validation, NLCIUS conformance, PDF/A-3 validity
- [ ] Performance tests: 1000+ concurrent invoices in queue, dashboard response under load

---

## Phase 8: Deployment & Operations

### Documentation

- [ ] API documentation (OpenAPI/Swagger spec):
  - All endpoints, request/response schema, error codes
  - Authentication & authorization requirements
  - Rate limits
  - Examples for common workflows
- [ ] Runbook: operations procedures
  - How to handle duplicate alerts
  - How to manually attach PO to invoice
  - How to adjust tolerance rules
  - How to escalate to decidesk
  - How to retrieve archived invoices
- [ ] Architecture diagram: system components, data flows, event streams
- [ ] Database schema documentation: tables, relationships, indexes, constraints
- [ ] Third-party integrations documentation: openconnector, shillinq, purchaseq, docudesk, decidesk

### Deployment & Infrastructure

- [ ] Docker image: build container for async job services
- [ ] Database migrations: create schema, add indexes, set up audit logging
- [ ] Environment variables: API endpoints, event stream config, tolerance defaults, timeouts
- [ ] Monitoring & observability:
  - Log all invoice state changes (INFO level)
  - Log all errors & escalations (ERROR level)
  - Metrics: invoice throughput (invokes/sec), match rate (%), cycle time (avg days)
  - Alerts: error rate > 1%, cycle time > 7 days (avg), shillinq integration down
- [ ] Health check endpoint: `/health` returns service status + dependencies status
- [ ] Graceful shutdown: complete in-flight invoices before terminating

### Release Checklist

- [ ] All tests pass (unit, integration, E2E)
- [ ] Code review approved
- [ ] Documentation complete & reviewed
- [ ] Performance tests completed (benchmarks met)
- [ ] Security review completed (no vulnerabilities)
- [ ] Backup of production data before deployment
- [ ] Deployment to staging, smoke test in staging
- [ ] Deployment to production during low-traffic window
- [ ] Monitor for errors in first 24 hours
- [ ] Post-release: gather user feedback, track KPIs (STP rate, cycle time)

---

## Phase 9: Post-Launch Monitoring & Iteration

### First 30 Days

- [ ] Monitor STP rate: target ≥70%, investigate if <60%
- [ ] Monitor cycle time: target ≤5 days, investigate if >7 days
- [ ] Monitor exception rate: target <5%, track top reasons for mismatches
- [ ] Monitor error rate: target <1%, log all errors, escalate any critical patterns
- [ ] Track vendor feedback: any integration issues with Peppol submissions?
- [ ] Track user feedback: are PO-owners finding the approval UI intuitive? Any UX friction?
- [ ] Track auditor feedback: can auditors retrieve & verify archives easily?

### Optimization Opportunities

- [ ] If STP rate < 70%: analyze top mismatch reasons, consider loosening tolerances or improving GR data quality
- [ ] If cycle time > 5 days: check if approval task deadlines are being met, or if escalations are pending too long
- [ ] If error rate > 1%: root-cause analysis, fix root causes, improve error messages for users
- [ ] If top vendors have high exception rates: coordinate with them to improve data quality (better PO references, GR timeliness, invoice accuracy)

---

## Task Summary by Type

**Database (Phase 1):** 10 tasks  
**Backend Services (Phase 2):** ~60 tasks (across 10 services)  
**APIs & Events (Phase 3):** ~30 tasks (REST endpoints + event handlers)  
**Frontend (Phase 4):** ~50 tasks (5 pages + components + interactions)  
**Integrations (Phase 5):** ~15 tasks (5 integrations)  
**Configuration (Phase 6):** ~10 tasks (3 admin features)  
**Testing (Phase 7):** ~40 tasks (unit + integration + E2E + manual)  
**Deployment (Phase 8):** ~15 tasks (docs + infrastructure + release)  
**Monitoring (Phase 9):** ~10 tasks (first 30 days)  

**Total: ~240 implementation tasks**

---

## Estimated Effort

- **Backend (Phases 1-3, 5):** 8-10 weeks (2 senior backend engineers)
- **Frontend (Phase 4):** 4-5 weeks (1 senior frontend engineer + 1 UX designer)
- **QA & Testing (Phase 7):** 3-4 weeks (2 QA engineers)
- **Deployment & Monitoring (Phases 8-9):** 2-3 weeks (1 DevOps + team on-call)

**Total: 4-5 months, 5-6 FTE**

---

## Risks & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Peppol/UBL schema changes mid-project | Low | High | Lock schema version early, monitor Openpeppol announcements |
| Third-party integration delays (openconnector, shillinq) | Medium | High | Implement mock adapters for testing, parallel development |
| Low STP rate at launch (<70%) | Medium | High | Run pilot with early adopters, gather feedback, tune tolerances |
| Performance issues (high invoice volumes) | Low | High | Load testing early, scale async job queue, optimize queries |
| Compliance audit failures (7-year archival) | Low | High | Work with auditors early, validate PDF/A-3 format, test archival + retrieval flows |

