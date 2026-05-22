# Tasks: Supplier Onboarding — Vragenlijst & Workflow

## Task Phase — Implementation Checklist

> All tasks are implementation steps. Each [ ] checkbox must be marked complete (✓) before moving to the next task. Dependencies are listed where applicable.

---

## Data Layer & Schemas (OpenRegister)

### [ ] Task 1: Define Leverancier Schema

**Description**: Create Leverancier (supplier) schema in OpenRegister with all required fields per ADR-001 data model.

**Implementation Steps**:
1. Create `lib/Settings/purchaseq_register.json` with `x-openregister.type: "application"` flag.
2. Define `Leverancier` schema (PascalCase, schema.org vocabulary):
   - Properties: handelsnaam (string), statutaireNaam (string), kvkNummer (string), btwNummer (string, pattern for NL prefix), peppolId (string), rechtsvorm (enum), vestigingsadres (object with street, number, postcode, plaats, land), correspondentieadres (same), iban (string, pattern IBAN), ibanTnv (string), contactpersoon (string), email (string, format email), telefoon (string, format phone).
   - Mark kvkNummer, btwNummer, iban, email as required.
   - Add description fields per schema.org conventions.
3. Register under purchaseq register with unique slug identifier.
4. Validate schema against JSON Schema draft 2020-12.

**Dependencies**: None

---

### [ ] Task 2: Define OnboardingDossier Schema

**Description**: Create OnboardingDossier schema for supplier intake lifecycle.

**Implementation Steps**:
1. Add `OnboardingDossier` schema to purchaseq_register.json.
2. Properties:
   - leverancier (relation: {register: "purchaseq", schema: "Leverancier", objectId: uuid})
   - categorie (enum: logistiek, zorg, bouw, ict, dienstverlening, ...)
   - risicoTier (enum: laag, midden, hoog)
   - status (enum: uitnodiging, ingevuld, intern_review, goedgekeurd, afgewezen, verlopen)
   - uitnodigingsDatum (date)
   - gereeddatum (date, nullable)
   - vervaldatum (date, nullable)
   - toelichting (string, nullable)
3. Mark leverancier, categorie, risicoTier, status as required.
4. Add createdAt, updatedAt auto-timestamps.

**Dependencies**: Task 1

---

### [ ] Task 3: Define VragenlijstAntwoord Schema

**Description**: Create questionnaire response schema.

**Implementation Steps**:
1. Add `VragenlijstAntwoord` schema to purchaseq_register.json.
2. Properties:
   - dossier (relation to OnboardingDossier)
   - vraagCode (string, e.g., "LOG-001-transport-license")
   - vraagTitel (string)
   - antwoord (string | boolean | object, depends on question type)
   - antwoordType (enum: string, bool, select, file-ref)
   - validatieStatus (enum: ok, te_controleren, fout)
   - validatieBron (enum: KvK-API, VIES, Surepay, Peppol-SMP, handmatig)
   - validatieBericht (string, nullable, error message if validatieStatus=fout)
3. Mark dossier, vraagCode, antwoord as required.
4. Add createdAt, updatedAt.

**Dependencies**: Task 2

---

### [ ] Task 4: Define Bewijsstuk Schema

**Description**: Create proof document schema.

**Implementation Steps**:
1. Add `Bewijsstuk` schema to purchaseq_register.json.
2. Properties:
   - dossier (relation to OnboardingDossier)
   - type (enum: kvk-uittreksel, iban-bewijs, vog, var, mvi-cert, iso-cert, sbr-uittreksel)
   - bestand (relation to docudesk Document)
   - geldigTot (date, nullable)
   - geverifieerd (boolean)
   - geverifieerd_door (string, email, nullable)
   - geverifieerd_op (date, nullable)
3. Mark dossier, type, bestand as required.

**Dependencies**: Task 2

---

### [ ] Task 5: Define BIBOBToets Schema

**Description**: Create BIBOB integrity assessment schema.

**Implementation Steps**:
1. Add `BIBOBToets` schema to purchaseq_register.json.
2. Properties:
   - dossier (relation to OnboardingDossier)
   - aanleiding (string, e.g., "Leverancier in categorie bouw (risico-hoog)")
   - status (enum: openstaand, in_behandeling, afgewerkt)
   - behandelaar (string, email)
   - conclusie (string, nullable; enum when set: ok, met_voorbehoud, afgewezen)
   - vastgelegd_op (date)
   - opmerkingen (string, nullable)
3. Mark dossier, aanleiding as required; mark status, behandelaar, vastgelegd_op as required on create.

**Dependencies**: Task 2

---

### [ ] Task 6: Define ApprovalStep Schema

**Description**: Create role-based approval decision schema.

**Implementation Steps**:
1. Add `ApprovalStep` schema to purchaseq_register.json.
2. Properties:
   - dossier (relation to OnboardingDossier)
   - rol (enum: categorie-inkoper, contractmanager, financien, integriteit)
   - volgorde (integer, 1-4)
   - status (enum: openstaand, afgewerkt, afgewerkt_met_vraag, afgewerkt_afgewezen)
   - beslisser (string, email, nullable until decision made)
   - beslissing (enum: approve, request-info, reject, null)
   - motivatie (string, required if decision made)
   - datum (date, nullable until decision)
   - notificatie_verstuurd (boolean, default false)
3. Mark dossier, rol, volgorde as required.

**Dependencies**: Task 2

