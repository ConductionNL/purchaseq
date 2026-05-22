---
status: proposed
app: purchaseq
spec: bibob-toetsing-leveranciers
version: 1.0
---

# Implementation Tasks — BIBOB Toetsing Leveranciers en Opdrachtnemers

## Phase 1: Data Model & OpenRegister Setup

### T1.1: Define OpenRegister schemas and seed data

- [ ] Create `lib/Settings/bibob_register.json` (OpenAPI 3.0 + x-openregister extensions)
  - BibobDossier schema with properties (dossiernummer, aanleiding, risico_*, onderzoeksfase, status, conclusie, etc.)
  - BibobBetrokkene schema (type, naam, kvk_nummer, bsn, geboortedatum, woonadres, etc.)
  - BibobFormulier1 schema (dossier_id, ingevuld_door, antwoorden_json, bijgevoegde_stukken, eerste_indicatie)
  - BibobFormulier2 schema (dossier_id, aanvullende_vragen, openbare_bronnen_geraadpleegd, zienswijze_betrokkene, voorlopige_conclusie)
  - LbbAdviesaanvraag schema (dossier_id, aanvraagdatum, aanvraag_tekst, lbb_zaaknummer, advies_*, geldigheidsduur)
  - BibobBesluit schema (dossier_id, besluit_type, feitelijke_grondslag, juridische_grondslag, belangenafweging, proportionaliteitstoets, motivering, ondertekenaar, datum_uitreiking, bezwaartermijn_startdatum)
  - BibobBewaarregel schema (dossier_id, bewaarduur_basis, vernietigingsdatum, anonimisering_geplanned, archivaris_tweede_handtekening, vernietigingslog)
  - All schemas use schema.org vocabulary
  - All encrypted fields (bsn, woonadres) marked with encryption metadata

- [ ] Add seed data (3-5 realistic objects per schema)
  - 3x BibobDossier (bouw/horeca/transport scenarios)
  - 4x BibobBetrokkene (mix of aanvrager, bestuurder, aandeelhouder)
  - 2x BibobFormulier1 with responses
  - 1x BibobFormulier2 with sources
  - 1x LbbAdviesaanvraag
  - Use valid Dutch postcodes (1000–9999 format), realistic KvK numbers, proper names
  - Mark with `@self` envelope and unique slugs for idempotent import

- [ ] Verify schema compliance
  - All property types explicit + required flags set
  - Description field present on all properties
  - No custom property names when schema.org equivalent exists
  - Contact schemas align with vCard (if applicable)

---

### T1.2: RBAC configuration for privacy-separated access

- [ ] Create PropertyRbacHandler configuration
  - BibobBetrokkene.bsn → accessible by [integriteitscoordinator, juridisch-adviseur, gemandateerd-bestuursorgaan] only
  - BibobBetrokkene.woonadres → same
  - BibobFormulier1/2 content → hide from inkopers, show to [integriteitscoordinator+]
  - LbbAdviesaanvraag.advies_tekst → show ONLY to [juridisch-adviseur, gemandateerd-bestuursorgaan]
  - All other BibobDossier fields → visible to all roles (dossier existence, conclusion)

- [ ] Test RBAC enforcement via API test suite
  - Inkoper GET /bibob/dossier/{id} → returns only dossier-number + conclusion, NOT sensitive fields
  - Coordinator GET /bibob/dossier/{id} → returns all fields
  - Unauthorized read attempts logged to auditlog

---

## Phase 2: Risk Detection & Notification

### T2.1: CloudEvent listener for risk detection

- [ ] Register listener for `purchaseq.order.*` CloudEvents (create, update)
  - Filter: geraamde waarde ≥ threshold OR sector in [bouw, horeca, afval, vastgoed, transport] OR company_age < 24 months OR ownership_change < 24 months

