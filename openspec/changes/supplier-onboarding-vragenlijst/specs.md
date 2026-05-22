# Specifications: Supplier Onboarding — Vragenlijst & Workflow

## Requirements

### REQ-001: Category-Driven Questionnaire Branching

**Intent**: Questionnaire adapts to supplier category and risk tier, showing only relevant questions and documents.

**Acceptance Criteria**:

GIVEN a supplier is on the self-service intake portal
WHEN they select category "logistiek" (low risk)
THEN the questionnaire shows:
  - Company info questions (KvK, name, address, contact)
  - Transport licensing questions (license type, expiry)
  - Insurance questions (cargo, liability)
THEN the questionnaire DOES NOT show:
  - BIG healthcare registration questions
  - Child care certification questions
  - Environmental compliance questions

GIVEN a supplier selects category "zorg" (high risk)
WHEN the questionnaire loads
THEN the questionnaire shows:
  - Company info questions
  - BIG healthcare registration questions (mandatory)
  - Child care/special populations certifications
  - GDPR/health data handling questions
  - Integrity (BIBOB) disclosure questions
THEN risk tier is automatically set to "hoog"

GIVEN a supplier selects category "bouw" with annual turnover < €50k
WHEN the questionnaire loads
THEN the questionnaire shows company info + construction certifications
THEN MVI proof document upload is optional

GIVEN a supplier in category "bouw" with annual turnover > €50k
WHEN the questionnaire loads
THEN MVI proof document upload is flagged REQUIRED

---

### REQ-002: Real-Time KvK Validation

**Intent**: Supplier enters KvK number; on blur, purchaseq calls KvK-API and pre-fills derived fields or flags error.

**Acceptance Criteria**:

GIVEN a supplier enters KvK number "34056216" in the company info section
WHEN the KvK field loses focus (blur event)
THEN purchaseq calls openconnector KvK-API adapter
THEN on success:
  - handelsnaam field is pre-filled with "ACME Logistics B.V."
  - rechtsvorm field is pre-filled with "besloten vennootschap"
  - vestigingsadres fields (street, number, postcode, city) are pre-filled
  - SBI code is stored (used for category validation if needed)
  - Validatie-status in VragenlijstAntwoord is set to "ok"
  - Validatie-bron is set to "KvK-API"

GIVEN the KvK number "99999999" (does not exist)
WHEN the KvK field loses focus
THEN purchaseq calls KvK-API
THEN on miss:
  - The KvK field is marked with red error border
  - Error message: "KvK number not found. Please verify and retry, or contact support for manual verification."
  - Validatie-status is set to "fout"
  - handelnaam, rechtsvorm, vestigingsadres remain unchanged (supplier can fill manually)
  - A manual review task is created for the categorie-inkoper

GIVEN the KvK number field is blank or invalid (fewer than 8 digits)
WHEN the supplier tries to submit the questionnaire
THEN the form validation prevents submission
THEN error message: "KvK number is required and must be 8 digits."

GIVEN a supplier corrects a KvK entry after a failed validation
WHEN the field loses focus again
THEN purchaseq re-calls KvK-API with the corrected number

---

### REQ-003: IBAN Validation with Name-on-Account Check

**Intent**: Supplier enters IBAN; on save, purchaseq validates checksum and (via Surepay) verifies account holder name.

**Acceptance Criteria**:

GIVEN a supplier enters IBAN "NL91ABNA0417164300"
WHEN the IBAN field is saved (form save, not just blur)
THEN purchaseq validates IBAN checksum using IBAN algorithm (mod-97)
THEN on success:
  - IBAN-validatie-status is set to "ok"
  - If Surepay adapter is available:
    - Supplier also enters "IBAN tenaamstelling" field (account holder name)
    - Purchaseq calls openconnector Surepay adapter: POST {iban, name}
    - Surepay returns {valid, name_match: true|false}
    - If name_match=true: tenaamstelling-status is "ok"
    - If name_match=false: tenaamstelling-status is "mismatch" (yellow warning)
      Error message: "Account holder name does not match company name. Please verify with your bank."
      Dossier CAN move to internal_review but flagged for financien review.

GIVEN an IBAN with invalid checksum "NL91ABNA0417164301" (last digit wrong)
WHEN the IBAN field is saved
THEN checksum validation fails
THEN IBAN-validatie-status is set to "fout"
THEN form prevents submission
THEN error message: "Invalid IBAN checksum. Please verify the number."

GIVEN a supplier provides IBAN for a country outside NL/EU (e.g., CH, GB)
WHEN the IBAN field is saved
THEN Surepay adapter may not support the country
THEN validatie-status is "handmatig" (manual review required by financien)
THEN yellow warning: "This IBAN will require manual verification by our finance team."

