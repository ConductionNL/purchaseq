---
app: purchaseq
spec: inhuur-derden-wnra-wnt
title: Inhuur Derden met Wet DBA en WNT Toetsing — Implementation Tasks
status: draft
version: 1.0
date: 2026-05-22
---

# Implementation Tasks: Inhuur Derden met Wet DBA en WNT Toetsing

## Overview

Implementation organized in phases: Foundation → Compliance Checks → Governance & Export → Audit Hardening.
Each phase is tracked as a series of focused, testable tasks.

Total estimated effort: 26 weeks (6 months, distributed across team).

---

## Phase 1: Foundation (Weeks 1-6)

### Task 1.1: Data Model & Schema Definition

**Description**: Define and validate all OpenRegister schemas (InhuurOpdracht, Opdrachtnemer, DbaAssessment, WntToets, InhuurRatio, PublicatieRecord).

**Subtasks**:
- [ ] Create `lib/Settings/purchaseq_register.json` with OpenAPI 3.0 + x-openregister extensions.
- [ ] Define InhuurOpdracht schema (PascalCase, schema.org alignment per ADR-011).
- [ ] Define Opdrachtnemer schema (supports both natural person ZZP and org/agency).
- [ ] Define DbaAssessment schema (12 questions, scoring, conclusie).
- [ ] Define WntToets schema (aggregation, percentage, publication flag).
- [ ] Define PublicatieRecord schema (disclosure-ready fields).
- [ ] Define InhuurRatio schema (ratio calc, thresholds).
- [ ] Add schema relations (1:n, 1:1) using OpenRegister relation mechanism.
- [ ] Validate schema types against schema.org vocabulary (not custom names).
- [ ] Generate TypeScript interfaces from schemas (lib/Types/).
- [ ] Peer review: architecture team, DBA/WNT domain expert.

**Done Criteria**:
- All 7 schemas defined, validated, passing schema-validator.
- Relations configured.
- TypeScript interfaces auto-generated and type-safe.
- Peer reviewed and approved.

---

### Task 1.2: Seed Data Generation

**Description**: Create realistic, Dutch-localized seed data (3-5 objects per schema) for dev/test.

**Subtasks**:
- [ ] Create seed data for Opdrachtnemer: 5 realistic contractors (ZZP, agency, consultant).
  - Use real Dutch street names, valid postcodes (regex: `[1-9][0-9]{3}[A-Z]{2}`).
  - Use realistic KvK numbers (8-digit).
  - Vary: gender, company size, sectors (IT, HR, strategy).
- [ ] Create seed data for InhuurOpdracht: 5 contracts linked to seed Opdrachtnemers.
  - Vary: hire-type (zzp, uitzend, consultancy), status (concept, actief, afgerond).
  - Include examples with different dates (past, current, future-dated).
- [ ] Create seed DbaAssessment: 1-2 per contract (show green, orange, red scores).
- [ ] Create seed WntToets: for contracts with wntRelevant = true, show various aggregation % levels (25%, 80%, 105%).
- [ ] Create seed PublicatieRecord: show examples with all required fields.
- [ ] Create seed InhuurRatio: 2-3 cost centers, show compliant and non-compliant ratios.
- [ ] Validate seed data:
  - All references resolvable (OpdrachtnemerIds exist, etc.).
  - Dates logically consistent (startDatum < endDatum).
  - Financial values realistic (uurtarief 25-200 EUR, budgets 10k-500k EUR).
  - Cross-register consistency (BAG postcodes, KvK numbers in valid range).
- [ ] Load seed data via ImportService.importFromApp() and verify no duplicates on re-import.
- [ ] Create seed data JSON per design.md "Seed Data" section.

**Done Criteria**:
- 30+ seed objects created (5 Opdrachtnemer, 5 InhuurOpdracht, 5+ DbaAssessment, 3+ WntToets, 2+ PublicatieRecord, 2+ InhuurRatio).
- All objects pass validation.
- Loadable via ImportService, idempotent.
- Seed data used in automated browser tests (see Phase 2).

---

### Task 1.3: DBA Assessment Service (Backend)

**Description**: Implement DbaAssessmentService with 12-question framework, risk-scoring logic, and persistence.

**Subtasks**:
- [ ] Create DbaAssessmentService class with methods:
  - `createAssessment(contractId: string): DbaAssessment` — new assessment for contract.
  - `updateAssessment(assessmentId: string, answers: {q: number, answer: string}[]): DbaAssessment` — save Q&A.
  - `calculateRiskScore(answers: {}): {score: 0-100, conclusie: "groen"|"oranje"|"rood"}` — scoring logic.
  - `requiresEscalation(score: number): boolean` — flag for three-sig if red.
  - `getAssessmentHistory(contractId: string): DbaAssessment[]` — audit trail.
- [ ] Define 12 standard DBA questions (hardcoded per Belastingdienst):
  - Q1: Gezagsrelatie / controle.
  - Q2: Instructiemacht.
  - Q3: Vervangbaarheid.
  - Q4: Ondernemersrisico.
  - Q5: Eigen werkmiddelen.
  - Q6: Eigen verzekeringen.
  - Q7: Meerdere opdrachtgevers.
  - Q8: Tarietstelling / onderhandelingen.
  - Q9: Factuurfrequentie.
  - Q10-Q12: Additional risk factors per guidance.
- [ ] Scoring algorithm:
  - Each "ja" answer = risk increase (varies by question).
  - Calculate 0-100 score via weighted sum.
  - Conclusie mapping: <30 = groen, 30-60 = oranje, >60 = rood.
- [ ] Integration with ObjectService:
  - Save assessment as OpenRegister object (DbaAssessment schema).
  - Link to InhuurOpdracht via relation.
- [ ] Validation:
  - All 12 questions must be answered (no null/empty).
  - Answers must be from enum: {ja, nee, deels, onbekend}.
- [ ] Error handling:
  - Contract not found → throw ContractNotFoundException.
  - Invalid answer → throw ValidationException.
- [ ] Logging: AuditTrailService (automatic via ObjectService).
- [ ] Unit tests:
  - Test score calculation: all-green → score <30, all-red → score >60.
  - Test conclusie mapping: score 20 → groen, 45 → oranje, 75 → rood.
  - Test question validation: missing Q5 → ValidationException.
  - Test history retrieval: contract with 2 assessments → both returned in order.

**Done Criteria**:
- DbaAssessmentService fully implemented and tested.
- 12 questions defined and validated.
- Scoring logic tested with edge cases.
- Integration with ObjectService verified.
- Unit test coverage >90%.

---

### Task 1.4: Contract Creation UI (Frontend)

**Description**: Build the InhuurOpdracht creation form using CnDetailPage + CnFormDialog, with inline DBA assessment form.

**Subtasks**:
- [ ] Create contract-create page (`ContractCreatePage.vue`):
  - Route: `/contracts/new`.
  - Page layout: two-column (form + preview).
- [ ] Build form (using CnFormDialog or inline form) with fields:
  - Opdrachtgever (employee selector, searchable).
  - Opdrachtnemer (vendor selector, linked to WAADI validation—defer to Phase 2).
  - OpdrachtOmschrijving (textarea, max 500 chars).
  - InhuurType (radio: zzp / uitzend / detachering / consultancy / interim).
  - StartDatum, Geplande EindDatum (date pickers).
  - VerlengingsOptie (checkbox).
  - Uurtarief (currency input, validation min >0).
  - GeschatTotaalbudget (auto-calc from uurtarief * estimated hrs if provided, or manual).
  - Kostenplaats (code selector, auto-complete from HRMQ).
  - ProjectCode (optional text).
- [ ] Save button → POST /api/contracts (backend creates InhuurOpdracht).
- [ ] Validation:
  - Required fields: opdrachtgever, opdrachtnemer, opdrachtOmschrijving, startDatum, uurtarief.
  - Date validation: startDatum < geplande EindDatum.
  - Uurtarief > 0.
- [ ] Success: redirect to contract detail page, show toast "Contract aangemaakt: [opdrachtNummer]".
- [ ] Error: show error banner with validation details.
- [ ] Undo/cancel: navigate back without saving.