- [ ] On match: emit `bibob.risk.detected` event with payload:
  ```json
  {
    "order_id": "INK-2024-XXXX",
    "leverancier_kvk": "12345678",
    "leverancier_naam": "Example BV",
    "risico_factoren": ["sector:bouw", "bedrijfsleeftijd:<24m", "opvallende_prijs"],
    "timestamp": "2024-03-15T10:30:00Z"
  }
  ```

- [ ] Link: purchaseq → bibob CloudEvent channel (verify OpenZaak/decidesk/shillinq connectivity)

---

### T2.2: Notification dispatch & dossier creation link

- [ ] Subscribe to `bibob.risk.detected` → NotificationService
  - Recipient: roles matching [integriteitscoordinator]
  - Message body: leverancier name, risico-factoren, link to order, "Create dossier?" button

- [ ] "Create dossier" button → pre-fill BibobDossier form
  - aanleiding = "aanbesteding"
  - aanleiding_referentie = order-id
  - risico_* flags auto-populated from event
  - Open CnFormDialog for initial dossier metadata entry

---

### T2.3: Test risk-detection workflow end-to-end

- [ ] Create test purchase order with value > threshold + bouw-sector
- [ ] Verify risk-event fires synchronously
- [ ] Verify notification reaches coordinator in test environment
- [ ] Verify dossier-creation form pre-fills correctly

---

## Phase 3: Form-1 & KvK Enrichment

### T3.1: BIBOB Formulier-1 editor component

- [ ] Build CnFormDialog component for BibobFormulier1 schema
  - Five question blocks (financiering, strafbare feiten, schulden, structuur, relaties)
  - Text-area inputs with minimum-length indicators
  - File upload slots (KvK-uittreksel, jaarrekening, financieringsovereenkomsten)
  - First-indicatie radio-selection (geen-twijfel / twijfel / sterke-twijfel)
  - Auto-save on field blur

- [ ] Integrate FileService for document uploads
  - Files stored in OpenRegister via ObjectService.saveObject()
  - References in bijgevoegde_stukken array [{ bestand_id, titel, upload_datum }]

- [ ] Ensure dossier-closure blocker
  - REQ-003-001: can't move purchase order to "voorlopige-gunning" without form-1 + first-indicatie OR explicit exemption

---

### T3.2: KvK-adapter integration for auto-enrichment

- [ ] Call openconnector kvk-adapter on form-1 open:
  ```
  openconnector.searchBusiness(kvk_number) → returns {
    naam, juridische_vorm, oprichting_datum,
    bestuurders: [{ naam, rol, ... }],
    aandeelhouders: [{ naam, percentage, ... }],
    ubo_register: [{ ... }]
  }
  ```

- [ ] Auto-populate BibobBetrokkene.naam + create initial BibobBetrokkene records for bestuurders
  - Coordinator can edit/add/remove betrokkenen
  - Pre-fill richting questionnaire

- [ ] Error handling: if kvk-call fails, show "KvK data unavailable" notice, allow form to proceed

---

### T3.3: Test form-1 submission & first-indicatie