---

### [ ] Task 7: Load Seed Data

**Description**: Populate purchaseq_register.json with 3-5 realistic supplier objects per entity.

**Implementation Steps**:
1. Add `components.objects[]` array to purchaseq_register.json with `@self` envelope.
2. Create 5 Leverancier seed objects (see design.md Seed Data section):
   - ACME Logistics BV (logistiek, laag)
   - De Zorggroep Utrecht (zorg, hoog)
   - BuildRight Projecten (bouw, hoog)
   - Digitale Diensten NL (ict, midden)
   - Groen Energie Co-op (dienstverlening, laag)
3. Create 5 OnboardingDossier seed objects, one per Leverancier, with varied statuses.
4. Create 10 VragenlijstAntwoord seed objects (2-3 per dossier, mix of validationStatuses).
5. Create 5 Bewijsstuk seed objects (1-2 per dossier).
6. Use valid Dutch postcodes, real street names, valid KvK/BTW numbers or obviously fictional ones.
7. Ensure `slug` fields are unique and human-readable (e.g., "lev-acme-logistiek", "dos-acme-2026-05").

**Dependencies**: Tasks 1-6

---

## Backend Services

### [ ] Task 8: Create QuestionnaireService

**Description**: Implement logic for category-driven questionnaire branching and question filtering.

**Implementation Steps**:
1. Create `lib/Service/QuestionnaireService.php` (3-layer: Controller → Service → Mapper).
2. Implement `loadQuestionnaire(string $categorie, string $risicoTier): array`:
   - Load question catalog from config (or IAppConfig).
   - Filter questions by categorie (match question tags).
   - Filter by risicoTier (e.g., questions tagged "risico=hoog" only appear if tier=hoog).
   - Return: {questions: [{code, title, type, required, help_text, ...}], documents: [{type, required, ...}]}.
3. Implement `determinRisicoTier(string $categorie, float $annualTurnover): string`:
   - Lookup categoria from config to base risk (e.g., logistiek=laag, zorg=hoog, bouw=midden).
   - If turnover > €50k: escalate to hoog (MVI required).
   - Return "laag" | "midden" | "hoog".
4. Implement `validateQuestion(string $antwoord, array $questionDef): array`:
   - Type checking (bool, string, enum, etc.).
   - Required field validation.
   - Return {valid: true|false, errors: [...]}.
5. Add `@spec openspec/changes/supplier-onboarding-vragenlijst/tasks.md#task-8` PHPDoc tags.

**Dependencies**: Task 2

---

### [ ] Task 9: Create ValidationService

**Description**: Implement real-time validation against external APIs (KvK, VIES, Surepay, Peppol).

**Implementation Steps**:
1. Create `lib/Service/ValidationService.php`.
2. Implement `validateKvK(string $kvkNumber): array`:
   - Call openconnector KvK-API adapter.
   - Return {valid: true|false, handelsnaam, rechtsvorm, vestigingsadres, sbi_code, error?}.
   - Cache results (24 hours, keyed by kvkNumber).
   - If API down, return {valid: null, error: "Service unavailable. Manual review required."}.
3. Implement `validateIBAN(string $iban, ?string $name): array`:
   - Validate IBAN checksum (mod-97).
   - If $name provided, call Surepay adapter.
   - Return {valid: true|false, checksumOk: true|false, nameMatch: ?true|false, error?}.
4. Implement `validateBTW(string $btwNummer): array`:
   - Call VIES adapter (openconnector).
   - Return {valid: true|false, error?}.
5. Implement `validatePeppolId(string $peppolId): array`:
   - Query Peppol-SMP directory.
   - Return {valid: true|false, endpointActive: true|false, supportedDocuments: [], error?}.
   - Cache results (7 days).
6. Add error handling: API failures logged, return "handmatig" validatieBron.
7. Add `@spec` PHPDoc tags.

**Dependencies**: Task 3

---

### [ ] Task 10: Create ApprovalService

**Description**: Implement role-based approval workflow and state machine.

**Implementation Steps**:
1. Create `lib/Service/ApprovalService.php`.
2. Implement `createApprovalSteps(OnboardingDossier $dossier): array`:
   - Determine risicoTier from dossier.
   - Create sequential ApprovalSteps: categorie-inkoper (volgorde=1) → contractmanager (2) → financien (3).
   - If risicoTier=hoog: add parallel integriteit step (volgorde=4, marked as parallel).
   - Save all ApprovalSteps with status=openstaand.
   - Return created steps.
3. Implement `processApprovalDecision(ApprovalStep $step, string $decision, string $motivatie): void`:
   - Validate decision: approve | request-info | reject.
   - Update ApprovalStep: status=afgewerkt, beslissing, motivatie, datum, beslisser.
   - If request-info: send email to supplier with request; revert dossier to ingevuld.
   - If approve: check if all prior steps (volgorde < current) are approved; if not, hold.
   - If reject: mark dossier → status: afgewezen; send rejection email; stop workflow.
   - If approve and all prior + parallel steps approved: chain to next volgorde step.
4. Implement `canTransitionToNextStep(ApprovalStep $step): bool`:
   - Check all prior sequential steps are afgewerkt with approval.
   - If parallel integriteit exists, check its conclusie (ok | met_voorbehoud allowed; afgewezen blocks).
   - Return true only if all conditions met.
