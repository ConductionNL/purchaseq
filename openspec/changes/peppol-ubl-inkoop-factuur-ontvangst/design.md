---
status: design
app: purchaseq
spec: peppol-ubl-inkoop-factuur-ontvangst
created: 2026-05-23
---

# Design: Peppol UBL Inkoopfactuur Ontvangst en Drie-weg-match

## Information Model

### Entity: InboundPeppolFactuur

Immutable record of a received Peppol invoice message.

**Primary fields:**
- `id` (UUID) — internal identifier
- `peppol_message_id` (string) — Peppol AS4 message UUID
- `received_at` (timestamp) — when the message arrived at the AP
- `sender_peppol_id` (string) — Peppol Participant ID of the vendor (e.g., `0192:12345678901234`)
- `receiver_peppol_id` (string) — Peppol Participant ID of our organization
- `ubl_document_raw` (xml, immutable) — original UBL 2.1 XML, never modified
- `document_type` (enum: Invoice | CreditNote) — parsed from UBL root element
- `validation_status` (enum: in-behandeling | valide | ongeldig) — schema + semantic check result
- `validation_errors` (jsonb) — array of error objects: `{code, severity, element, message}`
- `nlcius_compliant` (boolean) — true if all NLCIUS rules pass
- `created_at` (timestamp)
- `created_by` (user_id | system)

**Example:**

```yaml
id: "550e8400-e29b-41d4-a716-446655440000"
peppol_message_id: "urn:uuid:a1a1a1a1-b2b2-b2b2-b2b2-c3c3c3c3c3c3"
received_at: 2026-05-15T14:32:00Z
sender_peppol_id: "0192:15293949300000"  # example NL vendor
receiver_peppol_id: "0192:00000000000000"  # our org
document_type: "Invoice"
validation_status: "valide"
validation_errors: []
nlcius_compliant: true
created_at: 2026-05-15T14:32:00Z
```

---

### Entity: FactuurHeader

Parsed header data from UBL Invoice.

