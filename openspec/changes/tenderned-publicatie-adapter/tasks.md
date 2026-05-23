# TenderNed Publicatie Adapter — Implementation Tasks

**Status:** Task Breakdown  
**Version:** 1.0  
**Created:** 2026-05-23

## Phase 1: Foundation (OpenRegister Schemas & Seed Data)

### Task 1.1: Define OpenRegister Schemas
**ID:** T-TPA-1-1  
**Owner:** Backend Lead  
**Effort:** 8 hours  
**Blockers:** None  

**Description:**
Create schema definitions for all seven entities in `lib/Settings/purchaseq_register.json`:
- `tenderned_publicatie`
- `tenderned_bijlage`
- `tenderned_vraag_antwoord`
- `tenderned_inschrijving`
- `cpv_code`
- `nuts_code`
- `drempelbedrag_periode`

**Steps:**
- [ ] Read design.md schema sections (Schema Definitions)
- [ ] Create schema JSON for each entity with:
  - Required/optional fields
  - Type validation (string, integer, enum, date-time, decimal, boolean, array, object)
  - Descriptions per field
  - Enumerations for status/type fields
  - Pattern validation for codes (CPV: `^[0-9]{8}-[0-9]$`, NUTS: `^NL[0-9]{3}$`)
- [ ] Add `x-openregister` metadata per adr-001-data-layer.md rules
- [ ] Review schema MUST against context-brief.md requirements
- [ ] Submit PR; await schema review

**Acceptance Criteria:**
- All 7 schemas are defined in purchaseq_register.json
- No custom properties are used (schema.org vocabulary only)
- Required flags match context-brief.md specifications
- Enum values match requirement scenarios (e.g., publicatieType enum matches REQ-TPA-004 scenarios)

---

### Task 1.2: Generate Seed Data and Import Configuration
**ID:** T-TPA-1-2  
**Owner:** QA / Data Setup  
**Effort:** 6 hours  
**Blockers:** T-TPA-1-1 (schemas must be defined first)  

**Description:**
Create seed data for testing and QA in `lib/Settings/purchaseq_register.json` under `components.objects[]`:
- 3-5 realistic `tenderned_publicatie` objects with various statuses
- 2-3 `tenderned_bijlage` objects (complete and failed uploads)
- 2-3 `tenderned_vraag_antwoord` objects
- 5+ `cpv_code` examples (hierarchy from level 1 to 3)
- 3+ `nuts_code` examples (NUTS-1, NUTS-2, NUTS-3)
- 1 `drempelbedrag_periode` object (2024-2025 threshold values)

**Steps:**
- [ ] Use design.md Seed Data section as template
- [ ] Create realistic Dutch values:
  - Valid KVK numbers (format: 8 digits, one leading zero for gemeente)
  - Valid NUTS codes (NL-prefixed, existing codes only)
  - Valid CPV codes from reference registry (with correct check digits)
  - Valid postal codes (format `[1-9][0-9]{3}[A-Z]{2}`, e.g., "1012 AB")
- [ ] Create diverse scenarios:
  - One concept publicatie (not yet validated)
  - One gepubliceerd (fully successful flow)
  - One geweigerd (rejected by TenderNed)
  - One with validation errors
  - One with completed uploads
  - One with failed checksum
- [ ] Use `@self` envelope per adr-001-data-layer.md
- [ ] Test import idempotency (run import twice; no duplicates)
- [ ] Verify seed data loads on app install

**Acceptance Criteria:**
- Seed data is realistic and diverse
- All object properties match schema definitions
- Slugs are unique and human-readable
- Import is idempotent (no duplicates on re-import)
- Data loads automatically on app install

---

## Phase 2: Backend Services and APIs

### Task 2.1: TenderNed OpenConnector Source Configuration
**ID:** T-TPA-2-1  
**Owner:** Backend Lead  
**Effort:** 6 hours  
**Blockers:** None  

**Description:**
Configure the `tenderned` source in OpenConnector with OAuth2 client-credentials auth, base URLs for prod/pre-prod, and the retry/timeout envelope.

**Steps:**
- [ ] Read openconnector documentation for source setup
- [ ] Define `tenderned` source in openconnector config:
  - OAuth2 client-credentials endpoint
  - Client ID and secret (from PIANOo developer portal)
  - Base URLs: `https://api.tenderned.nl` (prod), `https://pre.tenderned.nl` (pre-prod)
  - Configurable per environment (settings.json override)
- [ ] Configure retry envelope:
  - Exponential backoff: 1s, 2s, 4s, 8s, 16s, 32s (max 6 retries for 5xx)
  - No retry for 4xx errors
  - Circuit breaker: open after 5 consecutive 5xx, half-open after 60s
  - Jitter: ±10% on backoff delays
- [ ] Configure timeout envelope:
  - Connection timeout: 10s
  - Read timeout: 30s
  - Max total time: 5 minutes
- [ ] Ensure CallLog integration:
  - Every request/response is logged with timestamp, method, URL, response code
  - Request body and response body are logged (sanitized for PII)
  - Associated with publicatie correlatieId
- [ ] Test source connectivity to TenderNed pre-prod endpoint

**Acceptance Criteria:**
- Source is registered in openconnector
- OAuth2 token refresh works
- Retry logic works (test by mocking 503 responses)
- Timeout logic works (test by mocking slow responses)
- CallLog records all requests/responses
- Config is environment-aware (prod/pre-prod switchable)

---

### Task 2.2: PublicatieGenerator Service
**ID:** T-TPA-2-2  
**Owner:** Backend Lead  
**Effort:** 12 hours  
**Blockers:** T-TPA-1-1 (schemas), T-TPA-2-1 (openconnector source)  