5. Implement `finalizeApproval(OnboardingDossier $dossier): void`:
   - Called when all steps approve.
   - Dossier → status: goedgekeurd.
   - Set vervaldatum = today + 12 months.
   - Queue ERP sync job.
   - Send welcome email to supplier.
6. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 2, 6

---

### [ ] Task 11: Create BIBOBService

**Description**: Implement BIBOB risk assessment and integration with decidesk.

**Implementation Steps**:
1. Create `lib/Service/BIBOBService.php`.
2. Implement `createBIBOBToets(OnboardingDossier $dossier): BIBOBToets`:
   - Check if risicoTier=hoog and dossier.status transitioning to intern_review.
   - Create BIBOBToets record: status=openstaand, aanleiding set, behandelaar assigned to integriteitscoördinator role.
   - Assign task in decidesk (via openconnector adapter).
   - Return BIBOBToets.
3. Implement `updateBIBOBConclusion(BIBOBToets $toets, string $conclusion, string $opmerkingen): void`:
   - Validate conclusion: ok | met_voorbehoud | afgewezen.
   - Update BIBOBToets: conclusie, status=afgewerkt, opmerkingen.
   - Notify ApprovalService to check if parallel workflow can proceed.
   - If afgewezen: trigger dossier rejection.
4. Implement `getBIBOBRiskFactors(string $categorie): array`:
   - Load risk factor checklist from config (based on municipal BIBOB policy).
   - Return [factors] for integrity coordinator to assess.
5. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 2, 5, 10

---

### [ ] Task 12: Create RevalidationService

**Description**: Implement annual revalidation logic and email dispatch.

**Implementation Steps**:
1. Create `lib/Service/RevalidationService.php`.
2. Implement `checkExpiredDossiers(): array`:
   - Query all dossiers with status=goedgekeurd and vervaldatum < today.
   - Return list of dossiers needing revalidation.
3. Implement `sendRevalidationEmail(OnboardingDossier $dossier): void`:
   - Generate 1-click confirmation link (token-based, no auth required).
   - Generate edit link with login.
   - Send email to leverancier contact: "Your supplier profile expires on {vervaldatum}. Confirm or update."
   - Track email sent in audit trail.
4. Implement `processConfirmUnchanged(string $token): void`:
   - Validate token, identify dossier.
   - Dossier → status: goedgekeurd.
   - vervaldatum += 12 months.
   - Log audit entry: "Supplier confirmed unchanged at {timestamp}".
5. Implement `processRevalidationSubmission(OnboardingDossier $dossier, array $changes): void`:
   - Detect which fields/questions changed.
   - Re-validate only changed answers (KvK, IBAN, etc.).
   - Create expedited approval workflow:
     - Only affected roles (usually contractmanager + financien, skip categorie-inkoper unless category changed).
   - Process approval as in ApprovalService.
6. Implement `markExpiredDossiers(): void`:
   - Daily job: find dossiers in revalidation pending for >60 days with no response.
   - Dossier → status: verlopen.
   - Mark ERP supplier record inactive.
   - Send final notice email: "Your profile has expired. Please reactivate."
7. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 2, 10

---

### [ ] Task 13: Create ERPSyncService

**Description**: Implement push to ERP crediteurenstam after approval.

**Implementation Steps**:
1. Create `lib/Service/ERPSyncService.php`.
2. Implement `syncApprovedSupplier(OnboardingDossier $dossier): array`:
   - Extract Leverancier data from dossier.
   - Determine ERP type (AFAS, Unit4, Exact, SAP) from config.
   - Call openconnector adapter: POST /suppliers {name, kvk, btw, iban, ibanTnv, email, address}.
   - On success: record supplier_id in dossier metadata, log audit entry.
   - On failure: log error, queue for retry, notify coordinator.
   - Return {success: true|false, supplier_id?: string, error?: string}.
3. Implement `updateSupplierInERP(Leverancier $leverancier): array`:
   - Called if leverancier already exists in ERP; update address, IBAN, email, status.
   - Return {success, error?}.
4. Implement `markSupplierInactive(Leverancier $leverancier): void`:
   - Called when revalidation expires.
   - Call ERP adapter to mark supplier inactive in crediteurenstam.
5. Background job `erp-sync`: Poll for dossiers with status=goedgekeurd and not yet synced; batch process 100 at a time.
6. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 2, 7, 10

---

### [ ] Task 14: Create Category Configuration

**Description**: Define category, risk tier, and question mappings in config.

**Implementation Steps**:
1. Create `lib/Settings/categoriesConfig.json` (or add to IAppConfig):
   ```json
   {
     "categories": [
       {
         "slug": "logistiek",
         "label": "Logistiek & Transport",
         "baseRiskTier": "laag",
         "questions": ["LOG-001", "LOG-002", ...],
         "documentTypes": ["kvk-uittreksel", "iban-bewijs", "transport-license"],
         "bibobPolicy": false
       },
       {
         "slug": "zorg",
         "label": "Zorg",
         "baseRiskTier": "hoog",
         "questions": ["ZOR-001", "ZOR-002", ...],
         "documentTypes": ["kvk-uittreksel", "iban-bewijs", "big-registration", "vog"],
         "bibobPolicy": true
       },
       ...
     ],
     "questions": {
       "LOG-001-transport-license": {
         "title": "Heeft u een geldig vervoerlicense?",
         "type": "bool",
         "required": true,
         "categories": ["logistiek"],
         "riskTiers": ["laag", "midden", "hoog"]
       },
       ...
     },
     "turnoverThresholds": {
       "mvi_required": 50000,
       "large_contract": 25000
     }
   }
   ```