**Done Criteria**:
- Contract-create form fully functional.
- All required fields validated client-side and server-side.
- Success/error messaging clear.
- Manual testing: create contract with all field types.

---

### Task 1.5: Contract Detail Page with DBA Form

**Description**: Build the contract detail view (CnDetailPage) with embedded DBA assessment section, status transitions, and action buttons.

**Subtasks**:
- [ ] Create contract-detail page (`ContractDetailPage.vue`):
  - Route: `/contracts/{id}`.
  - Layout: multiple sections/cards.
- [ ] Section 1: Contract Summary Card
  - Display: opdrachtNummer, opdrachtnemernaam, inhuurType, dates, tarief, budget.
  - Edit button: opens CnFormDialog for field updates.
  - Status badge (concept/in-onderzoek/goedgekeurd/actief/verlengd/afgerond/geblokkeerd).
- [ ] Section 2: DBA Assessment Card
  - Display: DBA assessment form (12 questions) if exists, else "Beoordeling vereist".
  - Real-time score display (bar chart, color-coded).
  - Risk conclusie (groen/oranje/rood) prominently shown.
  - Submit button to save assessment.
  - History link: show prior assessments (if reassessment triggered).
  - Red assessment indicator: "This contract requires three-signature approval before activation."
- [ ] Section 3: Model Agreement Card
  - Display: current model-agreement kenmerk + validation date.
  - If missing or expired: blocker banner + "Configure Model Agreement" button.
  - On click: open dialog from REQ-MOD-001 (three-path selection).
- [ ] Section 4: WNT Tracking (if wntRelevant = true)
  - Display: cumulative payment YTD, WNT norm, percentage reached, threshold alerts.
  - Linked PublicatieRecords (if any).
  - (Fully implemented in Phase 2 once Shillinq integration ready.)
- [ ] Section 5: Status Transitions
  - Show available status transitions (depends on current status + assessments).
  - Grayed-out transitions if blocked (e.g., "goedgekeurd" disabled if red DBA unresolved).
  - Tooltip on hover: explain blocker.
  - Save button: validate before transition, show appropriate dialog (e.g., three-sig flow).
- [ ] Section 6: Files / Audit Tab
  - Standard CnObjectSidebar tabs (files, audit trail, metadata).
  - Files: contract PDF, assessment PDF, knowledge-transfer-doc, exit-eval-doc.
  - Audit: full change history (via AuditTrailService).
- [ ] Page-level validations:
  - Contract exists (404 if not found).
  - User has permission to view/edit.
- [ ] Navigation:
  - Breadcrumb: Home > Contracts > [opdrachtNummer].
  - Back button.

**Done Criteria**:
- Contract detail page fully functional with all 6 sections.
- DBA form renders inline and calculates score in real-time.
- Status transitions validated and blocked appropriately.
- Manual testing: open contract, fill DBA form, see score update live.

---

### Task 1.6: Contract Status Lifecycle & Validation

**Description**: Implement status-transition logic with validation rules (DBA complete before "goedgekeurd", red DBA blocks approval, etc.).

**Subtasks**:
- [ ] Define status-transition matrix:
  - concept → in-onderzoek (no condition).
  - in-onderzoek → goedgekeurd (requires: DBA assessment complete, dbaConlusie != "rood" OR red_overrideAutorisatie.goedgekeurd = true, model agreement valid).
  - goedgekeurd → actief (no additional condition, but Shillinq invoicing may block).
  - actief → verlengd (optional, if verlengingsOptie = true).
  - actief/verlengd → afronding (triggers closeout checklist).
  - afronding → afgerond (requires: closeout checklist complete).
  - Any status → geblokkerd (admin override, with reason).
- [ ] Implement ContractStatusService:
  - `canTransition(contractId: string, newStatus: string): {allowed: boolean, blocker?: string}` — validation.
  - `getAvailableTransitions(contractId: string): string[]` — list next valid statuses.
  - `transitionStatus(contractId: string, newStatus: string, reason?: string): Promise<Contract>` — perform transition.
- [ ] Validation logic:
  - DBA check: if newStatus = "goedgekeurd", verify DbaAssessment.conclusie != "rood" OR red override present.
  - Model agreement check: if inhuurType = "zzp", verify Opdrachtnemer.modelovereenkomstKenmerk is present and not >5 yrs old.
  - Closeout check: if newStatus = "afgerond", verify all CloseoutChecklist items checked.
- [ ] For red DBA transitions: trigger Decidesk workflow (defer to Phase 3, placeholder for now).
- [ ] Audit trail: every status transition logged with reason.
- [ ] Error handling:
  - Invalid transition → throw InvalidTransitionException with available options.
  - Missing prerequisite (e.g., DBA) → throw PrerequisiteException with guidance.
- [ ] Unit tests:
  - Test valid transitions: concept → in-onderzoek → goedgekeurd → actief.
  - Test blocked transitions: rood DBA blocks goedgekeurd.
  - Test red override: with three-sig approval, rood DBA allows goedgekeurd.

**Done Criteria**:
- Status lifecycle fully implemented and tested.
- Validation prevents invalid transitions.
- Unit test coverage >90%.

---

## Phase 2: Compliance Checks (Weeks 7-12)

### Task 2.1: Shillinq Invoice Webhook Listener

**Description**: Implement webhook listener for invoice creation/payment from Shillinq, with vendor/contract matching and amount aggregation.

**Subtasks**:
- [ ] Create webhook endpoint: POST `/api/webhooks/shillinq-invoice` (CloudEvents format).
- [ ] Webhook handler logic:
  - Parse CloudEvents payload (vendor-id, amount, invoice-date, invoice-id).
  - Query Opdrachtnemer by vendor-id (external system ID or KvK).
  - Find active InhuurOpdracht: startDatum ≤ invoice-date < endDate.
  - If no match: log warning (invoice may be unrelated), return success.
  - If match found: aggregate amount to InhuurOpdracht.daadwerkelijkeUitgaven.
  - Call WNT aggregation service (REQ-WNT-001).
  - Call ratio recalculation trigger (REQ-RAT-001, async job).
