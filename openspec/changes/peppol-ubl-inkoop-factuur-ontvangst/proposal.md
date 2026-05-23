---
status: proposal
app: purchaseq
spec: peppol-ubl-inkoop-factuur-ontvangst
created: 2026-05-23
depends_on:
  - openconnector peppol-e-invoicing-adapter
  - shillinq
  - purchaseq base
target_users:
  - Inkoper / contractmanager (PO-eigenaar)
  - Magazijnmedewerker / goederen-ontvanger (GR)
  - Crediteuren-administrateur (AP)
  - Budgethouder / kostenplaats-eigenaar
  - Controller / financieel beheerder
  - Auditor / accountant
---

# Peppol UBL Inkoopfactuur Ontvangst en Drie-weg-match

## Demand

**Target:** Fully automated inbound Peppol invoice receipt with 3-way matching to PO and GR, and straight-through processing to payment.

### Market Context

- **Regulatory:** Peppol is mandatory for Dutch government procurement; rapidly growing in commercial market.
- **Current reality:** Many organizations receive Peppol invoices but manually re-enter them into ERP, losing automation gains.
- **Problem:** Invoices land in a mailbox → manual entry → errors, delays, lost audit trail, no conformance proof.
- **Opportunity:** Eliminate manual re-entry, achieve 70-80% straight-through-processing (STP) rate, prove compliance.

### Features & Demand Scores

| Feature | Demand | Description |
|---------|--------|-------------|
| UBL validation (BIS 3.0 + NLCIUS) | 9/10 | Schema validation, conformance check, reject invalid syntax |
| Leverancier matching (Peppol ID + KvK) | 9/10 | Auto-match on Peppol ID, secondary on KvK; propose new suppliers |
| PO lookup via BT-13 reference | 9/10 | Match invoice to purchase order, handle missing/invalid PO refs |
| 3-way match (qty/price within tolerances) | 9/10 | Compare invoice ↔ PO ↔ GR per line, configurable tolerances |
| Guided exception workflow | 8/10 | Side-by-side diff for mismatches; PO owner decides: approve / adjust PO / dispute |
| Auto-pass to shillinq | 8/10 | Send approved invoices to payment app with verified data |
| Dispute workflow (Peppol response) | 7/10 | Generate UBL Response, send back via AP, track resolution |
| Duplicate detection | 7/10 | Block or alert on vendor-invnum-date collision |
| Compliance archival (7-year audit trail) | 9/10 | Store UBL + signature chain + integrity hash + PDF/A-3 |
| STP dashboard (rate, SLA, exceptions) | 7/10 | Monitor process KPIs, top suppliers with mismatches, open tasks |

### User Stories

#### Story 1: Inkoper receives and auto-approves compliant invoice

**As a** Inkoper (PO-eigenaar)  
**I want to** receive invoices that match my PO automatically, without manual re-entry or approval work  
**So that** I can keep procuring without invoice-admin burden, and auditors can see a complete STP trail.

**Acceptance Criteria:**
- GIVEN a valid Peppol invoice arrives with matching PO and GR
  - WHEN all lines match qty/price within tolerance
  - THEN the invoice is marked "auto-approved" and sent to shillinq within 5 seconds; notification shows "Factuur X van leverancier Y doorgezet naar betaling"
- GIVEN the invoice is stored in the archival layer
  - WHEN the auditor requests the original UBL
  - THEN the original UBL + Peppol signature + hash are retrievable

#### Story 2: Krediteurenadministrator handles new supplier from Peppol

**As a** Krediteurenadministrator  
**I want to** review suppliers that appear for the first time via Peppol and decide whether to add them  
**So that** we don't auto-process invoices from suppliers we don't want to use.

**Acceptance Criteria:**
- GIVEN a Peppol invoice arrives from an unknown Peppol ID
  - WHEN the system cannot match the vendor to our database
  - THEN a "Voorstel nieuwe leverancier" task is created for me with all UBL-extracted data (name, BTW, IBAN, contact)
- GIVEN I approve the supplier
  - WHEN I save the approval
  - THEN the invoice re-enters the matching queue and proceeds to 3-way match
- GIVEN I reject the supplier
  - WHEN I click "Weigeren"
  - THEN the invoice stays in "in-behandeling" status and an auto-reply is sent to the Peppol AP

#### Story 3: PO owner resolves price mismatch with guided choice

**As a** Budgethouder (PO-eigenaar)  
**I want to** see side-by-side when a Peppol invoice line differs from my PO, and quickly decide  
**So that** I can approve the invoice OR update the PO once AND move on.

**Acceptance Criteria:**
- GIVEN a factuurregel has a 2.5% price difference from the PO line
  - WHEN I open the approval-taak
  - THEN I see three columns: PO-regel | Goederenontvangst | Factuurregel with price/qty diffs highlighted red
- GIVEN I choose "Accorderen met motivering"
  - WHEN I save with a short reason ("Afgesproken prijsaanpassing")
  - THEN the line is marked "geaccordeerd" and the invoice continues to shillinq