2. Load config in QuestionnaireService & ApprovalService.

**Dependencies**: Task 8

---

## Frontend Components

### [ ] Task 15: Create SupplierPortal.vue (Main Entry)

**Description**: Build supplier self-service portal entry component with category selector.

**Implementation Steps**:
1. Create `src/components/SupplierPortal.vue` (Options API, Pinia store).
2. Implement:
   - Check if user is authenticated (supplier or staff reviewing as supplier).
   - If new intake: show category selector grid (Logistics, Healthcare, Construction, etc.).
   - Each card shows category name, typical requirements, icon.
   - Click category → route to `/supplier-intake/:category`.
   - If continuing existing dossier: show dossier status & resume button.
3. Fetch categories from backend API GET /api/categories.
4. Use `@conduction/nextcloud-vue` components (NcButton, NcCard, etc.).
5. Translations: all strings via `t(appName, 'text')`.

**Dependencies**: Task 14

---

### [ ] Task 16: Create QuestionnaireForm.vue

**Description**: Build dynamic questionnaire form with validation feedback and branching logic.

**Implementation Steps**:
1. Create `src/components/QuestionnaireForm.vue`.
2. Props: `dossier` (OnboardingDossier), `categorie` (string).
3. Implement:
   - Fetch questionnaire via GET /api/questionnaires/{categorie}/{risicoTier}.
   - Render sections (Company Info, Certifications, MVI, etc.).
   - For each question:
     - Render input type (text, bool, select, date, etc.).
     - On blur (KvK, IBAN, Peppol): call validation endpoint.
     - Display validation-status badge (ok=green, fout=red, te_controleren=yellow).
     - Show validation-bericht if error.
   - Dynamically show/hide sections based on answered questions (e.g., "Are you in high-risk category?").
   - Progress bar: % questions answered, % docs uploaded.
   - MVI proof section: show as REQUIRED if annual turnover > €50k.
   - Save button (auto-save on blur for individual answers).
   - Submit button (only enabled if all required fields + docs present).
4. Use `CnAdvancedFormDialog` or schema-driven `CnFormDialog` for field generation.
5. Use Pinia store to persist dossier state.
6. Add error handling: API failures → "Service temporarily unavailable" message.

**Dependencies**: Tasks 8, 9

---

### [ ] Task 17: Create DocumentUploadSection.vue

**Description**: Build document upload UI with docudesk integration.

**Implementation Steps**:
1. Create `src/components/DocumentUploadSection.vue`.
2. Props: `dossier` (OnboardingDossier), `requiredDocTypes` (array).
3. Implement:
   - For each document type (KvK, IBAN proof, MVI, VOG, etc.):
     - Show required/optional badge.
     - Drag-drop zone or file input.
     - File preview (PDF thumbnail, image).
     - Upload progress bar.
     - On upload success: create Bewijsstuk record (call POST /api/bewijsstukken).
     - Display file meta: name, size, upload date, expiry date (if certificate).
     - Status badge: pending (unverified), verified (geverifieerd=true), expired.
   - Show help text per document type (e.g., "PSO certificate must be current").
   - Delete button to remove file (soft delete, keep audit).
4. File upload to FileService (openregister integration).
5. Handle docudesk archival link.

**Dependencies**: Tasks 4, 16

---

### [ ] Task 18: Create SupplierReviewPage.vue

**Description**: Build read-only review page summarizing questionnaire & docs before submit.

**Implementation Steps**:
1. Create `src/components/SupplierReviewPage.vue`.
2. Props: `dossierId` (uuid).
3. Implement:
   - Load dossier, all VragenlijstAntwoord, all Bewijsstuk.
   - Display questionnaire responses in read-only format.
   - Display document gallery with links.
   - Show validation summary (KvK: ok | fout, IBAN: ok | mismatch | fout, Peppol: ok | inactive).
   - Show validation-bron labels (KvK-API, Surepay, handmatig, etc.).
   - Highlight any validation-status=fout or te_controleren as yellow warnings.
   - "Submit" button → POST /api/dossiers/{id}/submit → dossier.status = intern_review.
   - "Back" button → return to QuestionnaireForm to edit.

**Dependencies**: Tasks 16, 17

---

### [ ] Task 19: Create ApprovalDashboard.vue

**Description**: Build internal approval queue for all roles.

**Implementation Steps**:
1. Create `src/components/ApprovalDashboard.vue`.
2. Implement:
   - Fetch dossiers with status=intern_review (or assigned to current user).
   - Table: Supplier name, Category, Risk tier, Current approver, Due date, Status.
   - Filters: status, role, category, overdue (radio/checkbox).
   - Sort: due date, category, supplier name.
   - Row click → route to `/dossier/{id}/approval`.
   - Use `CnDataTable` with pagination.
3. RBAC: Show only dossiers assigned to current user's role (categorie-inkoper, contractmanager, etc.).

**Dependencies**: Task 6

---

### [ ] Task 20: Create ApprovalDetail.vue

**Description**: Build role-based approval review page with decision form.