- [ ] Idempotency:
  - Use invoice-id as idempotency key (don't double-count same invoice).
  - Check if invoice already processed (e.g., stored in invoice-receipt log).
- [ ] Error handling:
  - Payload missing required fields → log error, return 400.
  - Vendor not found → log warning, return 202 (acceptable, may be external vendor).
  - Contract matching fails → log warning, return 202.
  - DB update fails → retry 3x with exponential backoff, eventually dead-letter for manual review.
- [ ] Logging: AuditTrailService per invoice (vendor_matched, contract_matched, amount_aggregated).
- [ ] Integration test:
  - Send mock CloudEvents payload → verify InhuurOpdracht.daadwerkelijkeUitgaven updated.
  - Verify WNT check triggered.
  - Verify ratio job queued.

**Done Criteria**:
- Webhook endpoint functional and receiving Shillinq payloads (test with mock).
- Vendor/contract matching logic validated.
- Amount aggregation tested.
- Idempotency verified (same invoice not double-counted).
- Error handling covers all paths.
- Integration tests passing.

---

### Task 2.2: WNT Wage-Cap Aggregation Service

**Description**: Implement WntService with cumulative payment aggregation, threshold detection, and notification dispatch.

**Subtasks**:
- [ ] Create WntService with methods:
  - `aggregatePaymentsForYear(opdrachtnemerId: string, year: number): {total: number, percentage: number}` — sum all invoices for contractor in calendar year.
  - `updateWntToets(opdrachtId: string, aggregatedTotal: number): WntToets` — update or create record.
  - `checkThresholds(wntToets: WntToets): {threshold80: boolean, threshold100: boolean}` — check 80% and 100% crossings.
  - `triggerThreshold80Notification(contract: InhuurOpdracht, wntToets: WntToets)` — notify Bestuurder + WNT officer (yellow alert).
  - `triggerThreshold100Notification(contract: InhuurOpdracht, wntToets: WntToets)` — notify + auto-create PublicatieRecord (red alert).
- [ ] Aggregation logic:
  - Query: all Shillinq invoices for this Opdrachtnemer in calendar year (startDate = Jan 1, endDate = Dec 31).
  - Sum: invoice.amount (filter out void/canceled invoices).
  - Store: WntToets.totaleuiutkeringTotNuToe.
  - Calculate percentage: (total / wntNormJaar) * 100, store in WntToets.percentageNormBereikt.
- [ ] Threshold logic:
  - if percentage >= 80: set threshold80_triggered = true, dispatch notification.
  - if percentage >= 100: set threshold100_triggered = true, trigger PublicatieRecord creation, dispatch notification.
- [ ] Notification dispatch (via NotificationService):
  - 80% alert: to role "Bestuurder" + "WNT-verantwoordelijke", title "WNT-norm approaching", body "[contractant] at 80% of cap (€X / €Y)".
  - 100% alert: same recipients, title "WNT-norm exceeded", body "[contractant] at 105% of cap (€X / €Y), publication record created".
- [ ] WNT norm configuration:
  - Query app config (IAppConfig) for norm for given year.
  - Default: 2026 = 150,000 EUR (per official WNT norm), etc.
  - If year not found: log error, skip aggregation.
- [ ] Idempotency:
  - On re-aggregation (e.g., invoice amended): update WntToets, re-check thresholds.
  - Don't create duplicate notifications (track notification_sent flag per threshold crossing).
- [ ] Unit tests:
  - Test aggregation: 3 invoices, total = 120k, norm = 150k → percentage = 80% → threshold80 = true.
  - Test 100% crossing: 3 invoices, total = 155k, norm = 150k → percentage = 103% → threshold100 = true.
  - Test notification dispatch: verify NotificationService.notify() called with correct recipients.

**Done Criteria**:
- WntService fully implemented with aggregation, thresholds, notifications.
- Configuration-driven norms.
- Unit tests covering all scenarios.
- Integration test: invoice webhook → WntService aggregation → notification dispatch.

---

### Task 2.3: Publication Record Generation

**Description**: Implement automatic creation of PublicatieRecord when WNT cap crossed (>100%).

**Subtasks**:
- [ ] Create PublicationService with method:
  - `createPublicationRecord(wntToets: WntToets): PublicatieRecord` — auto-create disclosure record.
- [ ] Field population:
  - naam: from Opdrachtnemer.naam.
  - functie: from InhuurOpdracht.opdrachtOmschrijving (truncate to function title if needed).
  - dienstbetrekking_aard: from InhuurOpdracht.inhuurType.
  - beloning: from WntToets.totaleuiutkeringTotNuToe.
  - startdatum: from InhuurOpdracht.startDatum.
  - einddatum: from InhuurOpdracht.geplande EindDatum or actueleEindDatum.
  - status: "concept" (awaiting WNT officer review).
- [ ] Persistence:
  - Save as OpenRegister object (PublicatieRecord schema).
  - Link to WntToets via relation.
- [ ] Idempotency:
  - If PublicatieRecord already exists for this WntToets: update (don't create duplicate).
  - Use wntToets_id as unique key.
- [ ] Validation:
  - All required fields populated (no nulls).
  - dienstbetrekking_aard must match enum (zzp, uitzend, detachering, consultancy, interim).
- [ ] Audit trail: publication_record_created, wnt_toets_id, contract_id.
- [ ] Unit tests:
  - Test creation: WntToets with percentage >100 → PublicatieRecord created with all fields.
  - Test idempotency: call twice → only one record exists.
  - Test field mapping: verify all fields correctly populated.

**Done Criteria**:
- PublicationService implemented and tested.
- PublicatieRecord schema and storage functional.
- Automatic creation triggered from WNT threshold check.
- Unit test coverage >90%.

---

### Task 2.4: Model Agreement Validation (KvK Adapter Integration)

**Description**: Implement model agreement check with KvK adapter lookup, and REQ-MOD-001 dialog flow (three registration paths).

**Subtasks**:
- [ ] Create ModelAgreementService with methods:
  - `validateModelAgreement(opdrachtnemer: Opdrachtnemer): {valid: boolean, reason?: string}` — check kenmerk presence + age.
  - `refreshFromKvK(opdrachtnemerId: string): {updated: boolean}` — query KvK adapter, update Opdrachtnemer.
  - `registerModelAgreement(opdrachtnemerId: string, type: string, kenmerk: string, file?: File)` — save agreement reference.
- [ ] Validation logic:
  - If Opdrachtnemer.modelovereenkomstKenmerk is null → invalid.
  - If Opdrachtnemer.datumLaatsteModelovereenkomstCheck > 5 years ago → invalid.
  - Otherwise: valid.
- [ ] KvK adapter integration (deferred if adapter unavailable, manual process):
  - Query Belastingdienst model-agreement registry (or mock list hardcoded for MVP).
  - Return: list of valid model agreements for this contractor type (sector, employee count, etc.).
- [ ] Registration paths:
  - Path A (select existing): dropdown of Belastingdienst models → user selects → save kenmerk + date.
  - Path B (upload branch): file upload (PDF/DOCX) → backend stores file, marks as "branche-specifiek", sets file-id as kenmerk.
  - Path C (bespoke): display external guidance + mailto link (no system state change).
- [ ] UI Dialog (REQ-MOD-001):
  - Trigger: status transition to "goedgekeurd" on ZZP contract.
  - Check: validateModelAgreement() == false → show blocker dialog.
  - Dialog content:
    - Title: "Modelovereenkomst Verplicht".
    - Explanation: why required.
    - Radio buttons: A, B, C.
    - For A: dropdown (populated from KvK adapter or mock).
    - For B: file-upload field (max 10 MB).
    - For C: text + external link.
    - Confirm button: save choice, update Opdrachtnemer, close dialog.
- [ ] Persisten ce:
  - Opdrachtnemer.modelovereenkomstType, modelovereenkomstKenmerk, datumLaatsteModelovereenkomstCheck updated.
  - File (path B): stored in backend (docstore or file service), reference saved.
- [ ] Audit trail: model_agreement_validated, registration_path_chosen, file_uploaded (if B), record_updated.
- [ ] Error handling:
  - KvK adapter unavailable → fallback to manual registration (skip auto-check, proceed with user input).
  - File upload fails → show error, allow retry or path change.
- [ ] Unit tests:
  - Test validation: kenmerk absent → invalid, date >5 yrs → invalid, date <5 yrs + kenmerk present → valid.
  - Test path A: dropdown shows models, user selects, kenmerk stored.
  - Test path B: file uploaded, file-id stored as kenmerk.
  - Test path C: external link functional (no state change).

**Done Criteria**:
- ModelAgreementService fully implemented.
- Dialog flow (REQ-MOD-001) functional in contract detail page.
- KvK adapter integration (or mock) working.
- Manual testing: ZZP contract, trigger model-agreement validation, select path A → complete dialog → status transitions.
- Unit test coverage >90%.

---

### Task 2.5: WAADI Registration Check (KvK Adapter)

**Description**: Implement WAADI validation for temp-staffing/secondment vendors (REQ-VEN-001, REQ-VEN-002), blocking hire if registration missing.

**Subtasks**:
- [ ] Create VendorValidationService with methods:
  - `validateWaadi(kvkNummer: string): {registered: boolean, waadi_number?: string}` — query KvK adapter.
  - `checkWaadiStatus(opdrachtnemerId: string): {valid: boolean, status: string}` — verify Opdrachtnemer WAADI status.
- [ ] KvK adapter integration:
  - Call: `kvk_adapter.lookup_vendor(kvk_number)`.
  - Parse: `waadi_registration` (true/false), `waadi_number` (string if present).
  - Handle error: adapter unavailable → flag for manual review (don't block).
- [ ] Blocker logic (REQ-VEN-001):
  - Trigger: vendor-selector form, user enters KvK number, clicks "Controleer WAADI" (or auto on blur).
  - For inhuurType = "uitzend" or "detachering": validate WAADI.
  - If registered = false: show blocking error dialog:
    - Title: "WAADI-registratie Ontbreekt".
    - Message: "Leverancier [naam] heeft geen geldige WAADI-registratie op het Handelsregister."
    - Link: "Controleer op KvK-website: [kvk_url]".
    - Button: "Annuleer" (no form submission).
  - If registered = true: Opdrachtnemer.waadiRegistratie = true, waadiNummer = [from adapter], proceed.
- [ ] Annual re-check (REQ-VEN-002):
  - Scheduled job: runs on contract anniversary (1 year after startDatum).
  - For each active InhuurOpdracht with inhuurType = "uitzend"|"detachering":
    - Call VendorValidationService.checkWaadiStatus().
    - If status changed from true → false: mark contract "geblokkerd", notify inkoper.
  - Log: anniversary_date, vendor_id, old_status, new_status, action_taken.
- [ ] Persistence:
  - Opdrachtnemer.waadiRegistratie (boolean), waadiNummer (string) updated.
  - InhuurOpdracht.status = "geblokkerd" if annual check fails.
- [ ] Error handling:
  - KvK adapter unavailable → log warning, skip check (don't block), allow manual review.
  - Invalid KvK number → show client error, allow retry.
- [ ] Unit tests:
  - Test validate: KvK with WAADI → registered = true, number stored.
  - Test validate: KvK without WAADI → registered = false, blocking error.
  - Test annual check: contract passes anniversary, WAADI still valid → no action.
  - Test annual check: WAADI revoked → contract blocked, notification sent.

**Done Criteria**:
- VendorValidationService implemented.
- WAADI validation integrated in vendor-selector form.
- Blocking error dialog functional.
- Annual re-check scheduled job working.
- Manual testing: create uitzend contract, select vendor without WAADI → blocker shown.
- Unit test coverage >90%.

---

### Task 2.6: Hire Duration & Budget Overrun Re-Assessment Trigger

**Description**: Implement automatic triggers for DBA re-assessment at 6 months or 125% budget overshoot (REQ-DBA-003).

**Subtasks**:
- [ ] Create ReassessmentTriggerService with methods:
  - `checkForReassessmentNeeded(contractId: string): {triggered: boolean, reason?: string}` — evaluate both triggers.
  - `triggerReassessment(contractId: string, reason: string): DbaAssessment` — create new assessment record.
  - `blockInvoiceApprovalUntilReassessment(contractId: string)` — set invoice-approval flag.
- [ ] Trigger 1: 6-month duration
  - Scheduled job: runs daily at 00:00 UTC.
  - For each InhuurOpdracht with status = "actief":
    - Calculate days elapsed: today - startDatum.
    - If days >= 180 AND no recent re-assessment (within 60 days):
      - Call triggerReassessment() with reason "6-month tenure milestone".
      - Set DbaAssessment.status = "vereist".
      - Set InhuurOpdracht flag: reassessment_required = true.
      - Notify inkoper: "[Contractor] has been hired for 6 months, DBA re-assessment required".
- [ ] Trigger 2: Budget overshoot
  - Webhook listener (in Shillinq hook handler):
    - After aggregating invoice amount to InhuurOpdracht.daadwerkelijkeUitgaven:
    - Check: daadwerkelijkeUitgaven > geschatTotaalbudget * 1.25.
    - If true AND no recent re-assessment:
      - Call triggerReassessment() with reason "Budget exceeded 125%".
      - Set InhuurOpdracht flag: reassessment_required = true.
      - Notify inkoper: "[Contractor] invoice exceeds budget by >25%, DBA re-assessment required".
- [ ] Invoice approval blocker:
  - When new invoice arrives for contract with reassessment_required = true:
    - Invoice approval button disabled in UI.
    - Tooltip: "Vereist her-assessment DBA voordat facturing kan doorgaan".
    - After DbaAssessment.status = "afgerond": clear reassessment_required flag, unblock approvals.
- [ ] Re-assessment form enhancements:
  - Show same 12 questions + additional question: "Zijn de gezagsrelatie of vervangbaarheid veranderd sinds begin contract?"
  - Compare new score vs. previous score: if score increased (e.g., 20 → 35), flag for escalation review.
- [ ] Persistence:
  - New DbaAssessment record created per reassessment.
  - InhuurOpdracht.reassessment_required flag tracks status.
  - Audit trail: trigger_reason, trigger_date, reassessment_created_date.
- [ ] Unit tests:
  - Test 6-month trigger: contract 180+ days old, no recent assessment → reassessment created.
  - Test budget trigger: invoice causing >125% overshoot → reassessment created, invoice blocked.
  - Test score escalation: old score 20, new score 65 → flag raised.

**Done Criteria**:
- ReassessmentTriggerService implemented.
- Scheduled job for 6-month check running daily.
- Webhook trigger for budget check in Shillinq listener.
- Invoice approval blocker functional.
- Manual testing: create ZZP contract, wait 180 days (mock date), verify reassessment triggered.
- Unit test coverage >90%.

---

## Phase 3: Governance & Export (Weeks 13-20)

### Task 3.1: Three-Signature Workflow Integration with Decidesk

**Description**: Implement red DBA approval flow (REQ-APP-001) via Decidesk decision engine.

**Subtasks**:
- [ ] Create ThreeSignatureService with methods:
  - `initiateRedDbaApproval(contractId: string): Decision` — create Decidesk workflow.
  - `handleApprovalCallback(decisionId: string, approvals: []): {allSigned: boolean}` — process Decidesk callback.
  - `finalizeApproval(contractId: string, approvals: {afdeling, controller, bestuurder})` — update InhuurOpdracht.red_overrideAutorisatie.
- [ ] Decidesk integration:
  - Call: `decidesk_client.create_decision(type="drie-handtekeningen-rood-dba", context={contract_id, contract_details})`.
  - Decidesk returns: decision_id, task_ids (3 tasks auto-created).
  - Callback URL: `/api/webhooks/decidesk-approval-completed`.
- [ ] Task assignment (Decidesk auto-routes):
  - Task 1 → role "Afdelingshoofd" (dept head).
  - Task 2 → role "Controller" (finance).
  - Task 3 → role "Bestuurder" (executive).
- [ ] Task content:
  - Contract details: opdrachtnemernaam, tarief, DBA-score, reden-voor-rood, risk-description.
  - Approval form:
    - Motivation field (min 50 chars, max 1000 chars, required).
    - Approve / Decline buttons.
  - Previous approvals visible (once signed, next signer sees prior motivations).
- [ ] Approval gate:
  - Workflow blocks on any "Decline" (fail state, contract remains "in-onderzoek").
  - Workflow succeeds only on all three "Approve" (unanimous gate).
  - Callback to this system: `/api/webhooks/decidesk-approval-completed?decision_id=XXX&status=approved`.
- [ ] Callback handler:
  - Verify decision_id, fetch decision details from Decidesk (optional, if details not in callback).
  - Extract: signer names, approval dates, motivations.
  - Update InhuurOpdracht.red_overrideAutorisatie:
    - goedgekeurd = true.
    - autoriseerderAfdeling, autoriseerderController, autoriseerderBestuurder = signer names/IDs.
    - datumGoedkeuring = latest approval timestamp.
    - motivering = concatenated motivations (or individual fields).
  - Clear reassessment_required flag (if applicable).
  - Notify inkoper: "Red DBA approval granted for [contractant], contract now approved".
- [ ] Contract detail UI:
  - If contract.status = "in-onderzoek" AND dbaConlusie = "rood" AND red_overrideAutorisatie empty:
    - Show banner: "Wacht op drie-handtekeningen-goedkeuring".
    - Show button: "Start Goedkeuringsstroom" (click → initiate Decidesk flow).
    - Show Decidesk decision status widget (3 task checklist, signer names, approval dates).
  - If red_overrideAutorisatie.goedgekeurd = true:
    - Show green checkmark: "Goedgekeurd door [3 signers] op [date]".
    - Show motivations as collapsed/expandable text.
- [ ] Error handling:
  - Decidesk unavailable → fallback UI: manual three-signature form (capture names, dates, motivations).
  - Callback missing → UI shows "Awaiting approval notification from Decidesk" (poll Decidesk API periodically).
- [ ] Audit trail: decidesk_initiated, task_created (3x), approvals_received (3x), contract_approved, motivations_logged.
- [ ] Unit tests:
  - Test initiation: create decision, 3 tasks auto-created.
  - Test callback: all three approvals → red_overrideAutorisatie.goedgekeurd = true.
  - Test gate: one decline → workflow fails, contract remains blocked.

**Done Criteria**:
- ThreeSignatureService fully implemented.
- Decidesk integration functional (or fallback form if unavailable).
- Callback handler processing Decidesk webhooks.
- Contract detail UI shows approval status and motivations.
- Manual testing: create red DBA contract, click "Start Goedkeuringsstroom", complete 3-signer approval in Decidesk, verify callback updates contract.
- Unit test coverage >90%.

---

### Task 3.2: Hiring Ratio Aggregation & Dashboard

**Description**: Implement InhuurRatio calculation (nightly scheduled job) and dashboard UI with trends and drill-down (REQ-RAT-001).

**Subtasks**:
- [ ] Create InhuurRatioService with methods:
  - `calculateRatioForCostCenter(kostenplaats: string, kwartaal: string, jaar: number): InhuurRatio` — compute one ratio record.
  - `calculateRatiosForAllCostCenters(kwartaal: string, jaar: number): InhuurRatio[]` — batch calculate all cost centers.
  - `calculateHistoricalRatios(kostenplaats: string, quarters: 8): InhuurRatio[]` — fetch last 8 quarters for trends.
- [ ] Ratio calculation logic:
  - Query HRMQ personnel: all employees with kostenplaats = [cost center].
  - Sum FTE: arbeidsuren / 40 (or from HRMQ field).
  - Sum payroll: salaries.
  - Query OpenRegister: all InhuurOpdracht with kostenplaats = [cost center], status in [actief, verlengd].
  - Sum external FTE: estimated from contract hours + dates.
  - Sum external cost: daadwerkelijkeUitgaven (from Shillinq aggregation).
  - Calculate ratio: externalCost / (internalCost + externalCost) * 100.
  - Compare to drempel: overschredenJaNee = (ratio > drempel).
  - Create/update InhuurRatio record.
- [ ] Scheduled job:
  - Runs nightly at 23:00 UTC (or post-Shillinq-webhook delay).
  - For each unique kostenplaats in OpenRegister: calculate current quarter + year.
  - Upsert InhuurRatio records.
  - Log: job_start, cost_centers_processed, records_updated, job_end.
- [ ] Drempel configuration:
  - Query app config (IAppConfig): hiring_ratio_threshold (default 25%).
  - Allow per-cost-center override (admin setting).
- [ ] Dashboard page (`HiringRatioDashboard.vue`):
  - Route: `/governance/hiring-ratio`.
  - Layout:
    - Top: filter bar (date range, cost center, compliance status), export button.
    - Main: grid of ratio cards (one per cost center, current quarter).
  - Ratio card:
    - Title: kostenplaats + description.
    - Large number: ratio % (bold, color-coded: green <30%, yellow 30-50%, orange 50-70%, red >70%).
    - Subtext: "Norm: X% | Actual: Y% | [✓ Compliant / ⚠ Above Norm]".
    - Trend sparkline: 8-quarter history (line chart, mini).
    - Click → drill-down modal.
- [ ] Drill-down modal:
  - Full-size line chart: 8 quarters, ratio % over time, threshold line, color bands.
  - Table: all active/recent contracts for cost center.
    - Columns: Contract ID, Contractor, Hire Date, Spend YTD, DBA Status, WNT Flag, Status.
    - Sortable/filterable.
  - Export: CSV of cost-center ratio history + detailed contracts.
- [ ] Filters:
  - Date range: quarter/year picker.
  - Cost center: multi-select dropdown (searchable).
  - Compliance: filter "Compliant Only" or "Above Norm Only".
- [ ] Color coding:
  - Green: ratio < 30% (or configured threshold).
  - Yellow: 30-50%.
  - Orange: 50-70%.
  - Red: > 70% (or threshold exceeded).
- [ ] Persistence:
  - InhuurRatio records persisted in OpenRegister.
  - Historical data retained indefinitely (7-year legal hold via Docudesk, but ratios are metrics).
- [ ] Error handling:
  - HRMQ query fails → skip cost center (log warning), continue others.
  - Missing threshold config → use default 25%.
- [ ] Unit tests:
  - Test calculation: 10 internal FTE, 2 external FTE, cost ratio 16% → ratio% = 16.
  - Test threshold: ratio 16% < threshold 25% → overschredenJaNee = false (compliant).
  - Test threshold: ratio 50% > threshold 25% → overschredenJaNee = true (non-compliant).
  - Test historical: query 8-quarter history, verify ordered by date.
- [ ] Integration tests:
  - Scheduled job runs, InhuurRatio records created.
  - Dashboard loads, displays ratio cards with correct colors.
  - Drill-down modal shows contract list.

**Done Criteria**:
- InhuurRatioService implemented with calculation logic.
- Scheduled job running nightly, ratios updated.
- Dashboard page fully functional with cards, trends, filters, drill-down.
- Manual testing: open dashboard, verify ratio cards, drill-down to contracts, export CSV.
- Unit test coverage >90%.
- Integration tests passing.

---

### Task 3.3: End-of-Contract Closeout Checklist UI

**Description**: Implement closeout checklist form (REQ-CLO-001) with 6 items, file uploads, and archival gate.

**Subtasks**:
- [ ] Create CloseoutChecklistService with methods:
  - `createChecklist(contractId: string): CloseoutChecklist` — initialize checklist on status transition.
  - `checklistItem(contractId: string, itemKey: string): {checked: boolean, evidence?: File}` — check/uncheck item.
  - `canArchive(contractId: string): {allowed: boolean, uncheckedItems?: string[]}` — validate readiness.
  - `archiveContract(contractId: string): {docudesk_dossier_id: string}` — finalize and archive.
- [ ] Checklist items (6 total):
  - [ ] Laatste factuurregel ontvangen (checkbox only).
  - [ ] Geheimhoudingsverklaring (NDA) geretourneerd (checkbox only).
  - [ ] Toegangspassen/badges ingenomen (checkbox only).
  - [ ] IT-accounts beëindigd (checkbox only).
  - [ ] Kennisoverdracht-document geüpload (checkbox + file upload, optional link field).
  - [ ] Eindevaluatie-formulier ingevuld (checkbox + file upload, optional link field).
- [ ] Persistence:
  - CloseoutChecklist schema: checklist_id, contract_id, items[] (each with checked, evidence_file_id, timestamp_checked_by).
  - File storage: knowledge-transfer, exit-eval PDFs stored in file service (linked to contract).
- [ ] UI Integration (in contract detail):
  - Section: "Afsluitingsproces".
  - Trigger: status transition actief/verlengd → afronding.
  - Dialog: "Afsluitings-Checklist", all 6 items shown as checkboxes.
  - For items 5-6 (file uploads): file-upload field appears below checkbox (conditional).
  - Action buttons:
    - "Archiveer & Sluit" (disabled until all items checked).
    - "Annuleer" (return to contract detail without changes).
  - Tooltip on disabled button: "Controleer alle items voordat archivering".
- [ ] Item checking:
  - On click checkbox: set item.checked = true, item.checked_at = now, item.checked_by = current_user.
  - UI updates in real-time.
  - Timestamp displayed: "Afgevinkt door [user] op [date] [time]".
- [ ] File uploads (items 5-6):
  - Allow drag-drop or file picker.
  - Max size: 10 MB per file.
  - Types: PDF, DOCX, XLS, JPG, PNG.
  - Store: backend file service, get file_id, save to CloseoutChecklist.items[].evidence_file_id.
  - Display: "[filename] uploaded on [date]" with delete/re-upload link.
- [ ] Archival flow:
  - On "Archiveer & Sluit" click:
    - Validate: all 6 items checked (canArchive() == true).
    - Disable button, show loading spinner.
    - Call archiveContract(contractId):
      - Fetch contract + all supporting docs (DBA assessment, WNT toets, closeout checklist, files).
      - Call Docudesk API: create_dossier(contract_id, documents[]).
      - Docudesk returns dossier_id.
      - Update InhuurOpdracht.status = "afgerond", docudesk_dossier_id = [id].
      - Notify inkoper: "Contract [opdrachtNummer] archived to Docudesk, dossier [id]".
    - Redirect to contract list (or detail, read-only).
- [ ] Audit trail: checklist_created, item_checked (per item, user, timestamp), file_uploaded, contract_archived, dossier_id.
- [ ] Error handling:
  - File upload fails → show error, allow retry.
  - Docudesk API unavailable → show warning, allow manual archival later.
- [ ] Unit tests:
  - Test checklist creation: 6 items, all unchecked.
  - Test item checking: check item 1, timestamp recorded.
  - Test canArchive: with 5 items checked, return false + list unchecked items.
  - Test canArchive: with all 6 checked, return true.
  - Test archival: call archiveContract, Docudesk dossier created, InhuurOpdracht.status = "afgerond".

**Done Criteria**:
- CloseoutChecklistService implemented.
- Checklist dialog UI integrated in contract detail.
- File uploads functional (items 5-6).
- Archival gate working (disabled until all items checked).
- Manual testing: create contract, move to afronding, complete checklist, archive.
- Unit test coverage >90%.

---

### Task 3.4: Annual WNT Report Export (PDF + XML)

**Description**: Implement annual board report export (REQ-EXP-001) with PDF and XML formats for publication.

**Subtasks**:
- [ ] Create WntExportService with methods:
  - `generateAnnualReport(year: number): {pdf_file: Buffer, xml_file: Buffer}` — generate both formats.
  - `generatePDF(year: number, records: PublicatieRecord[]): Buffer` — PDF generation.
  - `generateXML(year: number, records: PublicatieRecord[]): Buffer` — XML generation.
- [ ] Report data collection:
  - Query: all PublicatieRecords for year (status = "approved" or "published", if needed).
  - Query: InhuurRatio for year, aggregate external spend / total spend.
  - Gather: organization name, address, contact, WNT norm for year, audit info.
- [ ] PDF generation:
  - Template: use pdfkit or similar library (or pre-built template).
  - Content:
    - Header: organization logo, title "Jaarverslag Extern Personeel & WNT-Publicatie [Year]".
    - Section 1: Introduction (template text about WNT compliance).
    - Section 2: External Hiring Summary.
      - Total external spend (EUR).
      - Total internal spend (EUR, from payroll).
      - External hiring ratio %.
      - Comparison to WNT norm (e.g., "0 contractors exceeded norm" or "2 contractors exceeded norm").
    - Section 3: Contractor Disclosures (table format).
      - For each PublicatieRecord:
        - Name.
        - Function / role.
        - Employment type (zzp / uitzend / etc.).
        - Total remuneration (EUR).
        - Start date.
        - End date.
    - Section 4: Compliance Statement (template text).
    - Footer: generation date, organization contact, audit certifier (if applicable).
  - Output: PDF file, downloadable.
- [ ] XML generation:
  - Schema: simplified XML structure (or per official WNT schema if available).
  - Root element: `<jaarverslag>`.
  - Children:
    - `<jaar>2026</jaar>`.
    - `<organisatie><naam>...</naam><address>...</address><kvk>...</kvk></organisatie>`.
    - `<samenvatting><totale_extern_spend>...</totale_extern_spend><totale_intern_spend>...</totale_intern_spend><ratio>...</ratio><wnt_norm>...</wnt_norm></samenvatting>`.
    - `<contractanten>` (array of `<contractant>`):
      - `<naam>...</naam>`.
      - `<functie>...</functie>`.
      - `<arbeidstype>...</arbeidstype>`.
      - `<beloning>...</beloning>`.
      - `<startdatum>...</startdatum>`.
      - `<einddatum>...</einddatum>`.
  - Pretty-print XML for readability.
  - Output: XML file, downloadable.
- [ ] Export endpoint:
  - GET `/api/wnt-export?year=2026` (or POST if large payload).
  - Returns: ZIP archive containing both PDF and XML (or separate downloads, user choice).
- [ ] UI page (`WntExportPage.vue`):
  - Route: `/export/wnt-annual-report`.
  - Form:
    - Year selector (dropdown, 2020-2030).
    - Preview button: show summary (contractor count, spend totals, ratio, compliance status).
    - Export button: download ZIP or separate files.
  - On export:
    - Button shows loading spinner.
    - API call to generateAnnualReport(year).
    - Browser downloads files (Content-Disposition: attachment).
    - Toast: "Export successful, files ready for publication".
- [ ] Validation:
  - Year must be in valid range (not future year, not too old).
  - At least 1 PublicatieRecord exists for year (warn if none, but allow export anyway).
  - Organization metadata (name, address, KvK) configured (check IAppConfig).
- [ ] Audit trail: export_requested, year, record_count, pdf_generated, xml_generated, export_timestamp, user.
- [ ] Error handling:
  - Year not found → show error, suggest valid range.
  - PDF generation fails → fallback to text-only export, warn user.
  - XML schema invalid → log error, export continues (XML may be incomplete but valid structure).
- [ ] Unit tests:
  - Test PDF: 2 PublicatieRecords → PDF generated with both rows in table.
  - Test XML: same data → XML valid schema, all fields populated.
  - Test aggregation: calculate external spend from PublicatieRecords, verify in summary.

**Done Criteria**:
- WntExportService fully implemented.
- PDF and XML generation functional.
- Export page UI with year selector, preview, download.
- Manual testing: select 2026, export → download ZIP with both files, verify content.
- Unit test coverage >90%.

---

## Phase 4: Audit Hardening (Weeks 21-26)

### Task 4.1: Audit Trail Export & Validation (Hash-Chain Verification)

**Description**: Implement audit-trail export functionality and hash-chain verification for tax auditor reproduction (REQ-AUD-001).

**Subtasks**:
- [ ] AuditTrail leverage existing AuditTrailService (platform-provided):
  - Automatic logging on ObjectService.saveObject().
  - Fields: entity_type, entity_id, field_name, old_value, new_value, user, timestamp, hash.
- [ ] Hash-chain implementation (platform may provide):
  - On each audit entry creation: hash = SHA256(entry_json).
  - On subsequent entry: prev_hash = hash of previous entry.
  - Store both in audit log for tamper detection.
  - Validation endpoint: verify hash chain (recalculate hashes, compare to stored values).
- [ ] Export endpoint:
  - GET `/api/contracts/{id}/audit-trail?format=json|csv`.
  - Returns: full audit log for contract (all related entities: InhuurOpdracht, DbaAssessment, WntToets, etc.).
  - Format JSON: array of entries, sorted by timestamp.
  - Format CSV: columns (entity_type, entity_id, field_name, old_value, new_value, user, timestamp, hash, prev_hash).
- [ ] Validation endpoint:
  - GET `/api/contracts/{id}/audit-trail/validate`.
  - Recalculate hashes for entire chain, compare to stored values.
  - Returns: {valid: boolean, broken_at_entry?: number, errors?: []} (if chain broken, indicate which entry failed).
- [ ] UI (in contract detail):
  - Audit tab (standard CnObjectSidebar feature).
  - Show timeline of all changes.
  - Add button: "Export Audit Trail" → dropdown (JSON / CSV).
  - Add button: "Validate Audit Chain" → show result (green checkmark "Valid" or red error "Chain broken at entry X").
- [ ] Audit trail for sensitive data:
  - DBA assessment answers (questionnaire responses) are sensitive.
  - Audit log shows field names (e.g., "dbaAssessment.vragen[0].antwoord") but may mask values in logs (user does not see "ja"/"nee" values).
  - Export still includes all values (for auditor), but only after authorization check (role = auditor or administrator).
- [ ] Error handling:
  - Export endpoint: if no audit entries, return empty array.
  - Validation: if hashes missing (old entries pre-hash-chain), return warning "Hash verification not available for entries before [date]".
- [ ] Performance:
  - Audit query optimized (indexed by entity_id, timestamp).
  - Hash validation runs in background (long exports don't block UI).
- [ ] Unit tests:
  - Test export: contract with 5 audit entries → JSON export has 5 entries, CSV has 5 rows.
  - Test hash chain: 3 entries, hashes linked → validate chain returns true.
  - Test tamper detection: modify entry[1] hash, rerun validation → invalid, broken_at_entry = 1.

**Done Criteria**:
- Audit trail export functional (JSON + CSV).
- Hash-chain validation working.
- Audit tab in contract detail showing timeline.
- Manual testing: export audit trail, validate chain, detect tampering scenario.
- Unit test coverage >90%.

---

### Task 4.2: Docudesk Archival Integration

**Description**: Finalize archival workflow (REQ-CLO-001, closeout) with Docudesk dossier creation, retention policy, and auditor access (continuation of Task 3.3).

**Subtasks**:
- [ ] Docudesk API integration:
  - Service: DocudeskArchivalService with method:
    - `createDossier(contractId: string, documents: File[]): {dossier_id: string}`.
  - Docudesk API call: create dossier, set retention policy (7 years), upload documents.
- [ ] Documents included in dossier:
  - Original contract (PDF, if available).
  - DBA assessment (rendered as PDF or JSON export).
  - WNT toets (rendered as PDF or JSON).
  - PublicatieRecord (rendered as PDF or JSON, if applicable).
  - Closeout checklist (PDF or form export).
  - Knowledge-transfer document (uploaded file from checklist).
  - Exit-evaluation document (uploaded file from checklist).
  - Audit trail export (JSON, full history).
- [ ] Retention policy:
  - Set on dossier creation: 7 years from contract end date.
  - Docudesk will auto-delete after retention expires (configurable via Docudesk settings).
  - For re-hiring same contractor: dossier linked but accessible for historical audit.
- [ ] Access control:
  - Auditor role: can view/download dossier via Docudesk UI.
  - Controller: can view dossier (limited access).
  - Inkoper: can request dossier retrieval (admin approval required).
- [ ] Audit trail in archival:
  - Docudesk archival action logged to contract audit trail: "Contract archived to Docudesk, dossier_id = [id]".
- [ ] Error handling:
  - Docudesk unavailable → show warning, allow manual archival later (contract stays in "afronding" status).
  - Partial failure (some docs upload, some fail) → retry failed uploads, log errors.
- [ ] Testing:
  - Integration test: call createDossier with mock contract + docs, verify Docudesk API called, dossier_id returned.
  - Verify documents uploaded (mock Docudesk would confirm receipt).
  - Verify retention policy set (check dossier metadata).

**Done Criteria**:
- DocudeskArchivalService implemented.
- Archival integrated in closeout flow (Task 3.3).
- Manual testing: complete closeout checklist, archive contract, verify Docudesk dossier created with all docs.
- Integration tests passing.

---

### Task 4.3: Documentation & Training

**Description**: Create user guides, compliance guides, and training materials for purchaseq inhuur-derden feature.

**Subtasks**:
- [ ] User Guide (Inkoper):
  - PDF/HTML document.
  - Section 1: Overview of hiring process (concept → actief → afgerond).
  - Section 2: Creating a contract (form fields, required vs. optional, best practices).
  - Section 3: DBA Assessment (12 questions, scoring, interpreting risk level).
  - Section 4: Model agreements (three paths, when required).
  - Section 5: WAADI validation (what it checks, why important).
  - Section 6: Shillinq integration (how invoices aggregate, real-time WNT tracking).
  - Section 7: Closeout process (checklist, file uploads, archival).
  - Appendix: FAQ, troubleshooting.
- [ ] Compliance Guide (Controller / WNT Officer):
  - PDF/HTML document.
  - Section 1: DBA (Wet DBA, risk scoring, red DBA authorization).
  - Section 2: WNT (salary cap, aggregation, publication requirements, annual reporting).
  - Section 3: WAADI (Wet WAADI, validation, temp-staffing compliance).
  - Section 4: Audit trail (reproducibility for tax/wage audits, hash-chain verification).
  - Section 5: Reporting (hiring ratio dashboard, annual export).
  - Appendix: Regulatory references, model agreements.
- [ ] Auditor Guide (Internal/External Auditor):
  - PDF/HTML document.
  - Section 1: Accessing audit trails (export formats, hash validation).
  - Section 2: Reproducing compliance checks (DBA, WNT, WAADI).
  - Section 3: Dossier retrieval (Docudesk access, retention schedules).
  - Section 4: Common audit scenarios (testing red DBA approvals, verifying WNT publication, etc.).
- [ ] Training slides (PowerPoint/PDF):
  - Overview: why purchaseq inhuur-derden matters.
  - Demo: creating contract, DBA assessment, status transitions.
  - Demo: WNT tracking, threshold alerts, publication records.
  - Demo: dashboard, drill-down, export.
  - Hands-on exercises (sandbox data).
  - Q&A.
- [ ] Release notes:
  - Link to features by requirement (REQ-DBA-001, etc.).
  - Configuration guide (WNT norms, hiring ratio thresholds).
  - Known limitations / future work.
  - Support contact.

**Done Criteria**:
- All 4 guides written (user, compliance, auditor, training).
- Guides reviewed by domain experts (legal, tax, HR).
- Training slides prepared.
- Documentation published (wiki, PDF links from UI).
- Internal training delivered to stakeholders.

---

### Task 4.4: Deduplication Check & Reuse Validation

**Description**: Verify that all features leverage existing platform services (OpenRegister, Decidesk, Docudesk, KvK, HRMQ, Shillinq) and no custom duplication occurs.

**Subtasks**:
- [ ] Service overlap audit:
  - Review all implemented services (DbaAssessmentService, WntService, etc.).
  - Compare against existing platform services:
    - ObjectService (CRUD) → using ✓.
    - SchemaService (validation) → using ✓.
    - AuditTrailService (logging) → using ✓.
    - ImportService (seed data) → using ✓.
    - NotificationService (alerts) → using ✓.
    - AuthorizationService (RBAC) → using ✓.
    - CnDetailPage (UI layout) → using ✓.
    - CnFormDialog (forms) → using ✓.
    - CnChartWidget (dashboard) → using ✓.
  - Check: no custom CRUD, no custom auth, no custom forms, no custom dashboards.
  - Finding: all custom services are domain-specific (DBA scoring, WNT aggregation), not duplicating platform.
- [ ] Cross-app integration verification:
  - Shillinq integration: webhook listener, no re-implementation of invoice logic ✓.
  - KvK adapter: using existing OpenConnector service, no custom vendor lookup ✓.
  - Decidesk: using workflow engine, no custom three-sig logic ✓.
  - Docudesk: using dossier API, no custom file archival ✓.
  - HRMQ: using personnel API, no custom FTE calc library ✓.
- [ ] Code review checklist:
  - [ ] No custom ObjectService methods (all via platform ObjectService).
  - [ ] No custom permission/auth (all via AuthorizationService + roles).
  - [ ] No custom form validation (schema-based, not custom regex).
  - [ ] No custom data layer (OpenRegister + relations, no custom ORM/models).
  - [ ] No custom notification system (all via NotificationService).
  - [ ] No re-implemented aggregation (SQL/Elasticsearch for ratio, not application loop).
  - [ ] No custom file storage (use FileService, not custom upload handler).
- [ ] Documentation:
  - Create ARCHITECTURE.md with dependency diagram (purchaseq-inhuur-derden → OpenRegister, Shillinq, KvK, Decidesk, Docudesk, HRMQ).
  - Document each integration point and why not custom-built.

**Done Criteria**:
- Deduplication check completed.
- No significant overlaps found (or identified and justified).
- ARCHITECTURE.md written and reviewed.
- Code peer-reviewed by architect.

---

### Task 4.5: Integration & Smoke Tests

**Description**: Run full-stack integration tests covering end-to-end flows and smoke tests for UI/API.

**Subtasks**:
- [ ] Integration test suite (jest + backend test framework):
  - Test 1: Create ZZP contract → DBA assessment completes (green) → Status transitions to "goedgekeurd".
  - Test 2: Create Uitzend contract (red DBA) → Three-sig flow (mock Decidesk) → Contract approved.
  - Test 3: Receive Shillinq invoice → WNT aggregation triggers → Threshold 80% alert sent.
  - Test 4: WNT threshold 100% crossed → PublicatieRecord created → Export includes record.
  - Test 5: Budget overshoot → Re-assessment trigger → Invoice approval blocked → Re-assessment completes → Approval unblocked.
  - Test 6: 6-month duration → Re-assessment trigger → Form shows historical score comparison.
  - Test 7: Contract end → Closeout checklist → All items checked → Docudesk archival.
  - Test 8: Annual export → PDF + XML generated → Files contain all PublicatieRecords.
  - Test 9: Audit trail → Export JSON/CSV → Hash-chain validates.
  - Test 10: Hiring ratio dashboard → Filters apply → Drill-down table matches.
- [ ] Smoke tests (browser-based, Playwright/Cypress):
  - Test 1: Login → create contract form loads → fill fields → save contract → redirect to detail.
  - Test 2: Contract detail → DBA tab loads → fill 12 questions → score updates in real-time → submit.
  - Test 3: Model agreement validation → blocker dialog → select path A → dialog closes → contract approved.
  - Test 4: Hiring ratio dashboard → cards load → filter by cost center → drill-down opens.
  - Test 5: Closeout checklist → all items unchecked initially → check each item → file upload items → archival button enabled.
  - Test 6: WNT export → year selector → preview → download ZIP → files present.
  - Test 7: Audit trail export → JSON download → validate hash chain.
- [ ] Performance smoke tests:
  - Dashboard load: <2 seconds (with seed data 100 contracts).
  - Invoice webhook: <100ms (aggregate + notify).
  - PDF export: <5 seconds (annual report, 50 contractors).
- [ ] Error handling tests:
  - Create contract with missing required field → validation error shown.
  - Submit DBA assessment with unanswered question → form validation error.
  - Select vendor without WAADI → blocker error.
  - KvK adapter timeout → fallback to manual process (no error to user, just slower).
  - Docudesk archival fails → warning shown, contract remains in afronding, manual retry available.
- [ ] Run test suite:
  - npm test (unit + integration).
  - Playwright tests (browser smoke tests).
  - All tests pass with coverage >85%.

**Done Criteria**:
- 10 integration test scenarios passing.
- 7 smoke tests passing.
- 3 performance tests passing (<2s, <100ms, <5s).
- Error handling verified.
- Test coverage >85%.

---

### Task 4.6: Security Review & Penetration Testing

**Description**: Security audit of DBA assessment, WNT, WAADI, three-sig, and audit-log features.

**Subtasks**:
- [ ] Security checklist:
  - [ ] OWASP A01:2021 (broken access control):
    - Red DBA approvals require three distinct signers (no self-approval).
    - Audit log export requires "auditor" or "admin" role.
    - DBA assessment answers not viewable by non-authorized users.
  - [ ] OWASP A03:2021 (injection):
    - All Shillinq webhook payloads validated (schema validation).
    - KvK adapter responses parsed safely (no eval, strict JSON).
    - XML export generated via XML library (not string concat).
    - PDF export generated via PDF library (not template injection).
  - [ ] OWASP A02:2021 (cryptographic failures):
    - Audit log hash chain implemented (SHA256).
    - IBAN field validated but not logged in plaintext (schema masks in UI).
    - Confidentiality: DBA assessment answers logged only in audit trail (role-gated).
  - [ ] OWASP A04:2021 (insecure design):
    - DBA red-approval gate cannot be bypassed (three-sig, unanimous, Decidesk-enforced).
    - WAADI validation cannot be skipped (blocker on uitzend/detachering hire type).
    - Invoice approval cannot proceed until re-assessment done (InhuurOpdracht flag blocks at controller).
  - [ ] Input validation:
    - Uurtarief: min >0, max <10000 (reasonable bounds).
    - Postcode: regex `[1-9][0-9]{3}[A-Z]{2}` (Dutch format).
    - IBAN: regex `^NL[0-9]{2}[A-Z]{4}[0-9]{10}$` (Dutch IBAN).
    - Dates: startDatum < geplande EindDatum (logical order).
    - KvK: 8 digits (format validation).
    - All via schema-based validation (not custom code).
  - [ ] Rate limiting:
    - Webhook endpoint: standard DDoS protection (rate limit per IP / vendor_id).
    - Export endpoint: standard API rate limiting (10 req/min per user).
  - [ ] Audit trail immutability:
    - Audit log table: DELETE and UPDATE permissions revoked.
    - Hash chain: tamper detection on retrieval.
    - Export: read-only, no modification via export API.
- [ ] Penetration test scenarios:
  - Scenario 1: Attacker tries to bypass red-DBA approval by forging Decidesk callback.
    - Expected: callback validation (signature, decision_id, status) fails, approval rejected.
  - Scenario 2: Attacker tries to manipulate audit log (modify entry hash).
    - Expected: hash-chain validation detects tampering, subsequent entries invalidated.
  - Scenario 3: Attacker tries to skip WAADI validation by changing hire type after vendor selection.
    - Expected: vendor validation runs per hire type, re-validation required if hire type changes.
  - Scenario 4: Attacker tries to approve invoice before re-assessment required.
    - Expected: invoice approval button disabled if reassessment_required = true, backend validates.
  - Scenario 5: Attacker tries to export DBA answers without auditor role.
    - Expected: authorization check fails (role check in export endpoint), export denied.
- [ ] Fix any findings:
  - Security issues addressed, re-test.
  - Document all mitigations.
- [ ] Sign-off:
  - Security review completed, report generated.
  - All findings resolved.
  - Penetration test report archived.

**Done Criteria**:
- Security checklist completed.
- Penetration test scenarios run, all pass (no exploits successful).
- Findings documented and resolved.
- Security review report signed off.

---

## Rollout Plan

### Week 26: Deployment & Go-Live
- [ ] Final testing in production-like environment.
- [ ] Data migration (if applicable, existing contracts → new system).
- [ ] Stakeholder training (Inkopers, Controllers, WNT Officers, Auditors).
- [ ] Feature flag: inhuur-derden enabled for pilot group (1-2 cost centers).
- [ ] Monitor: log errors, performance, user feedback.

### Week 28: Expand Rollout
- [ ] Enable for all cost centers (if pilot successful).
- [ ] Collect feedback, iterate on UX.

### Week 30: Stabilization & Ongoing Support
- [ ] Address post-launch issues.
- [ ] Annual WNT export (Nov/Dec 2026).
- [ ] Documentation updates based on real usage.
- [ ] Plan Phase 2 enhancements (e.g., automated tax-form filing, Peppol compliance).

---

## Success Criteria (End of Phase 4)

1. **Functional Completeness**: All 10 features (REQ-DBA, REQ-MOD, REQ-WNT, REQ-VEN, REQ-RAT, REQ-APP, REQ-CLO, REQ-EXP, REQ-AUD, REQ-INT) implemented and tested.

2. **Compliance Readiness**: Contracts are fully traceable from initiation to archival, with reproducible audit trails for tax/wage audits. Zero undetected WNT wage-cap breaches.

3. **User Adoption**: >80% of Inkopers using system for contract creation (vs. manual process). Controllers accessing hiring-ratio dashboard monthly.

4. **Security**: No critical vulnerabilities, all red-DBA approvals properly three-signed, audit log immutable and hash-chain validated.

5. **Performance**: Dashboard <2s, invoice aggregation <100ms, annual export <5s. Scheduled jobs complete within SLA windows.

6. **Support Readiness**: Documentation complete (3 guides + training), team trained, support process defined.