- GIVEN I choose "PO bijwerken"
  - WHEN I click "PO aanpassen" and update the price
  - THEN the system automatically re-matches and confirms all lines now pass; the invoice auto-advances

#### Story 4: Crediteurenadministrator disputes invoice for duplicate/invalid charge

**As a** Krediteurenadministrator  
**I want to** block a suspicious invoice and send a formal dispute back to the supplier via Peppol  
**So that** the invoice doesn't disappear and we have proof we rejected it.

**Acceptance Criteria:**
- GIVEN I detect a duplicate invoice (same vendor-invnum-date)
  - WHEN I click "Betwisten"
  - THEN the system flags the original invoice with reason and displays a form to write a reason
- GIVEN I submit the dispute
  - WHEN the form is saved
  - THEN a Peppol ApplicationResponse (rejected) is sent to the supplier's AP, the invoice is marked "in-dispuut", and I receive a log of the sent message
- GIVEN the supplier sends a credit note
  - WHEN the credit note arrives and is matched to the dispute
  - THEN the dispute is auto-closed and logged as "afgehandeld"

### Stakeholder Profiles

| Stakeholder | Goals | Responsibilities | Key Concerns |
|---|---|---|---|
| **Inkoper / PO-eigenaar** | Fast PO cycle, no invoice-admin burden | Create POs, review exceptions, adjust prices as needed | Wants 70%+ STP, wants exceptions in <24h |
| **Magazijnmedewerker** | Accurate goods receipt | Register GR timely, flag discrepancies | Wants GR-to-invoice matching to work automatically |
| **Krediteurenadministrator** | Compliant, audit-ready supplier onboarding | Review new suppliers, handle disputes, manage tolerances | Wants one-click new supplier approval, wants dispute trail |
| **Budgethouder** | Cost control, approval efficiency | Monitor spend, approve exceptions within budget | Wants red flags for budget breaches, wants deadline reminders |
| **Controller / Financieel beheerder** | Complete audit trail, STP rate monitoring | Configure tolerances, review KPIs, investigate anomalies | Wants 70%+ STP, wants dashboard drill-down |
| **Auditor / Accountant** | Prove authenticity, integrity, readability (Btw-richtlijn) | Retrieve original UBL + signature chain + hash + PDF | Wants immutable archive, wants 7-year retention proof |

### Regulatory & Standards Compliance

- **Peppol BIS Billing 3.0** — invoice structure, validation rules
- **EN 16931** (Europese norm elektronisch factureren) — semantic requirements
- **UBL 2.1** — XML syntax binding
- **NLCIUS** — Dutch implementation guidelines (Forum Standaardisatie)
- **Btw-richtlijn 2010/45/EU / Wet OB 1968** — authenticiteit, integriteit, leesbaarheid
- **Archiefwet** — 7-year retention requirement for tax purposes
- **Wet elektronische facturering bij overheidsopdrachten** — mandatory for government

### Cross-app Dependencies

- **openconnector peppol-e-invoicing-adapter** (upstream) — Peppol AP, SMP registration, AS4 transport, event stream
- **shillinq** (downstream) — payment processing, GL posting
- **purchaseq base** — PO master, GR registry
- **docudesk** — long-term archival with integrity/authenticity proof
- **decidesk** — escalation for significant disputes
- **openconnector kvk-adapter** — new supplier verification

### Placement & Information Architecture

**Type:** `SUB_PAGE`  
**Location:** Inkopen > Facturen (per ADR-001 rule 7)  
**Rationale:** Inbound invoices are operational work (matching, approval, forwarding), not configuration. They live where daily purchasing happens.

### Success Metrics

- **STP rate:** ≥70% of invoices auto-approved without manual intervention within 30 days of go-live
- **Cycle time:** Average time from invoice receipt to payment instruction ≤5 days (currently >15 days with manual entry)
- **Compliance:** 100% of archived invoices retrievable with original UBL + signature chain + hash for 7 years
- **Exception rate:** <5% of invoices flagged for manual review (current baseline: 40% re-entry errors)
- **Adoption:** ≥80% of inkoper roles actively using the 3-way-match view within 60 days

### Assumptions & Constraints

- openconnector peppol-e-invoicing-adapter is operational and delivering events
- GR (goederen-ontvangst) data is registered in purchaseq base within ~1-2 days of goods arrival
- PO lines are complete and accurate (UBL comparison does not detect PO data quality issues)
- Tolerances are configurable per organization and per cost center (not fixed)
- Dispute resolution (credit notes, corrections) is out of scope for this spec; decidesk/shillinq handle payment blocking

### Out of Scope

- Supplier credit management (separate app or module)
- Multi-currency exchange rate management (use shillinq rates)
- GL posting or accounting integration (shillinq owns that)
- Goods return workflow (GR reversal is separate from invoice matching)
- Intra-company invoice flows (only B2B/B2G Peppol invoices)