**Description:**
Implement PublicatieGenerator service that:
1. Takes an `aanbesteding` object and publicatieType as input
2. Determines `eformsNoticeSubtype` based on drempelbedrag and procedure
3. Maps aanbesteding fields to eForms BT-fields
4. Serializes to eForms XML per SDK 1.13

**Steps:**
- [ ] Create `PublicatieGeneratorService` class in `src/Service/TenderNed/`
- [ ] Implement `publicatieTypeMapping()` method:
  - Input: aanbesteding object, requested publicatieType
  - Output: determined publicatieType + eformsNoticeSubtype (per design.md mapping table)
  - Logic:
    - If concessie=true → publicatieType="concessie", subtype="25"
    - Else if procedure="open"/"niet_openbaar" AND value >= drempel → boven-drempel (EU subtype)
    - Else → below-threshold (nationale TN subtype)
    - Special cases: sociale_specifieke_diensten, speciale_sector
- [ ] Implement `mapToEformsFields()` method:
  - Input: aanbesteding object, drempelbedrag_periode
  - Output: array of BT-field → value mappings
  - Map aanbesteding.titel → BT-21 (in NL and EN)
  - Map aanbesteding.cpv_main_code → BT-262 with check-digit validation
  - Map aanbesteding.geraamde_waarde → BT-27
  - Map aanbesteding.procedure → BT-105 (with enum translation)
  - Map aanbesteding.gunningscriteria → BT-539 array
  - Map aanbesteding.percelen → BT-137 array (for multi-lot)
  - Map aanbesteding.looptijd_in_maanden → BT-36 (as duration)
  - Map aanbesteding.plaats_van_uitvoering → BT-727 (resolve to NUTS)
  - Resolve MVI/SROI fields (fetch from mvi-sroi-aanbesteding capability)
  - Resolve speciale-sector fields (if applicable)
- [ ] Implement `generateConcept()` method:
  - Creates `tenderned_publicatie` record with status="concept"
  - Calls ObjectService.saveObject() to persist in OpenRegister
  - Returns new publicatie object
- [ ] Implement eForms XML serialization:
  - Use eForms 2.0 SDK (assumes SDKVersion is configured in settings)
  - Build XML tree from BT-field mappings
  - Validate structure per SDK validation rules
  - Return as string in `eformsXml` field
- [ ] Add tests:
  - Test above-threshold works (subtype 16)
  - Test below-threshold nationale (subtype T01-T30)
  - Test concessie (subtype 25)
  - Test sociale specifieke diensten (subtype 27)
  - Test speciale-sector (subtype 18)
  - Test multi-lot (BT-137 array)
  - Test MVI/SROI field injection

**Acceptance Criteria:**
- PublicatieGenerator creates concept publicaties with correct subtypes
- BT-field mappings match design.md table
- eForms XML is valid per SDK 1.13
- All aanbesteding fields are correctly mapped
- Multi-lot and multi-language handling works
- Tests pass for all publicatieType scenarios

---

### Task 2.3: eForms Validation Service (XSD, Schematron, TN-Extension)
**ID:** T-TPA-2-3  
**Owner:** Backend Lead  
**Effort:** 14 hours  
**Blockers:** T-TPA-2-2 (publicatie generator creates XML)  

**Description:**
Implement ValidatorService that validates eForms XML against XSD, Schematron rules, and TenderNed-specific extension rules.

**Steps:**
- [ ] Create `ValidatorService` class in `src/Service/TenderNed/`
- [ ] Implement `validateXSD()` method:
  - Input: eForms XML string, eformsNoticeSubtype
  - Use Xerces XML parser (or similar PHP library)
  - Load XSD for subtype from eForms SDK
  - Validate XML structure against XSD
  - Collect all XSD validation errors with xpath
  - Return array of ValidationResult objects
- [ ] Implement `validateSchematron()` method:
  - Input: eForms XML string, eformsNoticeSubtype
  - Use Saxon-HE (via composer package) to run Schematron rules
  - Load Schematron rules from eForms SDK (rule_NOTICE_SUBTYPE.sch)
  - Execute rules against XML
  - Collect all rule violations (error/warning) with ruleId, message, xpath
  - Return array of ValidationResult objects
- [ ] Implement `validateTenderNedExtensions()` method:
  - Input: eForms XML, aanbesteding object
  - Check TenderNed-specific rules (TN-EXT-001, TN-EXT-002, etc.):
    - TN-EXT-001: Opdrachten >EUR 1.000.000 niet opgedeeld require motivering
    - TN-EXT-002: Nationale aankondigingen require NL language
    - TN-EXT-003: Concessions require specific sector fields
  - Compare against aanbesteding.motivering_niet_opdelen if applicable
  - Return array of TN-specific validation results
- [ ] Implement `validate()` orchestrator method:
  - Calls XSD validation
  - Calls Schematron validation
  - Calls TN-extension validation
  - Consolidates all results (errors first, then warnings)
  - Updates publicatie.validatieResultaten with consolidated array
  - Sets publicatie.status = "gevalideerd" if no errors, otherwise "concept"
- [ ] Implement `getValidationSchema()` method:
  - Returns simplified schema of all BR-* and TN-EXT-* rules
  - Used by UI to display error hints
- [ ] Add tests:
  - Test valid XML passes all validations
  - Test missing mandatory field (e.g., BT-21) triggers BR-OPT-13 error
  - Test invalid CPV check-digit triggers XSD error
  - Test multi-language validation (requires NL+EN for above-threshold)
  - Test TN-EXT-001 motivering check
  - Test nationale rule validations

**Acceptance Criteria:**
- Validator detects all mandatory field violations
- XSD validation finds structure errors (invalid check digits, missing required fields)
- Schematron validation finds business rule violations (BR-*)
- TN-extension validation finds NL-specific rules (TN-EXT-*)
- Validation results are comprehensive (all errors reported together)
- Tests pass for major validation scenarios