**Primary fields:**
- `id` (UUID)
- `inbound_peppol_factuur_id` (UUID, FK) — reference to parent
- `vendor_id` (UUID, FK) — matched to our Leverancier entity
- `vendor_peppol_id` (string) — from sender_id field, denormalized
- `vendor_name` (string)
- `vendor_btw_number` (string) — VAT ID from PartyTaxScheme
- `invoice_number_vendor` (string) — BT-1 (vendor's own invoice number)
- `invoice_date` (date) — BT-2
- `due_date` (date) — BT-9
- `po_reference` (string) — BT-13, e.g., "PO-2026-001234"
- `currency_code` (string, ISO 4217) — BT-96, default EUR
- `total_excl_vat` (decimal) — BT-109 sum
- `total_vat` (decimal) — BT-110 sum
- `total_incl_vat` (decimal) — BT-112 sum (GrandTotal)
- `payment_terms_text` (string) — net-30, 2/10, etc.
- `payment_account_iban` (string) — BT-84 CreditTransfer identifier
- `discount_early_payment` (decimal, optional) — BT-20 discount percent
- `matched_po_id` (UUID, FK, nullable) — if PO-reference lookup succeeded
- `matched_po_reference` (string) — e.g., "PO-2026-001234" for easy linking
- `created_at` (timestamp)

**Example:**

```yaml
id: "660e8400-e29b-41d4-a716-446655440001"
inbound_peppol_factuur_id: "550e8400-e29b-41d4-a716-446655440000"
vendor_id: "770e8400-e29b-41d4-a716-446655440005"
vendor_peppol_id: "0192:15293949300000"
vendor_name: "IT Services B.V."
vendor_btw_number: "NL123456789B01"
invoice_number_vendor: "F-2026-004521"
invoice_date: 2026-05-10
due_date: 2026-06-10
po_reference: "PO-2026-001234"
currency_code: "EUR"
total_excl_vat: 1500.00
total_vat: 315.00
total_incl_vat: 1815.00
payment_terms_text: "Net 30"
payment_account_iban: "NL91ABNA0417164300"
matched_po_id: "880e8400-e29b-41d4-a716-446655440010"
matched_po_reference: "PO-2026-001234"
created_at: 2026-05-15T14:32:00Z
```

---

### Entity: FactuurRegel

Invoice line from UBL InvoiceLine.

**Primary fields:**
- `id` (UUID)
- `factuur_header_id` (UUID, FK)
- `line_number` (integer) — sequence from UBL
- `article_code` (string) — SKU/EAN, if present
- `description` (string) — BT-154 name
- `quantity` (decimal) — BT-129 invoiced qty
- `unit_code` (string) — ISO 80000-1 (e.g., "PCE", "KGM")
- `unit_price` (decimal) — BT-146 net price per unit
- `line_amount_excl_vat` (decimal) — BT-131 subtotal
- `vat_category_code` (string) — S (standard) | Z (zero) | E (exempt) | etc.
- `vat_percentage` (decimal) — BT-152
- `vat_amount` (decimal) — BT-118 VAT on line
- `po_line_reference` (string) — BT-132, e.g., "1" or "PO-2026-001234-001"
- `cost_center_suggestion` (string, nullable) — from UBL AccountingReference
- `created_at` (timestamp)

**Example:**

```yaml
id: "990e8400-e29b-41d4-a716-446655440020"
factuur_header_id: "660e8400-e29b-41d4-a716-446655440001"
line_number: 1
article_code: "IT-SVC-001"
description: "Software license maintenance Q2 2026"
quantity: 1
unit_code: "PCE"
unit_price: 1500.00
line_amount_excl_vat: 1500.00
vat_category_code: "S"
vat_percentage: 21.0
vat_amount: 315.00
po_line_reference: "1"
cost_center_suggestion: "5000-IT-LICENSES"
created_at: 2026-05-15T14:32:00Z
```

---

### Entity: MatchingPoging

Result of 3-way match for a single invoice line.

**Primary fields:**
- `id` (UUID)
- `factuur_regel_id` (UUID, FK)
- `matched_po_line_id` (UUID, FK, nullable) — if PO line found
- `matched_gr_line_id` (UUID, FK, nullable) — if GR line found
- `match_status` (enum: volledig | gedeeltelijk | geen) — match quality
- `price_difference_amount` (decimal, signed) — invoice unit_price - PO unit_price
- `price_difference_percentage` (decimal, signed) — (price_diff / po_price) * 100
- `quantity_difference` (decimal, signed) — invoice_qty - po_qty
- `within_tolerance` (boolean) — true if both price & qty diffs ≤ thresholds
- `conclusion` (enum: auto-akkoord | wacht-op-accordering | auto-afkeur) — matched decision
- `tolerance_check_details` (jsonb) — {price_tolerance: 2%, abs_threshold: 25.00, qty_tolerance: 3%, ...}
- `created_at` (timestamp)
- `created_by` (system)

**Examples:**

```yaml
# Example 1: Exact match
id: "aa0e8400-e29b-41d4-a716-446655440030"
factuur_regel_id: "990e8400-e29b-41d4-a716-446655440020"
matched_po_line_id: "bb0e8400-e29b-41d4-a716-446655440040"
matched_gr_line_id: "cc0e8400-e29b-41d4-a716-446655440050"
match_status: "volledig"
price_difference_amount: 0.00
price_difference_percentage: 0.0
quantity_difference: 0
within_tolerance: true
conclusion: "auto-akkoord"
tolerance_check_details:
  price_tolerance: "2%"
  abs_threshold: "25.00"
  qty_tolerance: "3%"
  approved_at: "2026-05-15T14:32:30Z"
created_at: 2026-05-15T14:32:30Z

# Example 2: Price mismatch within tolerance
id: "dd0e8400-e29b-41d4-a716-446655440031"
factuur_regel_id: "990e8400-e29b-41d4-a716-446655440020"
matched_po_line_id: "bb0e8400-e29b-41d4-a716-446655440040"
matched_gr_line_id: "cc0e8400-e29b-41d4-a716-446655440050"
match_status: "gedeeltelijk"
price_difference_amount: 18.50
price_difference_percentage: 1.23
quantity_difference: 0
within_tolerance: true
conclusion: "wacht-op-accordering"
tolerance_check_details:
  price_tolerance: "2%"
  abs_threshold: "25.00"
  qty_tolerance: "3%"
  exceeds_abs: false
  exceeds_pct: false
created_at: 2026-05-15T14:32:30Z
```

---

### Entity: ApprovalTaak

Exception workflow task for a mismatched invoice line.

**Primary fields:**
- `id` (UUID)
- `factuur_regel_id` (UUID, FK)
- `matching_poging_id` (UUID, FK) — the mismatch that triggered this task
- `assigned_to` (user_id) — PO-eigenaar or budgethouder
- `created_at` (timestamp)
- `deadline` (date) — 5 business days from creation, configurable
- `status` (enum: open | geaccordeerd | afgekeurd | escalatie) — resolution state
- `action_taken` (enum: null | accordeer | po-adjusted | dispute | escalated)
- `action_reason` (text, nullable) — user's motivation for the choice
- `treated_at` (timestamp, nullable) — when the decision was made
- `treated_by` (user_id, nullable)

**Example:**

```yaml
id: "ee0e8400-e29b-41d4-a716-446655440060"
factuur_regel_id: "990e8400-e29b-41d4-a716-446655440020"
matching_poging_id: "dd0e8400-e29b-41d4-a716-446655440031"
assigned_to: "user-po-owner-001"
created_at: 2026-05-15T14:32:30Z
deadline: 2026-05-22
status: "geaccordeerd"
action_taken: "accordeer"
action_reason: "Leverancier heeft schriftelijk bevestigd prijsaanpassung vanwege marktomstandigheden. Goedgekeurd onder voorbehoud van credit-memo."
treated_at: 2026-05-16T10:15:00Z
treated_by: "user-po-owner-001"
```

---

### Entity: MatchingTolerantie

Organization configuration for 3-way-match thresholds.

**Primary fields:**
- `id` (UUID)
- `organization_id` (UUID, FK)
- `cost_category` (string, nullable) — if null, applies org-wide; else specific category code
- `price_tolerance_percentage` (decimal) — e.g., 2.0 for ±2%
- `price_tolerance_absolute` (decimal) — e.g., 25.00 EUR; whichever is looser
- `quantity_tolerance_percentage` (decimal) — e.g., 3.0
- `quantity_tolerance_absolute` (decimal) — e.g., 5 units
- `is_default` (boolean) — true for the fallback rule
- `created_at` (timestamp)
- `created_by` (user_id)

**Example:**

```yaml
id: "ff0e8400-e29b-41d4-a716-446655440070"
organization_id: "gg0e8400-e29b-41d4-a716-446655440080"
cost_category: null  # org default
price_tolerance_percentage: 2.0
price_tolerance_absolute: 25.00
quantity_tolerance_percentage: 3.0
quantity_tolerance_absolute: 5
is_default: true
created_at: 2026-04-01T09:00:00Z
created_by: "admin-user-001"
```

---

### Entity: FactuurDispuut

Record for disputed or rejected invoices.

**Primary fields:**
- `id` (UUID)
- `factuur_header_id` (UUID, FK)
- `dispute_reason` (enum: duplicate | invalid_charge | qty_mismatch | other)
- `dispute_message` (text) — detailed reason
- `status` (enum: gemeld | afgehandeld | escalatie) — workflow state
- `peppol_response_message_id` (string, nullable) — the ApplicationResponse UUID we sent back
- `credit_note_received_id` (UUID, FK, nullable) — if the supplier's credit-note resolves it
- `created_at` (timestamp)
- `created_by` (user_id)
- `resolved_at` (timestamp, nullable)
- `resolved_by` (user_id, nullable)

**Example:**

```yaml
id: "hh0e8400-e29b-41d4-a716-446655440090"
factuur_header_id: "660e8400-e29b-41d4-a716-446655440001"
dispute_reason: "duplicate"
dispute_message: "Duplicate invoice detected: same vendor, invoice number, and date as 2026-05-10 invoice processed earlier."
status: "gemeld"
peppol_response_message_id: "urn:uuid:dispute-resp-2026-05-15-001"
credit_note_received_id: null
created_at: 2026-05-15T14:35:00Z
created_by: "user-ap-admin-001"
resolved_at: null
resolved_by: null
```

---

### Entity: ArchiefRecord

Archival storage for compliance (7-year retention).

**Primary fields:**
- `id` (UUID)
- `factuur_header_id` (UUID, FK)
- `archival_location` (string) — docudesk URI or S3 path
- `hash_checksum_sha256` (string) — integrity proof
- `authenticity_proof` (jsonb) — Peppol signature chain metadata
- `readability_version_pdfa3` (binary) — PDF/A-3 with embedded UBL
- `retention_deadline` (date) — 7 years from invoice_date + 1 month, configurable
- `destruction_scheduled_at` (timestamp, nullable)
- `destroyed_at` (timestamp, nullable)
- `created_at` (timestamp)

**Example:**

```yaml
id: "ii0e8400-e29b-41d4-a716-446655440100"
factuur_header_id: "660e8400-e29b-41d4-a716-446655440001"
archival_location: "s3://docudesk-eu/purchaseq/invoices/2026-05-15/550e8400-e29b-41d4-a716-446655440000"
hash_checksum_sha256: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
authenticity_proof:
  peppol_signature_chain: "urn:uuid:signature-2026-05-15-001"
  timestamp: "2026-05-15T14:32:00Z"
  signer_certificate_thumbprint: "abc123def456..."
readability_version_pdfa3: "[binary blob]"
retention_deadline: 2033-06-15
destruction_scheduled_at: null
destroyed_at: null
created_at: 2026-05-15T14:32:30Z
```

---

## Workflows

### Workflow 1: Inbound Receipt & Validation

```
Peppol AP (openconnector)
  ↓ [publishes InboundPeppolFactuur event]
purchaseq peppol-receiver (async job)
  ├─ Parse UBL XML
  ├─ Validate schema (BIS 3.0 + NLCIUS)
  ├─ Extract document metadata
  ├─ Create InboundPeppolFactuur record (validation_status = in-behandeling)
  ├─ IF validation errors → set validation_status = ongeldig, send MLR negative to AP, STOP
  ├─ ELSE → set validation_status = valide
  ├─ Create FactuurHeader + FactuurRegel records
  ├─ Publish FactuurValidated event
  └─ Enqueue invoice for leverancier matching

Matching service (async job triggered by FactuurValidated)
  ├─ Lookup vendor by Peppol ID (primary key)
  ├─ IF not found → lookup by KvK from EndpointID (secondary)
  ├─ IF still not found → create "Voorstel nieuwe leverancier" task, STOP
  ├─ ELSE → set FactuurHeader.vendor_id, Publish VendorMatched event
  └─ Enqueue invoice for PO lookup
```

---

### Workflow 2: PO Lookup & 3-Way Match

```
PO Matching service (async job triggered by VendorMatched)
  ├─ Extract BT-13 (po_reference) from FactuurHeader
  ├─ IF po_reference is null/empty → move to "no-PO" queue, STOP (or default to expense invoice)
  ├─ ELSE search purchaseq PO by reference
  ├─ IF not found → create "PO non trovato" task for AP, mark status = waiting-po, STOP
  ├─ ELSE validate vendor on PO matches invoice vendor
  ├─ IF vendor mismatch → log error, escalate to AP, STOP
  ├─ ELSE set FactuurHeader.matched_po_id, Publish POMatched event
  └─ Enqueue invoice for GR lookup

GR Matching service (async job triggered by POMatched)
  ├─ Search GR registrations linked to the matched PO
  ├─ IF GR found and received_qty matches or exceeds invoice qty → record GR line link
  ├─ ELSE GR optional; continue with 2-way match (invoice ↔ PO)
  └─ Publish GRMatched event, enqueue for line-level 3-way match

Line Matching service (async job triggered by GRMatched)
  ├─ For each FactuurRegel:
  │  ├─ Find matching PO line (usually by line_number or article_code)
  │  ├─ Compare qty: invoice_qty vs po_qty; calc difference & tolerance check
  │  ├─ Compare price: invoice_unit_price vs po_unit_price; calc difference & tolerance check
  │  ├─ Load MatchingTolerantie rules (org-wide or by cost_category)
  │  ├─ Create MatchingPoging record with match_status (volledig|gedeeltelijk|geen)
  │  ├─ IF match_status = volledig AND within_tolerance → conclusion = auto-akkoord
  │  ├─ ELSE IF match_status = gedeeltelijk AND within_tolerance → conclusion = wacht-op-accordering, create ApprovalTaak
  │  ├─ ELSE → conclusion = wacht-op-accordering, create ApprovalTaak
  │  └─ Publish LineMatched event
  ├─ Check if ALL FactuurRegel lines are auto-akkoord
  └─ Publish FactuurReadyForPayment or FactuurAwaitingApproval event
```

---

### Workflow 3: Exception Handling & Approval

```
Approval Task UI (PO-eigenaar receives push notification)
  ├─ Navigate to Approval > Pending
  ├─ Click on mismatched line
  ├─ View side-by-side comparison:
  │  ├─ PO-regel: unit_price, qty, line_amount
  │  ├─ GR-regel (if exists): received_qty, date
  │  └─ FactuurRegel: unit_price, qty, line_amount [highlighted differences]
  ├─ Choose action:
  │  ├─ A) "Accordeer" → enter reason, save
  │  │   └─ Set ApprovalTaak.status = geaccordeerd, action_taken = accordeer
  │  │   └─ If ALL lines now approved, trigger FactuurReadyForPayment event
  │  │
  │  ├─ B) "PO bijwerken" → edit PO price/qty, save
  │  │   └─ Update PO line in purchaseq base
  │  │   └─ Trigger re-match for this invoice (LineMatchingService runs again)
  │  │   └─ If ALL lines now match, skip ApprovalTaak approval step, auto-advance
  │  │
  │  └─ C) "Betwisten" → write reason, confirm
  │      └─ Create FactuurDispuut record
  │      └─ Generate Peppol ApplicationResponse (negative)
  │      └─ Send response via AP, log message_id
  │      └─ Set FactuurHeader status = in-dispuut, ApprovalTaak status = afgekeurd
```

---

### Workflow 4: Payment Submission & Archival

```
Payment Submission service (triggered by FactuurReadyForPayment event)
  ├─ Verify ALL FactuurRegel lines have conclusion = auto-akkoord or geaccordeerd
  ├─ Lock FactuurHeader for further modifications (read-only)
  ├─ Extract payment details:
  │  ├─ IBAN from FactuurHeader.payment_account_iban
  │  ├─ amount = FactuurHeader.total_incl_vat
  │  ├─ due_date from FactuurHeader.due_date
  │  ├─ discount (if applicable, from BT-20)
  │  └─ reference = vendor_invnum for vendor reconciliation
  ├─ Create event InvoiceReadyForPayment with payment DTO
  ├─ Publish to shillinq event stream
  ├─ Set FactuurHeader status = submitted-to-payment
  └─ Publish FactuurSubmittedToPayment event

Archival service (triggered by FactuurValidated event + FactuurSubmittedToPayment event)
  ├─ Retrieve original UBL from InboundPeppolFactuur.ubl_document_raw
  ├─ Calculate SHA-256 hash of UBL
  ├─ Extract Peppol signature chain from message metadata
  ├─ Generate PDF/A-3 rendering with UBL embedded as attachment
  ├─ Store to docudesk:
  │  ├─ UBL XML (immutable original)
  │  ├─ Signature chain metadata
  │  ├─ PDF/A-3 file
  │  └─ Hash checksum
  ├─ Create ArchiefRecord
  ├─ Set retention_deadline = invoice_date + 7 years + 1 month
  ├─ Monitor for automatic destruction after deadline expires
  └─ Publish FactuurArchived event
```

---

## Integration Points

### Upstream: openconnector peppol-e-invoicing-adapter

**Input:** Peppol AS4 message (UBL Invoice or CreditNote)

**Event flow:**
```
peppol-e-invoicing-adapter
  ├─ Receive AS4 message
  ├─ Validate AS4 envelope
  ├─ Extract payload (UBL XML)
  └─ Publish InboundPeppolMessage event
    └→ purchaseq subscribes, creates InboundPeppolFactuur + FactuurHeader + FactuurRegel
```

**Output:** Peppol ApplicationResponse (MLR positive if valid, negative if schema errors)

---

### Downstream: shillinq

**Input event:** InvoiceReadyForPayment
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

**Expected outcome:** shillinq creates payment instruction, scheduled for due_date minus X days (configurable).

---

### Bidirectional: purchaseq base (PO + GR)

**Reads:**
- PO lines (for matching)
- GR lines (for 3-way match)
- GR receipt date (for aging calc)

**Updates:** (if PO owner clicks "PO bijwerken")
- PO line unit_price
- PO line qty

---

### Downstream (escalation): decidesk

**Trigger:** Significant disputes or threshold breaches
- Disputed invoice amount > EUR 10,000
- Vendor relationship at risk (N+ disputed invoices in 30 days)

**Event:** EscalateInvoiceDispute

---

### Downstream (archival): docudesk

**Input:** ArchiefRecord with UBL, PDF/A-3, signature chain

**Output:** 7-year retention proof, destruction schedule

---

## UI Layout (Pages & Components)

### Page 1: Facturen (List View)

Location: **Inkopen > Facturen** (per ADR-001)

**Sections:**
1. **Quick stats** (3 widgets)
   - STP rate (last 30 days): 72%
   - Avg. cycle time: 4.2 days
   - Open approval tasks: 3

2. **Filters & sort**
   - Status: All | In Ontvangst | Matching | Awaiting Approval | Approved | Disputed
   - Date range: Last 7/30/90 days
   - Vendor: dropdown or search
   - Sort: date desc | amount desc | status

3. **Table:**
   - Columns: invoice_date | vendor_name | invoice_number | total_incl_vat | status [badge] | actions
   - Inline status badges:
     - 🟢 auto-akkoord
     - 🟡 awaiting-approval (click to open task)
     - 🔴 in-dispuut
   - Row click → detail page

---

### Page 2: Factuur Detail

Location: **Inkopen > Facturen > [invoice_id]**

**Sections:**
1. **Header summary**
   - Vendor name & Peppol ID
   - Invoice number (vendor) | Date | Due date
   - Status badge + any alerts
   - Actions: (Dispute | Print | Archive link)

2. **Financial summary**
   - Total excl VAT | VAT | Total incl VAT
   - Payment account (IBAN masked)

3. **Matching summary** (side-by-side)
   - PO reference + link to PO
   - GR link (if matched)
   - Overall match status (volledig | gedeeltelijk)

4. **Line-by-line detail** (expandable rows)
   - For each FactuurRegel:
     ```
     [+] Line 1: Software license Q2 2026
         Invoice:  1 × €1500.00 = €1500.00
         PO:       1 × €1500.00 = €1500.00
         Match:    ✓ auto-akkoord
     ```
   - On expand, show:
     - PO matched line
     - GR matched line (if exists)
     - MatchingPoging details (price diff, qty diff, tolerance check)
     - If awaiting approval: action buttons (Approve | Adjust PO | Dispute)

---

### Page 3: Approval Task

Location: **Approval > [task_id]** (or modal from list)

**Layout:**
1. **Task header**
   - Assigned to: [name] | Deadline: [date] [color: red if overdue]
   - Invoice: [vendor] [date] [amount]

2. **Side-by-side comparison**
   ```
   ┌──────────────┬──────────────┬──────────────┐
   │ PO Line 1    │ GR Line 1    │ Invoice L. 1 │
   ├──────────────┼──────────────┼──────────────┤
   │ SKU: IT-SVC  │ Received: XX │ Qty: 1       │
   │ Qty: 1 PCE   │ Date: 2026.. │ Price: 1518€ │  ← price +1.2%
   │ Price: 1500€ │ ...          │ VAT: 318.78€ │
   └──────────────┴──────────────┴──────────────┘
   ```
   Differences highlighted in red.

3. **Decision buttons**
   - "Accordeer": Opens text area for reason, saves approval
   - "PO bijwerken": Opens inline editor for PO line, re-match on save
   - "Betwisten": Opens dispute form, generates Peppol response

---

### Page 4: Dashboard (Controller / Financial Manager)

Location: **Inkopen > Facturen > Dashboard** (or separate tab on main view)

**KPI cards:**
- STP rate (%) last 30/90/365 days
- Avg cycle time (days)
- Total invoice volume & value
- Exception rate (%)

**Charts:**
- STP rate trend (line chart, 30-day rolling)
- Invoice value by status (pie: auto-akkoord | awaiting | disputed)
- Top 10 vendors by exception rate (bar chart)

**Open tasks table:**
- User | # of open tasks | oldest deadline | overdue count

---

## Data Validation Rules

1. **InboundPeppolFactuur.validation_status** must move in order: in-behandeling → (valide | ongeldig)
2. **FactuurHeader.vendor_id** must exist before lines are matched (REQ-002)
3. **FactuurHeader.matched_po_id** can be null (no-PO invoices), but if not null, the PO must exist and vendor must match
4. **FactuurRegel lines** must have non-null quantity, unit_price, and vat_percentage
5. **MatchingPoging.match_status** logic:
   - `volledig` → both qty and price within tolerance
   - `gedeeltelijk` → one of qty/price differs but within tolerance
   - `geen` → no PO line found
6. **ApprovalTaak.assigned_to** must be a user with role "inkoper" or "budgethouder"
7. **ArchiefRecord.retention_deadline** = invoice_date + 7 years, cannot be modified after creation

---

## Constraints & Limits

- Invoice document size: max 50 MB (UBL + attachments)
- Max 500 lines per invoice
- Tolerance configuration: 0.0% to 10.0% (price), 0 to 999 units (qty)
- Approval task deadline: 5 business days (configurable per org)
- Archive retention: minimum 7 years, max 20 years
- Concurrent 3-way matches: up to 100/second (async job queue)