- [ ] Submit form-1 with test responses
- [ ] Verify eerste_indicatie radio is mandatory (can't save without)
- [ ] Verify BibobDossier.onderzoeksfase = "formulier-1", status = "onderzoek"
- [ ] Verify auditlog records timestamp, user, choice

---

## Phase 4: Escalation & Form-2

### T4.1: Escalation-decision interface

- [ ] Build escalation modal (CnDetailPage section or modal per ADR-004)
  - Show first-indicatie result from form-1
  - Display two mutually-exclusive buttons: "Escalate to Form-2" / "Close without escalation"

- [ ] Escalate path:
  - Text field: "Onderbouwing escalatie" (min. 250 chars)
  - Create BibobFormulier2 record
  - Stel BibobDossier.onderzoeksfase = "formulier-2", status = "onderzoek"
  - Auditlog: user + timestamp + onderbouwing

- [ ] Close path:
  - Text field: "Waarom afwezig escalatie" (min. 200 chars)
  - Stel BibobDossier.status = "gesloten", conclusie = "geen-bezwaar"
  - Auditlog: user + timestamp + reasoning

---

### T4.2: BibobFormulier2 editor component

- [ ] Build form-2 CnFormDialog:
  - Aanvullende vragen list (add/remove rows)
  - Openbare bronnen checklist (Faillissementsregister, BIG-register, openbare veroordelingen, media sources) with findings text per source
  - Zienswijze-betrokkene date field + document upload
  - Voorlopige-conclusie radio (geen-bezwaar / lichte-mate / ernstige-mate-van-gevaar)

- [ ] Zienswijze-procedure automation:
  - When form-2 marked "ready for zienswijze" → generate draft letter (template text)
  - Modal to review/edit letter
  - Click "Send" → stel BibobDossier.status = "advies", zienswijze deadline = now + 14 days
  - Zienswijze deadline blocking: can't save final decision until deadline passed or response received (REQ-006-003)

---

### T4.3: Test escalation workflow

- [ ] Form-1 with "twijfel" → escalate to form-2
- [ ] Verify form-2 created, dossier phase updated
- [ ] Verify zienswijze draft-letter generation
- [ ] Verify deadline calculation

---

## Phase 5: LBB Integration

### T5.1: LBB-adviesaanvraag builder

- [ ] Build CnFormDialog for LbbAdviesaanvraag:
  - Aanleiding text (min. 100 chars)
  - Onderzoeksresultaten text (min. 250 chars)
  - Specifieke vragen list (min. 2, add/remove)
  - Betrokkenen table (auto-populated from BibobBetrokkene records for this dossier, editable)
  - Gewenste-antwoord-datum date picker (min. 14 days)
  - Submit validation: all fields required + min-lengths enforced

- [ ] On submit:
  - Compile aanvraag-tekst from all fields
  - Call openconnector/lbb-adapter (or openzaak SOAP endpoint) to send request
  - Stel lbb_zaaknummer + verzending_bevestiging on success response
  - Stel LbbAdviesaanvraag.status = "sent"

- [ ] Error handling: if send fails, show error modal, allow retry

---

### T5.2: LBB-advies receipt & storage

- [ ] Implement webhook receiver for `lbb.advies.received` event:
  ```json
  {
    "lbb_zaaknummer": "LBB-2024-12345",
    "advies_datum": "2024-04-15",
    "advies_conclusie": "ernstige-mate-van-gevaar",
    "advies_tekst": "[encrypted blob]"
  }
  ```

- [ ] On receipt:
  - Find LbbAdviesaanvraag by lbb_zaaknummer
  - Decrypt advies_tekst if necessary (check encryption requirements)
  - Store separately (not inline in LbbAdviesaanvraag): blob storage with reference
  - Stel advies_conclusie, advies_datum
  - Stel parent BibobDossier.onderzoeksfase = "lbb-advies"

- [ ] Access control: PropertyRbacHandler limit advies_tekst read to [juridisch-adviseur, gemandateerd-bestuursorgaan]

- [ ] Test: mock LBB webhook → verify advies stored + access restricted

---

## Phase 6: Decision & Awb Compliance

### T6.1: BIBOB-besluit template & enforcement

- [ ] Build CnAdvancedFormDialog for BibobBesluit:
  - Four textarea blocks with live char-count:
    1. Feitelijke grondslag (min. 250)
    2. Juridische grondslag (min. 50)
    3. Belangenafweging (min. 100)
    4. Proportionaliteitstoets (min. 100)
  - Besluittype radio (gunning / niet-gunning / weigering / intrekking / voorwaardelijk)
  - Validation: submit button disabled until all fields ≥ min-length

- [ ] Show pre-filled summary of dossier:
  - Leverancier name + KvK
  - Aanleiding (order/permit)
  - Risico-factoren flagged
  - Form-1 conclusion + Form-2 conclusion (if applicable)
  - LBB advies (if available, abstract only)

- [ ] "Save as draft" vs "Submit for signature" distinction
  - Draft: saves editable template
  - Submit: locks fields + initiates decidesk workflow

---

### T6.2: Decidesk integration for signature

- [ ] On "Submit for signature":
  - POST to decidesk decision API:
    ```json
    {
      "title": "BIBOB-besluit {dossier-number}",
      "description": "{leverancier} {aanleiding}",
      "body": "[compiled 4-section decision text]",
      "required_signatory_role": "gemandateerd-bestuursorgaan",
      "reference": "bibob:{dossier-number}",
      "deadline": null
    }
    ```
  - Decidesk generates signature request, notifies mandaathouder
  - On completion callback: capture signatory details + signature timestamp
  - Lock BibobBesluit (make read-only)

- [ ] Store linkage: BibobBesluit._decidesk_decision_id for traceability

---

### T6.3: Completeness validation before signature

- [ ] Pre-submit checklist (REQ-014-001):
  - All 4 Awb sections ≥ min-length? ✓
  - Besluittype selected? ✓
  - Onderzoeks-phases complete (form-1 + form-2 + optional LBB)? ✓
  - Zienswijze-termijn expired (if applicable)? ✓
  - Leverancier details present? ✓
  - Fail any check → block with specific error

- [ ] Show pre-submit validation report to bestuurder

---

### T6.4: Test decision workflow

- [ ] Create decision template with all 4 sections
- [ ] Verify min-length enforcement
- [ ] Verify signature submission to decidesk (use mock if needed)
- [ ] Verify auditlog captures signature event

---

## Phase 7: Payment Blocking (shillinq)

### T7.1: shillinq CloudEvent integration

- [ ] On BibobDossier.conclusie change to "ernstige-mate-van-gevaar" (and dossier NOT gesloten):
  - Emit `bibob.decision.payment-block` CloudEvent:
    ```json
    {
      "leverancier_kvk": "{kvk}",
      "dossier_id": "{dossier-number}",
      "reason": "Bibob-onderzoek actief, ernstige mate van gevaar",
      "block_until_date": null
    }
    ```
  - Shillinq listener blocks all outgoing payments to that KvK

- [ ] On conclusion change to "lichte-mate" / "geen-bezwaar" OR dossier gesloten:
  - Emit `bibob.decision.payment-unblock` CloudEvent
  - Shillinq lifts block

- [ ] Test integration with shillinq (coordinate with shillinq team if in same fleet)

---

## Phase 8: Retention & Destruction

### T8.1: BibobBewaarregel automation

- [ ] On BibobDossier.status = "gesloten" save:
  - Auto-create BibobBewaarregel (or update if exists):
    - If LbbAdviesaanvraag.advies_conclusie exists:
      - bewaarduur_basis = "5-jaar-na-onherroepelijk-besluit"
      - start_date = (BibobBesluit.datum_uitreiking + 42 days + 90 days for higher appeal)
    - Else:
      - bewaarduur_basis = "5-jaar-na-sluiting"
      - start_date = BibobDossier.closure_date
    - vernietigingsdatum = start_date + 5 years
    - anonimisering_geplanned = vernietigingsdatum - 90 days

- [ ] Test: verify bewaarduur calculation for multiple scenarios

---

### T8.2: Anonymization workflow

- [ ] Daily job (ArchivalService.prepareForAnonymization):
  - Find all BibobBewaarregel where anonimisering_geplanned <= today
  - For each dossier:
    - Notify archivist: "BIBOB-XXXX scheduled for anonymization on {date}"
    - Archivist clicks "Proceed"
    - Redact: BibobBetrokkene.bsn, woonadres, geboortedatum → "***ANONIMIZED***"
    - Redact: BibobFormulier1/2 sensitive responses
    - DELETE: LbbAdviesaanvraag.advies_tekst
    - Set BibobBewaarregel.anonimisering_geplanned = now, status = "anonymized"
    - Auditlog: ANONYMIZATION_COMPLETE, {dossier-id}

---

### T8.3: Destruction workflow with second signature

- [ ] Daily job (ArchivalService.scheduleForDestruction):
  - Find all BibobBewaarregel where vernietigingsdatum <= today AND status = "anonymized"
  - For each dossier:
    - Notify archivist: "BIBOB-XXXX ready for destruction on {date}. Confirm second signature."
    - Archivist clicks "Confirm destruction"
    - Execute:
      1. DELETE BibobBetrokkene records
      2. DELETE BibobFormulier1/2 records
      3. DELETE LbbAdviesaanvraag record
      4. ANONYMIZE BibobBesluit (feitelijke grondslag → summary)
      5. Append to BibobBewaarregel.vernietigingslog: "Destroyed by {archivist} on {date}. {summary}."
    - Auditlog: DESTRUCTION_COMPLETE, {dossier-id}, signed

- [ ] Verify: post-destruction, no sensitive data in database (query check)

---

## Phase 9: OpenZaak Integration

### T9.1: Zaak-creatie on dossier-creation

- [ ] On BibobDossier.saveObject():
  - POST to OpenZaak:
    ```json
    {
      "zaaktype": "{bibob-zaaktype-url}",
      "identificatie": "{dossier-number}",
      "omschrijving": "BIBOB-onderzoek {leverancier}",
      "toelichting": "{aanleiding + risico-factoren}",
      "zaakstatus": "onderzoek"
    }
    ```
  - Store returned zaak-URI in BibobDossier._openzaak_url

- [ ] Error handling: async call (failure doesn't block bibob-dossier save)

---

### T9.2: Zaak-status sync

- [ ] On BibobDossier state transitions:
  - formulier-1 → zaakstatus = "onderzoek"
  - formulier-2 → zaakstatus = "onderzoek" (still)
  - lbb-advies → zaakstatus = "advies"
  - gesloten → zaakstatus = "gesloten", einddatum = now, resultaat = {conclusie}

- [ ] Test: verify zaak-status tracks dossier-phase

---

## Phase 10: Auditing & Compliance

### T10.1: Audit trail verification

- [ ] Verify AuditTrailService logs automatically for all BibobDossier + BibobFormulier*/BibobBesluit saves
  - Fields logged: user, timestamp, action, before/after snapshot
  - Sensitive fields (BSN) logged as "***" (not full values)

- [ ] Verify auditlog is append-only (no retroactive edits after 24 hours)

- [ ] Test endpoint: GET /bibob/dossier/{id}/audit → returns full trail

---

### T10.2: GDPR compliance verification

- [ ] Verify no sensitive data exported in bulk exports (ImportService/ExportService)
  - If export requested: ask for legitimate purpose, log to auditlog
  - If export includes restricted data: redact before delivery

- [ ] Verify verscherkingsverzoek (inzageverzoek) flow:
  - Data-subject requests access to their own data in dossier
  - Endpoint: GET /bibob/subject-access-request/{bsn}
  - Returns only data relating to that person, anonymized if not data-subject

---

## Phase 11: Testing & Documentation

### T11.1: End-to-end test scenarios

- [ ] Scenario A: Risk-detected → Form-1 → "geen-twijfel" → Close (no escalation)
  - Test: complete workflow from risk-event to dossier-closure
  - Verify: dossier lifecycle, auditlog, retention-schedule calculation

- [ ] Scenario B: Risk-detected → Form-1 → "twijfel" → Form-2 → Zienswijze → Decision → Signature
  - Test: full escalation + awb-compliant process
  - Verify: zienswijze-blocking, decision-completeness-checks, decidesk integration

- [ ] Scenario C: Risk-detected → Form-1 → "sterke-twijfel" → Form-2 → LBB-request → LBB-advice → Decision
  - Test: LBB-integration end-to-end
  - Verify: restricted access to LBB-advies, advies-incorporation in decision

- [ ] Scenario D: Dossier closes → Retention-period elapses → Anonymization → Destruction
  - Test: full archival lifecycle
  - Verify: PII-redaction, second-signature enforcement, indelible logging

---

### T11.2: API test suite (automated)

- [ ] REST API tests for all CRUD operations:
  - POST /bibob/dossier (create)
  - GET /bibob/dossier/{id} (read, with RBAC test)
  - PATCH /bibob/dossier/{id} (update, with status-transition validation)
  - GET /bibob/dossier/{id}/audit (auditlog)

- [ ] CloudEvent delivery tests:
  - Emit `bibob.risk.detected` → verify listener calls notification
  - Emit `lbb.advies.received` → verify advies stored

- [ ] RBAC tests:
  - Inkoper tries to read BibobBetrokkene.bsn → 403
  - Coordinator reads BibobBetrokkene.bsn → 200
  - Unauthorized read logged to auditlog

---

### T11.3: UI tests (if manual testing needed)

- [ ] Risk-detection notification appears on coordinator dashboard
- [ ] Form-1 editor loads, pre-fills KvK data, accepts responses, enforces first-indicatie
- [ ] Escalation modal forces choice + reasoning
- [ ] Form-2 editor allows zienswijze-letter generation
- [ ] Decision template enforces 4-section minimum-length
- [ ] Signature flow integrates with decidesk (mock OK)

---

### T11.4: Compliance documentation

- [ ] Write README documenting:
  - Wat Bibob-process in purchaseq does
  - How to initiate a dossier from risk-alert
  - Role-based access model
  - Retention & destruction schedule
  - How to access auditlog for compliance audits

- [ ] Write ADR on custom escalation-guard logic (if any non-declarative code exists)
  - Reference ADR-031 (schema-declarative business logic)
  - Explain why BibobEscalationGuard (PHP guard class) is needed vs. pure x-openregister-lifecycle

---

## Deduplication Check

- [ ] Reviewed openregister ObjectService, CnFormDialog, CnAdvancedFormDialog, PropertyRbacHandler, FileService, AuditTrailService, ArchivalService, NotificationService
  - **Finding**: All CRUD, form rendering, RBAC, file management, audit, notifications, retention, archival are provided by OpenRegister/platform
  - **No duplication**: No custom service classes for state-machine, CRUD, or notifications required
  - **Custom code only**: BibobEscalationGuard (lifecycle guard), KvK-adapter hand-off (openconnector), Decidesk signature-callback handler, Shillinq payment-block trigger
  - **Recommendation**: Treat custom code as glue-layer only; all core workflows declarative-first

---

## Seed Data Generation Task

- [ ] Load seed data into dev/test environment after schema creation
  - Command: `openspec load-seeds --app bibob --register bibob_register.json`
  - Verify: 3+ dossiers, 4+ betrokkenen, samples of each form, LBB advies, decision present in database
  - This enables meaningful manual testing from day one (non-empty app on install)

---

## Final Acceptance

- [ ] All 8 features (F1–F8 from proposal.md) testable end-to-end
- [ ] All 14 REQ-* specifications satisfied (test per REQ-XXX-NNN)
- [ ] Privacy RBAC enforced (auditlog verify)
- [ ] Awb Article 3:46 motivation enforced (template + min-length)
- [ ] Retention-schedule automated + second-signature enforced
- [ ] Cross-app integrations working (purchaseq risk-event, openconnector KvK, decidesk signature, shillinq payment-block, openzaak zaak-sync)
- [ ] Auditlog complete and append-only
- [ ] GDPR-compatible (no sensitive export, redaction on DPIA requests)