---

### Task 2.4: Document Upload Service with SHA-256 Verification
**ID:** T-TPA-2-4  
**Owner:** Backend Lead  
**Effort:** 10 hours  
**Blockers:** T-TPA-1-1 (schemas), T-TPA-2-1 (openconnector source)  

**Description:**
Implement DocumentUploadService that handles resumable uploads, chunking, progress reporting, and SHA-256 checksum verification.

**Steps:**
- [ ] Create `DocumentUploadService` class in `src/Service/TenderNed/`
- [ ] Implement `uploadDocument()` method:
  - Input: tenderned_bijlage object, file stream, upload session ID (if resuming)
  - Chunk file into 4 MB pieces
  - For each chunk:
    - Emit progress event (chunk/total)
    - Stream chunk to TenderNed via openconnector
    - Store upload session token returned by TenderNed
    - On 5xx error: retry with exponential backoff using session token
  - Compute SHA-256 incrementally as chunks stream (streaming hash)
  - Call finalization endpoint with computed SHA-256
  - Verify TenderNed-returned SHA-256 matches computed SHA-256
  - Update bijlage.uploadStatus, tendernedBijlageId, checksumSha256
  - Return success/failure status
- [ ] Implement resumable upload logic:
  - If upload session token exists, resume from last acknowledged byte
  - Use TenderNed's /documents/{documentId}/resume endpoint
  - Do not re-hash previously uploaded chunks; only hash remaining chunks
- [ ] Implement checksum verification:
  - If TenderNed-returned SHA-256 != computed SHA-256:
    - Mark bijlage.uploadStatus = "failed_checksum_mismatch"
    - Retry up to 3 times (re-upload entire file)
    - After 3 failures, set status = "failed"
    - Return error to caller
- [ ] Implement confidentiality flag handling:
  - If bijlage.vertrouwelijk = true:
    - Set confidentialityLevel = "RESTRICTED" in TenderNed request
    - Document is not exposed via public dossier URL
- [ ] Implement multi-language support:
  - Accept taal parameter (NL, EN, FR, DE)
  - Include in TenderNed request
- [ ] Add tests:
  - Test successful 150 MB upload with progress reporting
  - Test checksum mismatch detection and retry
  - Test resumable upload after 503 error
  - Test confidential document flag
  - Test multi-language document upload

**Acceptance Criteria:**
- Documents up to 200 MB can be uploaded successfully
- Progress is reported per chunk (0%, 3%, 6%, ..., 100%)
- SHA-256 is computed correctly end-to-end
- Checksum mismatch is detected and retried
- Resumable upload resumes from correct byte offset
- Confidential documents are not exposed publicly
- Tests pass for all upload scenarios

---

### Task 2.5: Publication Submission and Status Polling Service
**ID:** T-TPA-2-5  
**Owner:** Backend Lead  
**Effort:** 10 hours  
**Blockers:** T-TPA-2-3 (validation), T-TPA-2-4 (document upload), T-TPA-2-1 (openconnector)  

**Description:**
Implement SubmissionService that submits validated publicaties to TenderNed and polls for status.

**Steps:**
- [ ] Create `SubmissionService` class in `src/Service/TenderNed/`
- [ ] Implement `submitPublication()` method:
  - Input: tenderned_publicatie object
  - Check prerequisites:
    - status = "concept" or "gevalideerd"
    - validatieResultaten has no "error" level entries
    - All required bijlagen are uploaded
    - For above-threshold: goedkeuring is recorded (goedgekeurdDoor + goedkeuringsDatum)
  - POST to TenderNed /notices endpoint with eForms XML
  - On success (202 Accepted):
    - Capture correlationId
    - Update publicatie.status = "ingediend"
    - Update publicatie.correlatieId
    - Start polling job
    - Return success
  - On error (4xx):
    - Parse error details (e.g., INVALID_CPV_HIERARCHY)
    - Update publicatie.validatieResultaten with TenderNed error
    - Revert publicatie.status = "concept"
    - Return error details
  - On error (5xx):
    - Retry via openconnector (automatic via retry envelope)
- [ ] Implement `pollPublicationStatus()` method:
  - Input: tenderned_publicatie object with correlationId
  - Poll TenderNed GET /notices/{correlationId}/status every 30 seconds
  - Loop until:
    - status = "PUBLISHED" (success):
      - Extract publicationId (tendernedPublicatieId)
      - Extract tedPublicationId (if boven-drempel)
      - Extract publicationDate (from TN, not local clock)
      - Extract dossierUrl
      - Update publicatie record:
        - status = "gepubliceerd"
        - tendernedPublicatieId, tedPublicatieId, publicatieDatum, tendernedDossierUrl
      - Update base aanbesteding.actievePublicatieId
      - Add entry to aanbesteding.publicatieHistorie[]
      - Send notification: "Publicatie gepubliceerd op TenderNed"
      - Return success
    - status = "REJECTED" (failure):
      - Extract errorCode and violations
      - Revert publicatie.status = "concept"
      - Update publicatie.validatieResultaten with TN-side violations
      - Send notification: "Publicatie geweigerd door TenderNed"
      - Return failure with error details
    - status = "PENDING" (still processing):
      - Continue polling (max 10 minutes; then timeout and return)
- [ ] Implement polling timeout logic:
  - After 10 minutes without final status, stop polling
  - Keep publicatie in "ingediend" state
  - Show UI message: "Publicatie ingediend, wacht nog op bevestiging"
  - Offer "Controleer status" button for manual check
  - If status stays "ingediend" for >24h, raise ops alert
- [ ] Add background job integration:
  - Queue publicatie submissions for async processing
  - Store polling job with publicatie ID and correlationId
  - Resume polling if app restarts during polling phase