**Implementation Steps**:
1. Create `src/components/ApprovalDetail.vue`.
2. Props: `dossierId`, `approvalStepId`.
3. Implement:
   - Load dossier, approvalStep, all questionnaire responses, bewijsstukken.
   - Display summary (Supplier info, category, risk tier, docs).
   - Display questionnaire responses (read-only, highlighted per role):
     - categorie-inkoper: focus on sourcing fit questions (highlighted).
     - contractmanager: focus on MVI, terms, conditions.
     - financien: focus on IBAN, address, contact info.
     - integriteit: show BIBOB assessment form (separate task).
   - Validation summary (KvK, IBAN, Peppol).
   - Approval timeline (previous steps and their decisions).
   - Decision radio buttons: Approve | Request Info | Reject.
   - Motivatie textarea (required).
   - Submit button → POST /api/approval-steps/{id}/decide {decision, motivatie}.
   - On success: show success message, route back to ApprovalDashboard.
4. Show warnings (yellow banners):
   - "IBAN name mismatch – verify with supplier."
   - "KvK validation required manual review."
5. If request-info: pre-fill email template to supplier.

**Dependencies**: Tasks 6, 19

---

### [ ] Task 21: Create BIBOBPanel.vue

**Description**: Build BIBOB integrity assessment form for high-risk dossiers.

**Implementation Steps**:
1. Create `src/components/BIBOBPanel.vue`.
2. Props: `dossier`, `bibobToets`.
3. Implement:
   - Load risk factor checklist from config.
   - Display checklist: {factor, yes/no, notes} (e.g., "Ownership linked to sanctioned entities?", "Previous integrity violations?").
   - After user answers all factors: show conclusion options:
     - "OK" (groen)
     - "Conditional Approval" (orange, with caveats text)
     - "Rejected" (red, with reason)
   - Conclusion textarea.
   - Submit button → POST /api/bibob-toets/{id}/conclude {conclusie, opmerkingen}.
   - Link to decidesk for formal decision record (if applicable).
4. Styling: use NL Design System tokens.

**Dependencies**: Task 11

---

### [ ] Task 22: Create RevalidationEmail.vue (Server-Side Template)

**Description**: Build email template for annual revalidation prompt.

**Implementation Steps**:
1. Create `lib/Mail/RevalidationMail.php` (Nextcloud mail template).
2. Implement:
   - Subject: "Your supplier profile expires on {vervaldatum}. Confirm or update."
   - Body sections:
     - Greeting: "Dear {contactpersoon},"
     - Message: "Your supplier profile in our procurement system expires on {vervaldatum}. Please confirm or update your information."
     - 1-Click Confirm Link: button with token-based URL (no login). Text: "Confirm Without Changes".
     - Edit Link: button to re-login & edit. Text: "Update Information".
     - Deadline: "Please respond by {deadline = vervaldatum + 60 days}."
     - Signature.
3. Variables: {{leverancierNaam}}, {{vervaldatum}}, {{confirmUrl}}, {{editUrl}}, {{deadline}}.
4. Send via NotificationService or built-in mail.

**Dependencies**: Task 12

---

### [ ] Task 23: Create RevalidationPage.vue

**Description**: Build revalidation UI (1-click confirm or edit mode).

**Implementation Steps**:
1. Create `src/components/RevalidationPage.vue`.
2. Query params: `?token={revalidationToken}` (for 1-click confirm) or `?dossier={dossierId}` (for edit mode).
3. Implement:
   - **1-Click Confirm Path**:
     - Validate token, load dossier.
     - Show summary: "Your supplier profile for {leverancierNaam} has been confirmed unchanged."
     - Behind the scenes: POST /api/revalidation/confirm-token {token}.
     - On success: redirect to confirmation page "Thank you. Your profile is active until {newVervaldatum}."
   - **Edit Mode**:
     - Load dossier in edit mode (same as QuestionnaireForm but pre-filled).
     - Highlight changed fields (different styling).
     - On submit: detect changes, trigger expedited approval workflow.
     - Show message: "Your updates are under review. You'll be notified within 3 business days."

**Dependencies**: Tasks 12, 16

---

## API Controllers

### [ ] Task 24: Create QuestionnaireController

**Description**: Implement REST API for questionnaire loading and saving.

**Implementation Steps**:
1. Create `lib/Controller/QuestionnaireController.php`.
2. Routes:
   - `GET /api/questionnaires/{categorie}/{risicoTier}` → Load questionnaire questions & document types.
   - `POST /api/questions/validate` {field, value, question_type} → Call ValidationService, return {valid, message, validation_bron}.
   - `POST /api/dossiers/{id}/answers` {question_code, answer, answer_type} → Save VragenlijstAntwoord.
3. Auth: Public for self-service endpoints (mark `#[PublicPage]`); CSRF token required.
4. Response format: JSON with `total`, `page`, `pages` for lists.
5. Error handling: Appropriate HTTP status + `message` field.
6. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 8, 9, 3

---

### [ ] Task 25: Create DocumentController

**Description**: Implement REST API for document upload and retrieval.

**Implementation Steps**:
1. Create `lib/Controller/DocumentController.php`.
2. Routes:
   - `POST /api/dossiers/{id}/documents` {file, type} → Upload file, create Bewijsstuk, return file_id.
   - `GET /api/bewijsstukken/{id}` → Return Bewijsstuk metadata + docudesk link.
   - `DELETE /api/bewijsstukken/{id}` → Soft-delete Bewijsstuk (keep audit).
   - `GET /api/dossiers/{id}/documents` → List all Bewijsstukken for dossier.
