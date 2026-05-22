# Design: Supplier Onboarding — Vragenlijst & Workflow

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  External APIs                                                  │
│  ├─ KvK-API (Dutch Chamber of Commerce registry)               │
│  ├─ VIES (EU VAT validation)                                   │
│  ├─ Surepay (IBAN name-on-account check)                       │
│  ├─ Peppol-SMP (e-invoicing directory)                         │
│  └─ ERP connectors (AFAS, Unit4, Exact, SAP)                   │
└────────┬────────────────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────────────────┐
│  purchaseq App Layer                                            │
│  ├─ Self-service Supplier Portal                               │
│  │  ├─ Category selection                                      │
│  │  ├─ Dynamic questionnaire (category + risk-tier branching)  │
│  │  ├─ Document upload (bewijslast)                            │
│  │  └─ Validation feedback & progress                          │
│  │                                                             │
│  ├─ Internal Approval Dashboard                                │
│  │  ├─ Role-based approval queue (categorie-inkoper,           │
│  │  │  contractmanager, financien, integrity)                  │
│  │  ├─ Dossier review & decision entry                         │
│  │  └─ Parallel BIBOB task creation                            │
│  │                                                             │
│  ├─ Backend Services                                           │
│  │  ├─ QuestionnaireService (category branching logic)         │
│  │  ├─ ValidationService (KvK, IBAN, VIES, Peppol)            │
│  │  ├─ ApprovalService (role-based routing & state machine)    │
│  │  ├─ BIBOBService (risk assessment integration)              │
│  │  ├─ RevalidationService (annual refresh + email dispatch)   │
│  │  └─ ERP SyncService (push to crediteurenstam)               │
│  │                                                             │
│  └─ Background Jobs                                            │
│     ├─ daily-revalidation-check (mark expired dossiers)        │
│     ├─ send-revalidation-emails (annual prompt to suppliers)   │
│     └─ sync-erp (batch push approved suppliers)                │
└────────┬────────────────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────────────────┐
│  OpenRegister Data Layer                                        │
│  ├─ Leverancier (supplier master data)                         │
│  ├─ OnboardingDossier (intake lifecycle)                       │
│  ├─ VragenlijstAntwoord (questionnaire responses)              │
│  ├─ Bewijsstuk (proof documents, docudesk integration)         │
│  ├─ BIBOBToets (integrity assessments, decidesk integration)   │
│  └─ ApprovalStep (role-based decisions)                        │
└─────────────────────────────────────────────────────────────────┘
```

## Entity Schemas (OpenRegister)

All entities live in OpenRegister under the `purchaseq` register. Cross-references use OpenRegister relations (register+schema+objectId), not foreign keys.

### Leverancier (Supplier)

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "Leverancier",
    "slug": "acme-logistiek"
  },
  "handelsnaam": "ACME Logistics BV",
  "statutaireNaam": "ACME Logistics B.V.",
  "kvkNummer": "34056216",
  "btwNummer": "NL001234567B99",
  "peppolId": "9907:nl001234567b99",
  "rechtsvorm": "besloten vennootschap",
  "vestigingsadres": {
    "straat": "Transportweg 10",
    "huisnummer": "10",
    "postcode": "1234AB",
    "plaats": "Amsterdam",
    "land": "NL"
  },
  "correspondentieadres": {
    "straat": "PO Box 1000",
    "huisnummer": "",
    "postcode": "1234ZZ",
    "plaats": "Amsterdam",
    "land": "NL"
  },
  "iban": "NL91ABNA0417164300",
  "ibanTnv": "ACME Logistics B.V.",
  "contactpersoon": "Jan de Vries",
  "email": "jan.devries@acmelogistics.nl",
  "telefoon": "+31201234567"
}
```

### OnboardingDossier (Supplier Intake Case)

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "OnboardingDossier",
    "slug": "dos-acme-2026-05"
  },
  "leverancier": {
    "register": "purchaseq",
    "schema": "Leverancier",
    "objectId": "uuid-acme"
  },
  "categorie": "logistiek",
  "risicoTier": "laag",
  "status": "ingevuld",
  "uitnodigingsDatum": "2026-05-01",
  "gereeddatum": "2026-05-20",
  "vervaldatum": "2027-05-20",
  "toelichting": "Onboarding intake voor nieuwe logistieke partner."
}
```

### VragenlijstAntwoord (Questionnaire Response)

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "VragenlijstAntwoord",
    "slug": "ans-dossier-acme-q001"
  },
  "dossier": {
    "register": "purchaseq",
    "schema": "OnboardingDossier",
    "objectId": "uuid-dos-acme"
  },
  "vraagCode": "LOG-001-transport-license",
  "vraagTitel": "Heeft u een geldig vervoerlicense?",
  "antwoord": "ja",
  "antwoordType": "bool",
  "validatieStatus": "ok",
  "validatieBron": "handmatig"
}
```