- [ ] Add tests:
  - Test successful submission and publication flow
  - Test rejection due to invalid CPV
  - Test rejection due to missing mandatory field
  - Test polling timeout handling
  - Test nationale aankondiging (no tedPublicationId)
  - Test resumption after app restart

**Acceptance Criteria:**
- Publicaties are submitted successfully to TenderNed
- Status transitions from "ingediend" → "gepubliceerd"
- tendernedPublicatieId and tedPublicatieId are captured
- publicatieDatum is from TenderNed (not local clock)
- Rejections are handled gracefully with error details
- Polling timeout is respected (max 10 minutes)
- Notifications are sent for status changes
- Tests pass for all submission scenarios

---

### Task 2.6: Vragen Polling and Q&A Management Service
**ID:** T-TPA-2-6  
**Owner:** Backend Lead  
**Effort:** 8 hours  
**Blockers:** T-TPA-2-1 (openconnector), T-TPA-1-1 (schemas)  

**Description:**
Implement VragenPollService that polls TenderNed for questions during inlichtingenperiode, creates vraag_antwoord records, and generates nota van inlichtingen.

**Steps:**
- [ ] Create `VragenPollService` class in `src/Service/TenderNed/`
- [ ] Implement `pollVragen()` method (runs every 15 minutes via n8n):
  - Get all publicaties with status="gepubliceerd"
  - Filter to those within inlichtingenperiode (now < sluitingsDatum)
  - For each publicatie:
    - Call TenderNed GET /questions?since={lastPollTime}
    - For each new question:
      - Create tenderned_vraag_antwoord record with status="open"
      - Auto-increment vraagnummer per aanbesteding
      - Set vraagstelDatum from TenderNed response
      - Check if vraag is laat_ontvangen (< 6 days before sluitingsDatum)
      - If laat_ontvangen=true, set flag and log warning
      - Send notification to inkoper: "Nieuwe vraag #{vraagnummer}"
  - Log poll event with count of new vragen
- [ ] Implement `generateNotaVanInlichtingen()` method:
  - Input: publicatie object, array of vraag_antwoord IDs
  - Generate PDF with:
    - Titel van aanbesteding
    - Table of Q&A pairs (vraag text + antwoord text)
    - Publication date footer
    - Signature block (placeholder for inkoper)
  - Use a PDF library (e.g., TCPDF, DomPDF)
  - Save PDF as temporary file
  - Upload via DocumentUploadService (documentType = "nota_van_inlichtingen")
  - Create tenderned_bijlage record
  - Update vraag_antwoord.status = "verwerkt_in_nota" for all included vraag
  - Update vraag_antwoord.notaVanInlichtingenRef = document ref
- [ ] Implement rectificatie detection:
  - Compare generated NvI against original bestek/relevant documents
  - If antwoord materially changes procedure/requirements:
    - Automatically trigger rectificatie-aankondiging generation
    - User receives notification: "Materialele wijziging gedetecteerd. Rectificatie-aankondiging is voorbereid."
  - If no material change, generate NvI as standalone document
- [ ] Add n8n workflow integration:
  - Create workflow in n8n dashboard
  - Trigger: every 15 minutes (0 */15 * * * * *)
  - Steps:
    1. Call purchaseq API: GET /api/publicaties?status=gepubliceerd
    2. Filter to within inlichtingenperiode
    3. For each publicatie: call pollVragen(publicatieId)
    4. Log results
- [ ] Add tests:
  - Test poll creates vraag_antwoord records correctly
  - Test laat_ontvangen flag is set correctly
  - Test nota van inlichtingen PDF generation
  - Test rectificatie detection
  - Test multiple Q&A rounds

**Acceptance Criteria:**
- Vragen are polled successfully every 15 minutes
- vraag_antwoord records are created with correct fields
- laat_ontvangen flag is set for questions < 6 days before deadline
- Nota van inlichtingen PDF is generated correctly
- Documents are uploaded to TenderNed successfully
- Rectificatie detection triggers for material changes
- n8n workflow runs reliably
- Tests pass for all polling scenarios

---

## Phase 3: Frontend and UI Integration

### Task 3.1: Beheer > Connectors > TenderNed UI (Settings Page)
**ID:** T-TPA-3-1  
**Owner:** Frontend Lead  
**Effort:** 8 hours  
**Blockers:** T-TPA-1-1 (schemas)  

**Description:**
Create the settings page at Beheer > Connectors > TenderNed where inkopers can configure TenderNed connection and manage publicaties.

**Steps:**
- [ ] Create Vue component `BeheerTenderNedPage.vue` in `src/components/pages/`
- [ ] Implement sections:
  - **Connection Status:** Display current OAuth2 token status, last poll time for vragen
  - **Configuration:**
    - Environment selector (prod/pre-prod)
    - eForms SDK version (configurable, default 1.13)
    - Drempelbedrag period selector (loaded from reference data)
  - **Recent Publicaties:** Table of last 10 publicaties with status, publicatieDatum, dossierUrl link
  - **Action Buttons:**
    - "Publiceer nieuwe aanbesteding" → triggers publicatie generator
    - "Controleer status" → manual poll for pending publicaties
- [ ] Integrate with `CnDataTable` for publicaties list (per OpenRegister patterns)
- [ ] Add status badge styling (color-coded):
  - concept = grey
  - gevalideerd = blue
  - ingediend = yellow
  - gepubliceerd = green
  - ingetrokken = red
  - geweigerd = red
- [ ] Add publicatieDatum and sluitingsDatum columns with countdown (days remaining)
- [ ] Implement "Publiceer nieuwe" workflow:
  - Opens modal to select aanbesteding
  - Triggers PublicatieGenerator
  - Shows validation results inline
  - Lists required documents to upload