3. File upload: Use FileService (openregister), delegate to docudesk for archival.
4. Response: `{file_id, type, upload_date, geldig_tot, status}`.
5. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 4, 17

---

### [ ] Task 26: Create DossierController

**Description**: Implement REST API for OnboardingDossier CRUD and submission.

**Implementation Steps**:
1. Create `lib/Controller/DossierController.php`.
2. Routes:
   - `POST /api/dossiers` {leverancier_data, categorie} → Create new OnboardingDossier, return dossier_id.
   - `GET /api/dossiers/{id}` → Load dossier + nested answers + documents.
   - `PUT /api/dossiers/{id}` {updates} → Update dossier fields.
   - `POST /api/dossiers/{id}/submit` → Validate completeness, transition to intern_review, create ApprovalSteps.
   - `GET /api/dossiers?status=intern_review&role={role}` → List dossiers for approval queue (paginated).
3. Auth: Public (with token) for suppliers; RBAC for internal approvers.
4. Response: Full dossier object with nested relations (leverancier, answers, documents).
5. Error handling: 400 if validation fails; 422 if business logic prevents state transition.
6. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 2, 10

---

### [ ] Task 27: Create ApprovalController

**Description**: Implement REST API for approval decisions and workflow progression.

**Implementation Steps**:
1. Create `lib/Controller/ApprovalController.php`.
2. Routes:
   - `GET /api/approval-steps/{id}` → Load single ApprovalStep.
   - `POST /api/approval-steps/{id}/decide` {decision, motivatie} → Process decision (approve | request-info | reject).
   - `GET /api/dossiers/{id}/approval-steps` → List all steps for dossier with status.
3. Auth: RBAC - only assigned role can POST decide.
4. Workflow orchestration:
   - On approve: check prior steps, chain to next step if ready.
   - On request-info: send supplier email, revert dossier to ingevuld.
   - On reject: mark dossier afgewezen, send rejection email.
   - On final approve: call ERPSyncService.
5. Response: Updated ApprovalStep, status change notifications.
6. Add `@spec` PHPDoc tags.

**Dependencies**: Tasks 10, 27

---

### [ ] Task 28: Create ValidationController

**Description**: Implement REST API for real-time field validation.

**Implementation Steps**:
1. Create `lib/Controller/ValidationController.php`.
2. Routes:
   - `POST /api/validate/kvk` {kvk_nummer} → Call ValidationService.validateKvK, return {valid, name, address, sbi, error?}.
   - `POST /api/validate/iban` {iban, name?} → Call ValidationService.validateIBAN, return {checksumOk, nameMatch?, error?}.
   - `POST /api/validate/btw` {btw_nummer} → Call ValidationService.validateBTW, return {valid, error?}.
   - `POST /api/validate/peppol` {peppol_id} → Call ValidationService.validatePeppolId, return {endpointActive, supportedDocuments, error?}.
3. Auth: Public (CSRF protected) for supplier portal.
4. Error handling: API failures → return {valid: null, error: "Service unavailable"}.
5. Add `@spec` PHPDoc tags.

**Dependencies**: Task 9

---

### [ ] Task 29: Create RevalidationController

**Description**: Implement REST API for annual revalidation flow.

**Implementation Steps**:
1. Create `lib/Controller/RevalidationController.php`.
2. Routes:
   - `POST /api/revalidation/confirm-token` {token} → Process 1-click confirm link.
   - `GET /api/revalidation/status/{dossierId}` → Check revalidation status & deadline.
   - `POST /api/dossiers/{id}/revalidate` {updates} → Submit revalidation changes, trigger expedited approval.
3. Auth: Token-based for 1-click (no login); RBAC for revalidation edits.
4. Response: {success, newVervaldatum?, next_steps?}.
5. Add `@spec` PHPDoc tags.

**Dependencies**: Task 12

---

## Background Jobs

### [ ] Task 30: Create DailyRevalidationCheckJob

**Description**: Implement background job to check for expired dossiers and send revalidation emails.

**Implementation Steps**:
1. Create `lib/BackgroundJob/DailyRevalidationCheckJob.php` (extends QueuedJob).
2. Schedule: Daily, 2am–6am off-peak.
3. Logic:
   - Call RevalidationService.checkExpiredDossiers() → list of dossiers with vervaldatum < today.
   - For each: send revalidation email (RevalidationMail).
   - Track sent_at timestamp in dossier metadata.
   - Log results: "Sent revalidation emails to N suppliers."
4. Error handling: Catch exceptions, log, continue to next.
5. Retry: If email fails, queue for retry (max 3 attempts).

**Dependencies**: Task 12

---

### [ ] Task 31: Create ExpireDossiersJob

**Description**: Implement background job to mark unresponded dossiers as expired.

**Implementation Steps**:
1. Create `lib/BackgroundJob/ExpireDossiersJob.php`.
2. Schedule: Daily, 6am.
3. Logic:
   - Find dossiers with revalidation email sent >60 days ago and no response (status still != updated).
   - Mark status: verlopen.
   - Call ERPSyncService.markSupplierInactive(leverancier).
   - Send final notice email: "Your profile has expired. Click to reactivate."
4. Logging: "Marked N dossiers as expired."

**Dependencies**: Tasks 12, 13

---

### [ ] Task 32: Create ERPSyncJob

**Description**: Implement background job to batch push approved suppliers to ERP.