### Bewijsstuk (Proof Document)

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "Bewijsstuk",
    "slug": "bew-acme-kvk"
  },
  "dossier": {
    "register": "purchaseq",
    "schema": "OnboardingDossier",
    "objectId": "uuid-dos-acme"
  },
  "type": "kvk-uittreksel",
  "bestand": {
    "register": "docudesk",
    "schema": "Document",
    "objectId": "uuid-doc-kvk"
  },
  "geldigTot": "2027-05-21",
  "geverifieerd": true,
  "geverifieerd_door": "admin@municipality.nl"
}
```

### BIBOBToets (Integrity Assessment)

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "BIBOBToets",
    "slug": "bibob-acme"
  },
  "dossier": {
    "register": "purchaseq",
    "schema": "OnboardingDossier",
    "objectId": "uuid-dos-acme"
  },
  "aanleiding": "Leverancier in categorie bouw (risico-hoog).",
  "status": "openstaand",
  "behandelaar": "integrity-officer@municipality.nl",
  "conclusie": "",
  "vastgelegd_op": "2026-05-20"
}
```

### ApprovalStep (Role-Based Decision)

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "ApprovalStep",
    "slug": "step-acme-categorie-inkoper"
  },
  "dossier": {
    "register": "purchaseq",
    "schema": "OnboardingDossier",
    "objectId": "uuid-dos-acme"
  },
  "rol": "categorie-inkoper",
  "volgorde": 1,
  "status": "openstaand",
  "beslisser": "buyer@municipality.nl",
  "beslissing": null,
  "motivatie": null,
  "datum": null
}
```

## Seed Data

### Leveranciers

1. **ACME Logistics BV** (logistiek, laag risk)
   - KvK: 34056216 | BTW: NL001234567B99
   - Vestiging: Transportweg 10, 1234AB Amsterdam
   - Contact: jan.devries@acmelogistics.nl | +31201234567

2. **De Zorggroep Utrecht** (zorg, hoog risk)
   - KvK: 41098765 | BTW: NL004567890B88
   - Vestiging: Ziekenhuisplein 5, 3512AB Utrecht
   - Contact: mary.vanderberg@dezorggroep.nl | +31302345678

3. **BuildRight Projecten BV** (bouw, hoog risk)
   - KvK: 27345678 | BTW: NL007890123B77
   - Vestiging: Bouwsteenweg 42, 6800AA Arnhem
   - Contact: piet.construction@buildright.nl | +31261234567

4. **Digitale Diensten NL** (ict, midden risk)
   - KvK: 56123456 | BTW: NL002345678B66
   - Vestiging: Softwarestraat 88, 1097DM Amsterdam
   - Contact: support@digitalservices.nl | +31206789012

5. **Groen Energie Co-op** (dienstverlening, laag risk)
   - KvK: 65432109 | BTW: NL005678901B55
   - Vestiging: Hernieuwbare Energieweg 3, 2131AK Hoofddorp
   - Contact: info@groeenergie.nl | +31204567890

### OnboardingDossiers

- **dos-acme-2026-05**: ACME Logistics, logistiek, laag, ingevuld, expires 2027-05-20
- **dos-zorggroep-2026-05**: De Zorggroep, zorg, hoog, intern_review, expires 2027-05-20
- **dos-buildright-2026-04**: BuildRight, bouw, hoog, goedgekeurd, expires 2027-04-15
- **dos-digitale-2026-05**: Digitale Diensten, ict, midden, uitnodiging, expires 2027-05-20
- **dos-groen-2026-06**: Groen Energie, dienstverlening, laag, ingevuld, expires 2027-06-01

## Workflows

### Supplier Self-Service Flow

```
START
  ├─ [Supplier Portal]
  │  ├─ Register or sign in
  │  ├─ Select category (logistiek, zorg, bouw, ict, dienstverlening, ...)
  │  ├─ Determine risk tier (laag/midden/hoog) from category config
  │  ├─ Load dynamic questionnaire (category + risk tier filtered)
  │  ├─ Supplier completes questions
  │  │  └─ On blur KvK: real-time KvK-API validation
  │  │  └─ On save IBAN: IBAN checksum + Surepay name-check validation
  │  │  └─ On save Peppol ID: Peppol-SMP validation
  │  ├─ Upload proof documents
  │  │  ├─ KvK extract (required)
  │  │  ├─ IBAN proof (required)
  │  │  ├─ VOG (if risk≥midden)
  │  │  ├─ MVI proof (if annual turnover >€50k)
  │  │  └─ Certification docs (ISO, PSO, etc. if applicable)
  │  ├─ Review and submit
  │  └─ Dossier → status: ingevuld
  │
  ├─ [Internal Review Queue]
  │  ├─ Approval workflow starts if all validations pass
  │  ├─ Create sequential ApprovalSteps:
  │  │  1. categorie-inkoper (sourcing fit)
  │  │  2. contractmanager (MVI/conditions)
  │  │  3. financien (IBAN/ERP-ready)
  │  │  + parallel: integriteit (if risk=hoog)
  │  ├─ Dossier → status: intern_review
  │  │
  │  ├─ [Each ApprovalStep]
  │  │  ├─ Role reviewer fetches dossier
  │  │  ├─ Reviews questions, docs, validations
  │  │  ├─ Decision: approve, request-info, or reject
  │  │  ├─ Motivatie & timestamp recorded
  │  │  └─ ApprovalStep → status: afgewerkt
  │  │
  │  ├─ If all steps approve:
  │  │  ├─ Dossier → status: goedgekeurd
  │  │  ├─ Leverancier record created in ERP (if new)
  │  │  ├─ Push to crediteurenstam
  │  │  ├─ Set vervaldatum = today + 12 months
  │  │  └─ Send welcome email to supplier
  │  │
  │  └─ If any step rejects:
  │     ├─ Dossier → status: afgewezen
  │     ├─ Send rejection email with reason
  │     └─ Allow supplier to resubmit (new dossier)
  │
  └─ [Annual Revalidation]
     ├─ Daily job checks for expired dossiers (vervaldatum < today)
     ├─ For each expiring dossier:
     │  ├─ Create revalidation email with 1-click confirm link
     │  ├─ Send to leverancier contact email
     │  │
     │  ├─ [Supplier Response]
     │  │  ├─ If "confirm unchanged" (1-click):
     │  │  │  ├─ Dossier → status: goedgekeurd
     │  │  │  └─ vervaldatum += 12 months
     │  │  │
     │  │  └─ If "confirm with changes":
     │  │     ├─ Open questionnaire in edit mode
     │  │     ├─ Supplier updates answers/docs as needed
     │  │     ├─ Expedited review: only changed questions reviewed
     │  │     ├─ If all pass: status: goedgekeurd, vervaldatum += 12 months
     │  │     └─ If any fail: status: afgewezen
     │  │
     │  └─ If no response after 60 days:
     │     ├─ Dossier → status: verlopen
     │     ├─ ERP: mark leverancier as inactive
     │     └─ Send final notice email
     │