- [ ] Use NL Design System components per adr-010 (nldesign)
- [ ] Add help text for each configuration field
- [ ] Test responsiveness (mobile, tablet, desktop)

**Acceptance Criteria:**
- Settings page displays connection status correctly
- Publicaties list is sortable, filterable, paginated
- Configuration fields are editable
- "Publiceer nieuwe" workflow works end-to-end
- UI uses NL Design System components
- Mobile view is usable

---

### Task 3.2: Aanbesteding Detail: Publicatie Tab
**ID:** T-TPA-3-2  
**Owner:** Frontend Lead  
**Effort:** 10 hours  
**Blockers:** T-TPA-2-2 (publicatie generator), T-TPA-2-3 (validator)  

**Description:**
Add a "Publicatie" tab to the aanbesteding detail page where users can manage publication workflow.

**Steps:**
- [ ] Create Vue component `AanbestedingPublicatieTab.vue` in `src/components/tabs/`
- [ ] Implement sections:
  - **Publication Status:** Shows active publicatie (if any) with status badge and dossierUrl link
  - **Publication History:** Table of all prior publicaties (vooraankondiging, aankondiging_opdracht, gunning, wijzigingen, etc.)
  - **Concept Editor:**
    - Form to select publicatieType
    - Read-only display of mapped BT-fields (from aanbesteding)
    - Validation results inline (errors = red, warnings = yellow)
    - Deep links to source aanbesteding fields for validation errors
    - "Generate" button to create concept
  - **Document Upload:**
    - Drag-and-drop or file picker for bestek, leidraad, etc.
    - Progress bar per document (chunks, percentage)
    - Uploaded documents list with status badges
    - Delete/replace buttons
  - **Submission Workflow:**
    - "Validate" button (runs ValidatorService)
    - "Submit to TenderNed" button (requires validation pass + approval if above-threshold)
    - Polling status display while ingediend
    - Success/error messages
  - **Q&A Section (if inlichtingenperiode):**
    - List of incoming vragen (from polling)
    - Inline antwoord editing
    - "Publish Nota van Inlichtingen" button
    - Mark vragen as beantwoord/verwerkt
- [ ] Implement form state management using Pinia store (createObjectStore)
- [ ] Add validatie errors with deep links to source fields:
  - Error: "BT-21 title is missing"
  - Deep link: "Edit aanbesteding > Titel"
- [ ] Add approval gate logic:
  - Check mandaat threshold
  - If above-threshold: show "Awaiting approval from {mandaathouder}" message
  - Show approval notification
- [ ] Use NL Design System form components
- [ ] Add inline help for each section
- [ ] Test with all publicatieType scenarios

**Acceptance Criteria:**
- Publicatie tab displays all workflow sections correctly
- Concept generation works and shows validation results
- Document upload works with progress reporting
- Validation errors are clear with deep links to source
- Approval workflow works for above-threshold
- Polling status is displayed
- Q&A section appears only during inlichtingenperiode
- Tests pass for all workflow scenarios

---

### Task 3.3: Real-Time Status and Notification Integration
**ID:** T-TPA-3-3  
**Owner:** Frontend Lead  
**Effort:** 6 hours  
**Blockers:** T-TPA-2-5 (polling service), T-TPA-3-1, T-TPA-3-2  

**Description:**
Integrate real-time notifications and status updates into the UI.

**Steps:**
- [ ] Integrate with NotificationService:
  - "Nieuwe vraag #1 op uw aanbesteding"
  - "Publicatie gepubliceerd op TenderNed"
  - "Publicatie geweigerd door TenderNed"
  - "Awaiting approval from {mandaathouder}"
  - "Reconciliation drift detected: publicatie was withdrawn externally"
- [ ] Implement real-time status updates:
  - Use Nextcloud event system or WebSocket (if available)
  - Polling via periodic API calls (every 30 seconds for status changes)
  - Update UI immediately when status changes
- [ ] Add toast notifications for:
  - Validation successes/failures
  - Upload progress and completion
  - Submission acknowledgment (ingediend)
  - Publication confirmation (gepubliceerd)
  - Rejection with error details
- [ ] Add modal dialogs for:
  - Validation error summary (list of all errors with deep links)
  - Rejection details (error code, violation details)
  - Approval confirmation (show mandaathouder name, approval date)
- [ ] Test notification delivery and UI updates

**Acceptance Criteria:**
- Notifications are delivered promptly
- Status changes are reflected in UI immediately
- Error messages are clear and actionable
- Approval workflow shows correct mandaathouder
- All toast and modal messages follow NL Design System styling

---

## Phase 4: Background Jobs and Integrations

### Task 4.1: n8n Workflow: Vragen Polling (Every 15 Minutes)
**ID:** T-TPA-4-1  
**Owner:** Integrations Lead  
**Effort:** 4 hours  
**Blockers:** T-TPA-2-6 (vragen service)  

**Description:**
Set up n8n workflow for automated vragen polling.

**Steps:**
- [ ] Create n8n workflow in purchaseq workspace
- [ ] Configure trigger: Cron (0 */15 * * * * *)
- [ ] Workflow steps:
  1. Call purchaseq API: GET /api/v1/tenderned-publicaties?status=gepubliceerd
  2. Filter results to publicaties within inlichtingenperiode
  3. For each publicatie: call purchaseq API POST /api/v1/tenderned-publicaties/{id}/poll-vragen
  4. Log results (count of new vragen)
  5. Send error notification if any call fails
- [ ] Configure error handling:
  - Retry on 5xx (exponential backoff)
  - Log and continue on individual publicatie failures
  - Alert ops if >5 consecutive failures