**Implementation Steps**:
1. Create `lib/BackgroundJob/ERPSyncJob.php`.
2. Schedule: Every 30 minutes.
3. Logic:
   - Query dossiers with status=goedgekeurd and erp_synced=false (or missing).
   - Batch process up to 100 at a time.
   - For each: call ERPSyncService.syncApprovedSupplier().
   - Log success/failure per supplier.
   - On failure: queue for retry; notify coordinator if >3 failed attempts.
4. Monitoring: Track sync rate (suppliers/minute).

**Dependencies**: Task 13

---

## Testing

### [ ] Task 33: Create Feature Tests (BDD Scenarios)

**Description**: Write Gherkin scenarios for each REQ using Behat or PHPUnit.

**Implementation Steps**:
1. Create `features/supplier-onboarding.feature` (Gherkin):
   - Scenario: "Supplier completes logistiek intake (all questions, all docs, submit)"
   - Scenario: "KvK validation hit; pre-fills data"
   - Scenario: "KvK validation miss; flags for manual review"
   - Scenario: "IBAN name mismatch; warning shown; contractor can override"
   - Scenario: "Dossier with risk=hoog creates BIBOB; integrity routes in parallel"
   - Scenario: "MVI proof required if turnover >€50k; form prevents submit without"
   - Scenario: "Peppol validation; endpoint active logged; supported docs shown"
   - Scenario: "Approval workflow: categorie-inkoper → contractmanager → financien all approve"
   - Scenario: "Approval: request-info sent to supplier; dossier reverts to ingevuld"
   - Scenario: "Approval: reject ends workflow; rejection email sent"
   - Scenario: "Annual revalidation: supplier confirms unchanged; vervaldatum rolls +12 months"
   - Scenario: "Annual revalidation: supplier updates IBAN; expedited review only financien"
   - Scenario: "Dossier expires after 60 days no response; marked verlopen"
2. Implement step definitions in `features/bootstrap/SupplierOnboardingContext.php`.
3. Use DomCrawler, ApiTestCase, or browser test fixture (if UI-heavy).

**Dependencies**: All tasks

---

### [ ] Task 34: Create Unit Tests (Service Layer)

**Description**: Write PHPUnit tests for QuestionnaireService, ValidationService, ApprovalService, RevalidationService.

**Implementation Steps**:
1. Create test files:
   - `tests/Unit/Service/QuestionnaireServiceTest.php`
   - `tests/Unit/Service/ValidationServiceTest.php`
   - `tests/Unit/Service/ApprovalServiceTest.php`
   - `tests/Unit/Service/RevalidationServiceTest.php`
2. Test cases per service:
   - **QuestionnaireService**:
     - testLoadQuestionnaireFiltersByCategory
     - testLoadQuestionnaireFiltersByRiskTier
     - testDeterminRisicoTier (base category risk, turnover escalation)
   - **ValidationService**:
     - testValidateKvKHit (pre-fills data)
     - testValidateKvKMiss (flags error)
     - testValidateIBANChecksumPass
     - testValidateIBANChecksumFail
     - testValidateIBANNameMatch
     - testValidateIBANNameMismatch
     - testValidatePeppolIdActive
     - testValidatePeppolIdInactive
     - testValidationServiceDown (API failure → handmatig)
   - **ApprovalService**:
     - testCreateApprovalStepsSequential
     - testCreateApprovalStepsWithBIBOB (risicoTier=hoog)
     - testProcessApprovalDecisionApprove
     - testProcessApprovalDecisionRequestInfo (supplier email sent, dossier reverted)
     - testProcessApprovalDecisionReject
     - testCanTransitionToNextStepWaitsPrior
     - testFinalizeApprovalChainToERP
   - **RevalidationService**:
     - testCheckExpiredDossiers
     - testSendRevalidationEmail
     - testProcessConfirmUnchanged
     - testProcessRevalidationSubmissionExpedited
     - testMarkExpiredDossiers (60 day deadline)
3. Use mocks for external adapters (openconnector).

**Dependencies**: Tasks 8–13

---

### [ ] Task 35: Create Integration Tests (Database + Services)

**Description**: Write integration tests using OpenRegister database and full service stack.

**Implementation Steps**:
1. Create `tests/Integration/SupplierOnboardingIntegrationTest.php`.
2. Seed real (test) data to database (5 suppliers, 5 dossiers from Task 7).
3. Test scenarios:
   - End-to-end supplier intake → approval → ERP sync.
   - Annual revalidation workflow.
   - Rejection & re-submission.
   - BIBOB integration (dossier waits for integrity conclusion).
4. Use `PHPUnit\Framework\TestCase` + database rollback after each test.

**Dependencies**: Tasks 1–32

---

### [ ] Task 36: Create Browser Tests (UI E2E)

**Description**: Write automated browser tests using Nextcloud test browser fixture or Playwright.

**Implementation Steps**:
1. Create `tests/Browser/SupplierIntakeTest.php` or `tests/e2e/supplier-intake.spec.js`.
2. Test flows:
   - Supplier fills logistiek questionnaire (all questions, all docs).
   - KvK blur triggers validation; data pre-filled.
   - IBAN blur triggers validation; name mismatch warning shown.
   - Submit button disabled until all required fields + docs done.
   - Submit → dossier page with review summary.
   - Review & submit → dossier disappears from supplier portal.
   - Coordinator sees dossier in approval queue.
   - Coordinator approves → email to next role.
   - Final approval → dossier disappears from queue.