END
```

### Approval Sequence Diagram

```
Dossier status: ingevuld → intern_review

     Categorie-          Contractmanager        Financien        Integriteit
      Inkoper                                                     (parallel if
         │                    │                    │              risk=hoog)
         ├─── review          │                    │                   │
         │    (sourcing fit)  │                    │                   │
         ├─ approve/reject    │                    │                   │
         ├────────────────────┤                    │                   │
         │                    ├─── review          │                   │
         │                    │    (MVI/terms)     │                   │
         │                    ├─ approve/reject    │                   │
         │                    ├────────────────────┤                   │
         │                    │                    ├─── review          │
         │                    │                    │    (IBAN/ERP)      │
         │                    │                    ├─ approve/reject    │
         │                    │                    │                   │
         │                    │                    │                   ├─── BIBOB
         │                    │                    │                   │    assessment
         │                    │                    │                   ├─ conclude
         │                    │                    │                   │
         │                    │                    ├─ ALL APPROVE ──────┤
         │                    │                    │                   │
    status: goedgekeurd ◄─────┴────────────────────┴───────────────────┘
    (push to ERP)
```

## UI Screens & Components

### 1. Supplier Self-Service Portal

**Landing page**: Category selector with descriptions.
- Grid of category cards (logistiek, zorg, bouw, ict, dienstverlening, etc.)
- Each card shows typical requirements (documents, certifications).
- Click to start intake.

**Dynamic Questionnaire**: 
- Section-based layout (company info, certifications, MVI, etc.).
- Questions appear/hide based on category + risk tier.
- Real-time validation on blur for KvK, IBAN, Peppol.
- Error feedback with action (retry, manual review, contact support).
- Progress bar (% questions answered, % docs uploaded).

**Document Upload**:
- Drag-drop zone for each required document type.
- File preview (PDF, images).
- Status badges (pending, verified, expired).
- Link to docudesk for archival & retention.

**Review & Submit**:
- Summary page with all responses and docs.
- Check for completeness.
- Submit button triggers internal review flow.
- Confirmation: "Thank you. Your application is under review."

### 2. Internal Approval Dashboard

**Dossier Queue**:
- Table of all dossiers with status, category, risk tier, assignee, due date.
- Filters: status (openstaand, afgewerkt), rol (my role), category, overdue.
- Sort: by due date, by category, by risk.
- Bulk actions: reassign, expedite, archive.

**Dossier Detail**:
- Header: supplier name, category, risk tier, status timeline.
- Questionnaire responses (read-only, highlighted/dimmed based on role).
- Document gallery with docudesk links.
- Validation summary (KvK match, IBAN match, Peppol, BIBOB if applicable).
- Approval steps timeline (who approved what, when, with what message).

**Approval Form** (per role):
- Current questionnaire + docs read-only.
- Decision radio buttons: Approve | Request Info | Reject.
- Motivatie text field (required).
- Submit button → updates ApprovalStep, chains to next role.
- If reject: email template to supplier with reason.

**Integrity (BIBOB) Panel** (parallel for risk=hoog):
- Link to decidesk for formal BIBOB assessment.
- Risk factors checklist (based on municipal policy).
- Conclusion options: OK | Flagged for Review | Rejected.
- Can proceed to financien in parallel; financien waits for integrity conclusion.

### 3. Coordinator Dashboard

**KPI Summary**:
- Dossiers by status (pie chart): uitnodiging, ingevuld, intern_review, goedgekeurd, afgewezen, verlopen.
- Avg cycle time (intake → approval).
- Overdue dossiers (red flag).
- Annual revalidation status (% confirmed, % pending, % expired).

**Dossier Backlog**:
- Table filtered by status=intern_review.
- Assigned-to column, due date, escalation flag.
- Quick reassign action.
- Bulk archive completed dossiers.

## Integration Points

### openconnector Adapters

1. **KvK-API Adapter**: GET /lookup?kvk=34056216 → {handelsnaam, rechtsvorm, vestigingsadres, sbi_code}
2. **VIES BTW Adapter**: Validate BTW number against EU registry.
3. **Surepay Adapter**: POST /iban-check {iban, name} → {valid, name_match}
4. **Peppol-SMP Adapter**: Query SMP for endpoint details & supported document types.
5. **ERP Adapters** (AFAS, Unit4, Exact, SAP): POST /suppliers {name, kvk, btwNr, iban, email} → supplier_id

### docudesk Integration

- Bewijsstuk records link to docudesk Document objects.
- Retention policies: 7 years (fiscal), 10 years (contract).
- Text extraction for compliance scanning (PII detection).

### decidesk Integration

- BIBOBToets → creates formal decision record in decidesk.
- Integrity coordinator approves/rejects via decidesk UI.
- Decision pushes back to purchaseq dossier.

## Reuse Analysis

Leverages OpenRegister platform:
- **ObjectService**: CRUD for Leverancier, OnboardingDossier, VragenlijstAntwoord, Bewijsstuk, BIBOBToets, ApprovalStep.
- **CnIndexPage + CnDetailPage**: Dossier dashboard and detail views.
- **CnFormDialog**: Auto-generated forms for dossier creation/edit.
- **FileService**: Document upload/download via docudesk link.
- **AuditTrailService**: Full audit trail of questionnaire changes & approvals.
- **TasksController**: Task creation for approval steps & revalidation reminders.

No custom CRUD logic, list/search, or file upload components required.

---

**Status**: Draft — Ready for spec phase  
**Updated**: 2026-05-22