- [ ] Test workflow:
  - Manually trigger; verify vragen are polled
  - Verify notifications are sent for new vragen
  - Check n8n execution logs

**Acceptance Criteria:**
- Workflow runs every 15 minutes reliably
- Vragen are polled successfully
- Errors are logged and notifications sent
- Execution logs are available for debugging

---

### Task 4.2: n8n Workflow: Daily Reconciliation (Nightly at 02:00 UTC)
**ID:** T-TPA-4-2  
**Owner:** Integrations Lead  
**Effort:** 4 hours  
**Blockers:** T-TPA-2-7 (reconciliation service — to be created)  

**Description:**
Set up n8n workflow for daily status reconciliation.

**Steps:**
- [ ] Create n8n workflow in purchaseq workspace
- [ ] Configure trigger: Cron (0 2 * * * *)
- [ ] Workflow steps:
  1. Call purchaseq API: GET /api/v1/tenderned-publicaties?status=IN(gepubliceerd,ingediend,ingetrokken)
  2. For each publicatie: call purchaseq API POST /api/v1/tenderned-publicaties/{id}/reconcile
  3. Check response for drift detections
  4. If drift detected:
     - Send notification to inkoper
     - If critical (legal position divergence): send alert to ops
  5. Log reconciliation summary
- [ ] Configure error handling:
  - If TenderNed unreachable: log warning, keep publicaties at last-known status, alert ops after 24h
  - If API error: retry once, log and continue
- [ ] Test workflow:
  - Manually trigger; verify status checks complete
  - Verify drift notifications are sent
  - Check logs

**Acceptance Criteria:**
- Workflow runs nightly reliably
- Status reconciliation completes successfully
- Drift is detected and corrected
- Ops is alerted for critical drift
- TenderNed unavailability is handled gracefully

---

## Phase 5: Backend Service Layer (Reconciliation)

### Task 2.7: Reconciliation Service
**ID:** T-TPA-2-7  
**Owner:** Backend Lead  
**Effort:** 6 hours  
**Blockers:** T-TPA-2-5 (submission service), T-TPA-1-1 (schemas)  

**Description:**
Implement ReconciliationService for daily status synchronization.

**Steps:**
- [ ] Create `ReconciliationService` class in `src/Service/TenderNed/`
- [ ] Implement `reconcilePublicatie()` method:
  - Input: tenderned_publicatie object
  - Fetch TenderNed status via openconnector GET /notices/{publicatieId}/status
  - Compare local status vs TN status
  - If drift detected:
    - local="gepubliceerd" AND TN="withdrawn":
      - Update status = "ingetrokken"
      - Log reason "external_withdrawal"
      - Send notification to inkoper
    - local="ingetrokken" AND TN="published":
      - Raise CRITICAL alert
      - Email to ops with details
      - Do NOT auto-correct
    - Other drifts: log and investigate
  - Return drift summary
- [ ] Implement TenderNed unreachability handling:
  - If API call fails (connection error, timeout):
    - Increment retry counter
    - After 24h of unreachability: stop polling, alert ops
    - Do NOT update publicatie status
- [ ] Implement `reconcileAll()` method:
  - Gets all publicaties with status in (gepubliceerd, ingediend, ingetrokken)
  - Calls reconcilePublicatie() for each
  - Logs summary: count of publicaties, count of drifts, count of errors
  - Returns summary object
- [ ] Add tests:
  - Test external withdrawal detection
  - Test legal position divergence detection
  - Test TenderNed unreachability handling
  - Test summary reporting

**Acceptance Criteria:**
- Reconciliation detects drift correctly
- Drift is auto-corrected (external_withdrawal case)
- Critical drift is flagged (legal position divergence)
- TenderNed unreachability is handled gracefully
- Tests pass for all scenarios

---

## Phase 6: Testing and QA

### Task 6.1: Unit Tests (Validation, Mapping, Checksum)
**ID:** T-TPA-6-1  
**Owner:** QA / Backend  
**Effort:** 12 hours  
**Blockers:** T-TPA-2-2, T-TPA-2-3, T-TPA-2-4  

**Description:**
Write unit tests for core services (PublicatieGenerator, ValidatorService, DocumentUploadService).

**Steps:**
- [ ] Create test directory: `tests/Unit/Service/TenderNed/`
- [ ] Test PublicatieGenerator:
  - Test above-threshold works subtype selection
  - Test below-threshold nationale subtype
  - Test concessie subtype
  - Test sociale specifieke diensten
  - Test speciale-sector logic
  - Test multi-lot BT-137 mapping
  - Test multi-language BT-21 mapping
  - Test MVI/SROI field injection
- [ ] Test ValidatorService:
  - Test XSD validation with valid XML
  - Test XSD validation with missing mandatory field
  - Test Schematron rule violations (BR-OPT-13, etc.)
  - Test TN-extension rules (motivering_niet_opdelen)
  - Test CPV check-digit validation
  - Test multi-language requirements
- [ ] Test DocumentUploadService:
  - Test successful upload with progress reporting
  - Test checksum computation
  - Test checksum mismatch detection
  - Test resumable upload after 503
  - Test confidential document flag
  - Test multi-language metadata
- [ ] Test edge cases:
  - Zero-byte files
  - Very large files (200 MB boundary)
  - Unicode filenames
  - Special characters in metadata
- [ ] Code coverage: aim for >90% coverage on core services
- [ ] Run tests in CI/CD pipeline

**Acceptance Criteria:**
- Unit tests pass for all scenarios
- Code coverage is >90% for core services
- Tests are documented with clear expectations
- CI/CD runs tests on PR submissions

---

### Task 6.2: Integration Tests (API Workflows)
**ID:** T-TPA-6-2  
**Owner:** QA / Backend  
**Effort:** 10 hours  
**Blockers:** T-TPA-2-5, T-TPA-2-6  