3. Use `test-api` or `test-app` skills to automate (or test-persona for accessibility).

**Dependencies**: Tasks 15–23

---

## Documentation & Compliance

### [ ] Task 37: Add PHPDoc @spec Tags

**Description**: Ensure all public classes and methods have `@spec` PHPDoc tags linking to this change.

**Implementation Steps**:
1. Add file-level `@spec` tag in header docblock of every new PHP file:
   ```php
   /**
    * @spec openspec/changes/supplier-onboarding-vragenlijst/tasks.md#task-N
    */
   ```
2. Add method-level `@spec` tags for all public methods (Classes in Tasks 8–13, 24–29).
3. Verify: grep for all public methods; none missing `@spec`.
4. Link format: `openspec/changes/{change-name}/tasks.md#{anchor}`.

**Dependencies**: All tasks

---

### [ ] Task 38: Verify Deduplication Check

**Description**: Ensure no overlap with existing OpenRegister services or custom code.

**Implementation Steps**:
1. Search `openregister/lib/Service/` for ObjectService, RegisterService, SchemaService, ConfigurationService, etc.
2. Verify:
   - CRUD for Leverancier, OnboardingDossier, etc. → leverage ObjectService (built-in).
   - File upload → leverage FileService (built-in).
   - Audit trail → leverage AuditTrailService (built-in).
   - Task creation → leverage TasksController (built-in).
   - List/search → leverage CnIndexPage + CnDataTable (built-in).
3. Custom code only for:
   - QuestionnaireService (category branching logic).
   - ValidationService (external API calls).
   - ApprovalService (workflow state machine).
   - BIBOBService (BIBOB routing).
   - RevalidationService (annual revalidation logic).
   - ERPSyncService (ERP push).
4. Document findings in a DEDUPLICATION.md file (even if "no overlap found").

**Dependencies**: All tasks

---

### [ ] Task 39: Validate Against Standards

**Description**: Ensure data model and APIs conform to required standards.

**Implementation Steps**:
1. **NEN-EN-IEC 16931** (Peppol BIS): Verify e-invoice format (not core to supplier onboarding, but referenced for future).
2. **Wet BIBOB**: Verify BIBOB workflow (REQ-004) matches municipal policy structure.
3. **MVI criteria PIANOo**: Verify MVI proof acceptance (PSO, CO2 ladder, self-declaration) per spec.
4. **KvK API & VIES**: Verify openconnector adapters support required fields.
5. **ISO 20022**: Verify IBAN validation logic per spec.
6. Document compliance in STANDARDS.md.

**Dependencies**: All tasks

---

### [ ] Task 40: Create Migration (if needed)

**Description**: If OpenRegister schema is new, create repair step for first install.

**Implementation Steps**:
1. Create `lib/Migration/RegisterMigration.php` (implements IRepairStep).
2. Logic:
   - Check if purchaseq register exists.
   - If not: call ConfigurationService::importFromApp('purchaseq', purchaseq_register.json, version, force=false).
   - Load seed data via same pipeline.
3. Register in `appinfo/info.xml` with `<repair-steps>`.
4. Test: Install app fresh; verify schemas & seed data load.

**Dependencies**: Task 7

---

## Cleanup & Handoff

### [ ] Task 41: Update App Version & Changelog

**Description**: Bump version, update CHANGELOG.md, prepare release notes.

**Implementation Steps**:
1. Update `appinfo/info.xml`: version to next semver (e.g., 1.0.0 → 1.1.0).
2. Update `appinfo/info.xml`: dependency on openregister, openconnector, docudesk, decidesk.
3. Add CHANGELOG.md entry:
   ```
   ## [1.1.0] - 2026-05-22
   ### Added
   - Supplier onboarding portal with category-driven questionnaires (REQ-001).
   - Real-time KvK validation (REQ-002).
   - IBAN validation with name-on-account check (REQ-003).
   - BIBOB risk-tiering and routing for high-risk suppliers (REQ-004).
   - MVI compliance proof requirement for suppliers > €50k (REQ-005).
   - Peppol directory validation (REQ-006).
   - Multi-step role-based approval workflow (REQ-007).
   - Annual revalidation with 1-click confirm (REQ-008).
   ```
4. Create release notes (link to this OpenSpec change).

**Dependencies**: All tasks

---

## Verification Checklist

- [ ] All 5 entity schemas defined and seed data loaded.
- [ ] 6 backend services implemented with full business logic.
- [ ] 9 frontend Vue components complete and integrated.
- [ ] 6 API controllers (Questionnaire, Document, Dossier, Approval, Validation, Revalidation).
- [ ] 3 background jobs (Revalidation check, Expire, ERP sync).
- [ ] All public methods have `@spec` PHPDoc tags.
- [ ] Deduplication check completed (no overlap with OpenRegister).
- [ ] Feature tests (BDD) cover all REQ-001 to REQ-008.
- [ ] Unit tests for all services.
- [ ] Integration tests for end-to-end workflows.
- [ ] Browser/E2E tests for UI flows.
- [ ] Standards compliance verified.
- [ ] Migration created for schema install.
- [ ] App version bumped; changelog updated.
- [ ] All code passes linter (PHPStan, ESLint).
- [ ] All tests pass (100% coverage for services, >80% overall).

---

**Status**: Draft — Ready for implementation  
**Created**: 2026-05-22  
**Updated**: 2026-05-22