GIVEN IBAN and tenaamstelling are both entered correctly with name match
WHEN the dossier is submitted for internal review
THEN dossier → status: intern_review with IBAN validatie-status: ok

GIVEN IBAN and tenaamstelling have name mismatch
WHEN dossier reaches the financien approval step
THEN the financien reviewer sees yellow flag on IBAN section
THEN decision options: Approve (trust supplier's explanation), Request-Info, or Reject
THEN if approve despite mismatch: additional notation recorded in ApprovalStep motivatie

---

### REQ-004: BIBOB Risk-Tiering and Automatic Routing

**Intent**: Dossier in a risk-hoog category automatically triggers BIBOB assessment and routes to integrity coordinator.

**Acceptance Criteria**:

GIVEN a supplier selects category "zorg" (configured as risk-hoog in municipal BIBOB policy)
WHEN the questionnaire is loaded
THEN risicoTier is automatically set to "hoog"

GIVEN a supplier selects category "logistiek" (configured as risk-laag)
WHEN the questionnaire is loaded
THEN risicoTier is automatically set to "laag"

GIVEN an OnboardingDossier with risicoTier="hoog" is submitted (status → intern_review)
WHEN the ApprovalSteps are created
THEN a BIBOBToets record is automatically created:
  - status: "openstaand"
  - aanleiding: "Leverancier in categorie {categorie} (risico-hoog)"
  - behandelaar: assigned to integriteitscoördinator role (via permission handler)
  - vastgelegd_op: current timestamp

GIVEN a BIBOBToets exists for a dossier
WHEN the approval workflow begins
THEN ApprovalSteps include BOTH sequential steps (categorie-inkoper → contractmanager → financien) AND a parallel integriteit step
THEN financien step can proceed in parallel but MUST wait for integriteit conclusion before final approval

GIVEN a BIBOBToets is marked conclusie="afgewezen" (BIBOB rejection)
WHEN the integriteit step completes
THEN dossier → status: afgewezen (rejected)
THEN supplier receives rejection email citing BIBOB outcome

GIVEN a BIBOBToets is marked conclusie="met_voorbehoud" (conditional approval)
WHEN the integriteit step completes
THEN dossier CAN proceed to final approval (goedgekeurd) with notation in audit trail
THEN contractmanager and financien are notified of the caveat in their review

GIVEN a dossier with risicoTier="laag" is submitted
WHEN the ApprovalSteps are created
THEN NO BIBOBToets record is created
THEN approval workflow is: categorie-inkoper → contractmanager → financien (no parallel integriteit)

---

### REQ-005: MVI Compliance Proof Requirement

**Intent**: Suppliers with expected annual turnover > €50k must upload MVI compliance proof; missing proof blocks internal_review transition.

**Acceptance Criteria**:

GIVEN a supplier enters expected annual turnover "€75,000" on the questionnaire
WHEN the questionnaire saves
THEN purchaseq detects annual turnover > €50k threshold
THEN the document upload section is updated:
  - MVI proof document type becomes REQUIRED (red asterisk)
  - Help text: "As your annual turnover exceeds €50k, you must provide proof of socially responsible procurement compliance (MVI). Acceptable: PSO certificate, CO2 Performance Ladder certification, or signed local MVI self-declaration."

GIVEN the supplier has entered turnover €45,000
WHEN the questionnaire saves
THEN MVI proof remains optional

GIVEN a supplier with €75k turnover has NOT uploaded MVI proof
WHEN they click the "Submit for Review" button
THEN form validation prevents submission
THEN error: "MVI compliance proof is required. Please upload one of: PSO certificate, CO2 Ladder, or MVI declaration."

GIVEN a supplier uploads a file named "PSO-30+-Certificate-2026.pdf" to the MVI proof field
WHEN the file is uploaded
THEN Bewijsstuk record is created with:
  - type: "mvi-cert"
  - bestand: link to docudesk document
  - geldigTot: extracted from PDF (if certificate) or set to 1 year from upload date (if self-declaration)
  - geverifieerd: false (requires manual review)

GIVEN a supplier submits questionnaire with MVI proof uploaded
WHEN dossier → status: intern_review
THEN ApprovalSteps automatically include a note for contractmanager:
  - "Verify MVI compliance proof is valid, not expired, and matches company profile."

GIVEN the contractmanager reviews the dossier and MVI proof is expired
WHEN they mark their approval decision
THEN they MUST select "Request-Info" (cannot approve)
THEN auto-message to supplier: "Your MVI proof has expired. Please provide a current certificate or declaration."

---

### REQ-006: Peppol Directory Validation

**Intent**: Supplier enters Peppol ID; purchaseq queries Peppol-SMP to confirm endpoint activity and supported document types.

**Acceptance Criteria**:

GIVEN a supplier enters Peppol ID "9907:nl001234567b99" in the contact info section
WHEN the Peppol field loses focus
THEN purchaseq calls openconnector Peppol-SMP adapter
THEN on success:
  - Peppol endpoint is confirmed as active
  - Supported document types are returned (e.g., ["UBL-Invoice", "OrderResponse"])
  - A PeppolEndpoint record (or VragenlijstAntwoord entry) is created with:
    - peppolId: "9907:nl001234567b99"
    - endpointActive: true
    - supportedDocuments: ["UBL-Invoice", "OrderResponse"]
    - validatedAt: current timestamp
    - validatieBron: "Peppol-SMP"

GIVEN a supplier enters Peppol ID "9907:nl999999999999" (inactive)
WHEN the Peppol field loses focus
THEN Peppol-SMP query returns no active endpoint
THEN Peppol validation-status is set to "fout"
THEN warning message: "Peppol endpoint not found in directory. You may still proceed, but we recommend activating your Peppol profile at https://www.peppol.eu/."

GIVEN a supplier enters an invalid Peppol ID format
WHEN the Peppol field loses focus
THEN validation-status is "fout"
THEN error message: "Invalid Peppol ID format. Expected format: 9907:NLDDD12345678XXX"

GIVEN a supplier leaves the Peppol field blank
WHEN the questionnaire is submitted
THEN Peppol is optional for low-risk suppliers
THEN if not provided, validatie-status is "blank" (no requirement)

---

### REQ-007: Multi-Step Approval with Role-Based Routing

**Intent**: Dossier transitioning to intern_review triggers sequential ApprovalSteps per role; supplier pushed to ERP only after all approvals.

**Acceptance Criteria**:

GIVEN an OnboardingDossier with status="ingevuld" and all validations pass
WHEN the supplier clicks "Submit for Review"
THEN dossier → status: intern_review

GIVEN dossier → status: intern_review
WHEN the approval routing logic executes
THEN the following ApprovalSteps are created in sequence:
  1. volgorde=1, rol="categorie-inkoper", status="openstaand"
  2. volgorde=2, rol="contractmanager", status="openstaand"
  3. volgorde=3, rol="financien", status="openstaand"
  + if risicoTier="hoog": volgorde=4 (parallel), rol="integriteit", status="openstaand"

GIVEN ApprovalSteps are created
WHEN the categorie-inkoper opens the dossier in the approval dashboard
THEN they see the dossier with questionnaire responses and documents
THEN they can enter beslissing: "approve" / "request-info" / "reject"
THEN they must provide motivatie (required)

GIVEN the categorie-inkoper approves (beslissing="approve")
WHEN they save
THEN ApprovalStep volgorde=1 → status="afgewerkt"
THEN the contractmanager is notified (email, inbox, task)

GIVEN the categorie-inkoper requests info (beslissing="request-info")
WHEN they save
THEN ApprovalStep volgorde=1 → status="afgewerkt_met_vraag"
THEN supplier receives email: "Additional information requested: {motivatie}"
THEN supplier can resubmit via portal; dossier goes back to ingevuld
THEN contractmanager does NOT see the dossier yet

GIVEN the categorie-inkoper rejects (beslissing="reject")
WHEN they save
THEN ApprovalStep volgorde=1 → status="afgewerkt"
THEN dossier → status="afgewezen"
THEN supplier receives rejection email
THEN approval workflow ends; no further steps

GIVEN all sequential ApprovalSteps are "afgewerkt" with approval
AND (if risicoTier="hoog") the integriteit step is also approved
WHEN the financien step is marked approve
THEN dossier → status="goedgekeurd"

GIVEN dossier → status="goedgekeurd"
WHEN the ERP sync job runs
THEN:
  - If leverancier does not exist in ERP: CREATE in ERP (POST to openconnector adapter with leverancier data)
  - If leverancier exists: UPDATE ERP record
  - Push leverancier to crediteurenstam (general ledger) with verified IBAN & address
  - Set dossier.vervaldatum = today + 12 months
  - Send welcome email to supplier: "You are now approved and active in our procurement system."

---

### REQ-008: Annual Revalidation Workflow

**Intent**: Approved dossier reaching expiry date triggers annual revalidation; suppliers can confirm unchanged in 1 click or re-submit changes in expedited track.

**Acceptance Criteria**:

GIVEN an OnboardingDossier with status="goedgekeurd" and vervaldatum="2027-05-20"
WHEN the daily-revalidation-check background job runs on 2027-05-21
THEN the dossier is marked for revalidation
THEN a revalidation email is sent to the leverancier contact email
THEN email contains:
  - Link to "Confirm without changes" (1-click, no login required via token)
  - Link to "Update information" (opens full questionnaire in edit mode)
  - "Your onboarding expires on {vervaldatum}. Response required within 60 days."

GIVEN the supplier clicks "Confirm without changes" link
WHEN the link is processed
THEN dossier → status: "goedgekeurd"
THEN vervaldatum += 12 months (new expiry: 2028-05-20)
THEN audit trail records: "Supplier confirmed unchanged at {timestamp}"
THEN no approval steps are triggered

GIVEN the supplier clicks "Update information" link
WHEN they log in and edit the questionnaire
THEN the questionnaire opens in edit mode with previous answers pre-filled
THEN supplier can update answers and re-upload documents as needed
THEN on save/submit:
  - Purchaseq detects which questions have changed
  - Only changed questions are re-validated (KvK, IBAN, Peppol, etc.)
  - Re-upload of unchanged documents is optional

GIVEN a supplier updates their IBAN on revalidation
WHEN IBAN validation is re-run and fails
THEN the revalidation is blocked with error
THEN supplier must correct and resubmit

GIVEN a supplier updates information and resubmits
WHEN the updated dossier is submitted
THEN expedited approval workflow:
  - Only questions that changed are flagged for reviewer
  - Approval steps created for roles affected by changes (usually just contractmanager + financien)
  - Categorie-inkoper skips review unless category or company name changed fundamentally
  - No BIBOB re-assessment required unless risk tier changed

GIVEN all re-review approval steps are approved
WHEN the expedited workflow completes
THEN dossier → status: "goedgekeurd"
THEN vervaldatum += 12 months
THEN ERP is updated with new data (if changed)
THEN confirmation email sent to supplier

GIVEN a revalidation email is sent but the supplier does not respond within 60 days
WHEN the 60-day deadline passes
THEN daily job marks dossier → status: "verlopen"
THEN ERP supplier record is marked inactive (flagged in crediteurenstam)
THEN email reminder sent: "Your onboarding has expired. Contact us to reactivate."

GIVEN a supplier's dossier status is "verlopen"
WHEN they click "Reactivate" link in the reminder email
THEN a new OnboardingDossier is created (same leverancier, fresh dossier)
THEN supplier must complete full intake again (no expedited re-review)

---

## Error Handling & Validation

### Real-Time Validation Feedback

- **KvK**: On blur, call API; red border + error if not found; green checkmark if success.
- **IBAN**: On save, checksum check; red if invalid; yellow if name mismatch; green if both valid.
- **Peppol**: On blur, query SMP; red if inactive (but optional); green if active and documented.
- **Required fields**: Red asterisk in label; prevent form submission if blank.
- **External API failures**: If KvK-API, Surepay, or Peppol-SMP is down, user sees:
  - "Service temporarily unavailable. You can continue manually, but we'll validate this field during review."
  - Validatie-status = "handmatig" (manual)

### Approval Workflow Errors

- **Inconsistent state**: If a step is marked afgewerkt but next step not yet openstaand, error in workflow engine.
- **Timeout**: If an approval step is openstaand for >30 days, coordinator receives escalation email.
- **Rejection email fails**: Error logged; coordinator notified; supplier contact attempted manually.

---

## Performance & Load Considerations

- **KvK-API calls**: Rate limit to 10 req/sec per municipality; cache responses for 24 hours.
- **Peppol-SMP queries**: Cached for 7 days (endpoints change infrequently).
- **Approval dashboard**: Paginate dossier list (25 per page); lazy-load document previews.
- **Annual revalidation job**: Run off-peak (2am–6am); process 1,000 dossiers per minute; retry failed emails.

---

## Security & Data Protection

- **Public upload endpoint**: No auth required for initial questionnaire submission; CSRF token required.
- **Document storage**: Docudesk handles encryption at rest; 7-10 year retention per compliance.
- **Approval decisions**: Audited; only assigned reviewer can approve; timestamp & IP logged.
- **Data export**: Coordinator can export dossier list (CSV) with de-identified supplier names (GDPR).

---

## Testability

- **Manual test scenarios** (cucumber/BDD):
  - Supplier completes intake for logistiek (all questions, all docs, submit).
  - Supplier completes intake for zorg with KvK miss; manual review initiated.
  - Supplier enters IBAN with name mismatch; approvers see flag; one approves anyway.
  - Dossier with risicoTier=hoog creates parallel BIBOB; integrity concludes afgewezen; dossier rejected.
  - Supplier with €75k turnover forgets MVI proof; form prevents submission.
  - Annual revalidation: supplier confirms unchanged (1-click); vervaldatum rolls forward.
  - Annual revalidation: supplier updates IBAN; expedited workflow; only financien reviews.

---

**Status**: Draft — Ready for task phase  
**Updated**: 2026-05-22