**Description:**
Write integration tests for end-to-end workflows (submission, polling, reconciliation).

**Steps:**
- [ ] Create test directory: `tests/Integration/TenderNed/`
- [ ] Test submission workflow:
  - Create aanbesteding → generate concept → validate → upload documents → submit → poll for status
  - Verify status transitions correctly
  - Verify tendernedPublicatieId and tedPublicatieId are captured
  - Verify publicatieDatum is from TenderNed response
- [ ] Test vragen polling workflow:
  - Create published publicatie → poll for vragen → create vraag_antwoord records
  - Verify vraagnummer is auto-incremented
  - Verify laat_ontvangen flag is set correctly
- [ ] Test nota van inlichtingen workflow:
  - Create vraag_antwoord records → generate NvI PDF → upload → mark as verwerkt
  - Verify PDF is generated correctly
  - Verify document upload succeeds
- [ ] Test reconciliation workflow:
  - Create publicatie with local status → simulate TN status change → run reconciliation
  - Verify drift is detected
  - Verify status is auto-corrected
- [ ] Use mocked TenderNed API responses (OpenAPI mock library or similar)
- [ ] Test error scenarios:
  - TenderNed returns 4xx (validation error)
  - TenderNed returns 5xx (retry logic)
  - TenderNed timeout
  - Network error during upload
- [ ] Code coverage: aim for >80% coverage on integration workflows

**Acceptance Criteria:**
- Integration tests pass for all major workflows
- Error scenarios are handled gracefully
- Tests use realistic mocked TenderNed responses
- Code coverage is >80% on integration workflows

---

### Task 6.3: Manual QA: End-to-End User Workflows
**ID:** T-TPA-6-3  
**Owner:** QA / Testers  
**Effort:** 16 hours  
**Blockers:** T-TPA-3-1, T-TPA-3-2, T-TPA-3-3  

**Description:**
Manual testing of all end-to-end user workflows in a staging environment.

**Steps:**
- [ ] Test Workflow 1: Publish Open Procurement (Above-Threshold)
  - Create aanbesteding with titulo, CPV, value EUR 7M, procedure "open"
  - Click "Publiceer" → select type → view mapped fields → validate
  - Upload documents (bestek, leidraad) → verify progress bar
  - Click "Indienen bij TenderNed" → check status polling
  - Verify publication on TenderNed test environment
  - Verify deeplink to dossier works
- [ ] Test Workflow 2: Answer Inlichtingen
  - Go to published aanbesteding → check for new vragen
  - Add antwoord → check for material changes → publish NvI
  - Verify NvI PDF is generated correctly
  - Verify nota is uploaded to TenderNed
- [ ] Test Workflow 3: Award Notice
  - Add contract to aanbesteding → click "Genereer gunningsaankondiging"
  - Verify pre-filled fields (winner name, value, etc.)
  - Submit → verify publication
- [ ] Test Workflow 4: Error Handling
  - Attempt to publish with missing mandatory field
  - Verify error message and deep link to fix
  - Attempt to upload invalid document
  - Verify checksum mismatch is detected
  - Attempt submission without approval (if above-threshold)
  - Verify approval gate blocks submission
- [ ] Test Workflow 5: Status Reconciliation
  - Publish notice → manually withdraw on TenderNed test → run reconciliation
  - Verify drift is detected and local status is corrected
- [ ] Test Workflow 6: Mobile/Responsive UI
  - Test all workflows on mobile device
  - Verify forms are usable
  - Verify progress bars display correctly
- [ ] Test Accessibility (WCAG 2.1 AA):
  - Use keyboard navigation for all workflows
  - Check form labels and error messages for screen readers
  - Verify color contrast ratios
- [ ] Document test cases and results

**Acceptance Criteria:**
- All end-to-end workflows work without errors
- Error handling is user-friendly and clear
- UI is responsive on mobile
- Accessibility standards are met
- Test cases are documented for regression testing

---

### Task 6.4: Performance and Load Testing
**ID:** T-TPA-6-4  
**Owner:** QA / Performance Team  
**Effort:** 8 hours  
**Blockers:** T-TPA-2-4 (document upload)  

**Description:**
Test performance and load characteristics.

**Steps:**
- [ ] Performance test: Large document upload
  - Upload 200 MB file (TenderNed hard limit)
  - Measure time to completion
  - Verify checksum computation doesn't bottleneck
  - Measure memory usage during chunked upload
  - Goal: complete within 5 minutes, <500 MB memory peak
- [ ] Load test: Concurrent publicatie submissions
  - Submit 10 publicaties concurrently
  - Measure submission time and error rate
  - Goal: zero errors, <30s per submission
- [ ] Load test: vragen polling (100 publicaties)
  - Poll 100 concurrent publicaties for vragen
  - Measure total time
  - Goal: complete within 2 minutes
- [ ] Load test: reconciliation (1000 publicaties)
  - Reconcile 1000 publicaties for status drift
  - Measure time and memory usage
  - Goal: complete within 10 minutes
- [ ] Use Apache JMeter or similar load testing tool
- [ ] Document results and bottlenecks
- [ ] Recommend optimization if needed

**Acceptance Criteria:**
- Large document uploads complete within performance targets
- Concurrent submissions don't interfere with each other
- Polling and reconciliation complete within time targets
- Memory usage is acceptable
- No errors under load

---

## Phase 7: Documentation and Deployment

### Task 7.1: API Documentation and OpenAPI Spec
**ID:** T-TPA-7-1  
**Owner:** Documentation Lead  
**Effort:** 4 hours  
**Blockers:** T-TPA-2-1 through T-TPA-2-7 (all services)  

**Description:**
Document all REST APIs and generate OpenAPI/Swagger spec.

**Steps:**
- [ ] Document endpoints:
  - POST /api/v1/tenderned-publicaties (create concept)
  - PATCH /api/v1/tenderned-publicaties/{id} (update publicatie)
  - POST /api/v1/tenderned-publicaties/{id}/validate (run validation)
  - POST /api/v1/tenderned-publicaties/{id}/upload-document (upload bijlage)
  - POST /api/v1/tenderned-publicaties/{id}/submit (submit to TenderNed)
  - GET /api/v1/tenderned-publicaties/{id}/status (check status)
  - GET /api/v1/tenderned-publicaties/{id}/poll-vragen (poll for questions)
  - POST /api/v1/tenderned-publicaties/{id}/reconcile (reconciliation)
- [ ] For each endpoint, document:
  - Method, path, query/body parameters
  - Request/response examples
  - Error codes and messages
  - Rate limits (if applicable)
  - Authentication requirements
- [ ] Generate OpenAPI 3.0 spec (YAML or JSON)
- [ ] Include spec in API documentation website
- [ ] Create Postman collection for testing

**Acceptance Criteria:**
- All endpoints are documented
- OpenAPI spec is valid and complete
- Request/response examples are realistic
- Error codes are documented

---

### Task 7.2: Deployment Guide and Configuration
**ID:** T-TPA-7-2  
**Owner:** DevOps / Documentation  
**Effort:** 6 hours  
**Blockers:** All other tasks (implementation complete)  

**Description:**
Create deployment documentation and configuration guide.

**Steps:**
- [ ] Document deployment steps:
  - Prerequisites (PHP version, dependencies, eForms SDK)
  - Installation (composer, database migrations, seed data)
  - Configuration (settings.json for TenderNed OAuth2, environment selection)
  - n8n workflow installation
  - OpenConnector source setup
- [ ] Document environment configuration:
  - Production vs pre-prod TenderNed endpoints
  - OAuth2 credentials setup
  - eForms SDK version pinning
  - Drempelbedrag period configuration
  - Mandaat threshold configuration
- [ ] Document backup and recovery:
  - Audit trail retention (7 years)
  - Backup schedule for eForms XML payloads
  - Disaster recovery procedures
- [ ] Create troubleshooting guide:
  - Common errors and solutions
  - TenderNed connectivity issues
  - Document upload failures
  - Status polling timeouts
- [ ] Create operator runbook:
  - Daily monitoring tasks
  - Reconciliation troubleshooting
  - Emergency withdrawal procedures
  - Contact info for PIANOo support

**Acceptance Criteria:**
- Deployment guide is complete and tested
- Configuration is documented with examples
- Troubleshooting guide covers common issues
- Runbook is available for ops team

---

### Task 7.3: Release Notes and Migration Guide
**ID:** T-TPA-7-3  
**Owner:** Product / Documentation  
**Effort:** 4 hours  
**Blockers:** All other tasks (implementation complete)  

**Description:**
Create release notes and migration guide for operators.

**Steps:**
- [ ] Create release notes:
  - Feature summary
  - List of implemented requirements (REQ-TPA-001 through REQ-TPA-010)
  - Known limitations and future work
  - Breaking changes (if any)
  - Upgrade path from prior version (if any)
- [ ] Create migration guide:
  - How to migrate existing manual publication processes to this adapter
  - Data import steps (if existing publications need to be recorded)
  - Approval workflow setup
  - User training recommendations
- [ ] Create FAQ document:
  - Common questions about functionality
  - Regulatory compliance questions
  - Troubleshooting questions

**Acceptance Criteria:**
- Release notes clearly describe new capability
- Migration guide helps operators adopt the adapter
- FAQ answers common user questions

---

## Deduplication Check

**Task ID:** T-TPA-DEDUP  
**Owner:** Architecture Review  
**Effort:** 2 hours  

**Description:**
Verify that this capability does not duplicate existing OpenRegister or openconnector functionality.

**Steps:**
- [ ] Check OpenRegister service library for existing:
  - Document upload services (compare with FileService)
  - Audit trail services (compare with AuditTrailService)
  - Status transition tracking (compare with workflow engine)
  - Object CRUD (compare with ObjectService)
  - Approval workflow (compare with AuthorizationService)
- [ ] Check openconnector for existing:
  - HTTP source definitions
  - Retry envelope implementations
  - CallLog audit trail
  - OAuth2 client-credentials patterns
- [ ] Check purchaseq for existing:
  - aanbesteding-werkproces (owns drempelbedrag and procedure logic)
  - mvi-sroi-aanbesteding (owns MVI/SROI criteria)
  - docudesk (owns document generation)
- [ ] Verify no overlap; document reuse of existing services
- [ ] Report findings in design.md "Reuse Analysis" section

**Acceptance Criteria:**
- No duplicate capability is being built
- Existing services are reused appropriately
- Dependencies are documented

---

## Summary

**Total Effort:** ~160-170 hours  
**Phases:** 7 (Foundation → Design → Frontend → Background Jobs → Testing → Documentation → Deployment)  
**Critical Path:** Schema definition → Backend services → Frontend UI → n8n workflows → Testing → Deployment  

**Key Risks:**
- eForms SDK complexity and version management
- TenderNed API contract changes (mitigated by PIANOo support)
- Large file upload reliability (mitigated by resumable sessions and checksum verification)
- Reconciliation drift scenarios (mitigated by comprehensive logging and alerts)

**Success Criteria:**
- All 10 requirements (REQ-TPA-001 to REQ-TPA-010) are implemented
- Unit and integration tests pass
- Manual QA workflows succeed
- Performance targets are met
- Deployment guide is complete
- Operator training is provided

---

**Next Steps:** Begin Phase 1 (Schema definition). Assign tasks to team members and track progress via this task list.
