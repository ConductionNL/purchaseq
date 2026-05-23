# TenderNed Publicatie Adapter — Specifications

**Status:** Specification  
**Version:** 1.0  
**Last Updated:** 2026-05-23

## REQ-TPA-001: Concept Publication Generation and eForms Mapping

**Title:** Generate a concept `tenderned_publicatie` from an existing `aanbesteding` and serialize to eForms 2.0 XML.

**Description:**
The system MUST generate a concept `tenderned_publicatie` from an existing `aanbesteding` object, mapping internal fields (titel, CPV-hoofdcode, geraamde_waarde, procedure, gunningscriteria, perceelindeling, looptijd, plaats_van_uitvoering) to the corresponding eForms BT-fields. The publicatieType and eformsNoticeSubtype are determined by drempelbedrag checks and aanbesteding configuration.

**Scenarios:**

### REQ-TPA-001.S1: Open Above-Threshold Works Notice
**GIVEN** an aanbesteding with:
- hoofd-CPV `45000000-7` (construction works)
- drie percelen (3 lots)
- looptijd 24 maanden
- gunning op EMVI (price + quality)
- geraamde_waarde EUR 7.000.000 (above 2024-2025 drempel 5.538.000)
- procedure "open"

**WHEN** the user clicks "Genereer publicatie" for type `aankondiging_opdracht`

**THEN**
- A concept `tenderned_publicatie` is created with:
  - `publicatieType = "aankondiging_opdracht"`
  - `eformsNoticeSubtype = "16"` (open above-threshold works)
  - `status = "concept"`
  - `eformsXml` contains properly serialized eForms 2.0 XML per SDK 1.13
  - BT-21 (title) populated with aanbesteding.titel in NL and EN
  - BT-262 (main CPV) set to `45000000-7` with checksum validation
  - BT-27 (estimated value) set to EUR 7.000.000
  - BT-105 (procedure type) set to "open" (BT value 1)
  - BT-539 (award criteria) contains two BT-5411 (price weight %) and BT-5421 (quality weight %)
  - BT-137 (lot identifiers) populated with three LotGroup entries
  - BT-36 (duration) set to 24 months
  - BT-727 (place of performance) resolved to NUTS code (e.g., NL310 for Utrecht)

### REQ-TPA-001.S2: Below-Threshold Nationale Aankondiging
**GIVEN** an aanbesteding with:
- geraamde_waarde EUR 80.000
- CPV `33000000-5` (supplies)
- decentraleLeverancier entity
- procedure "open"

**WHEN** the concept is generated

**THEN**
- `publicatieType = "aankondiging_opdracht"` (nationale)
- `eformsNoticeSubtype` is a TenderNed nationale range code (T01–T30), not an EU subtype
- Drempel check determines this is nationale (EUR 80k < 221k decentrale supplies drempel)
- No tedPublicatieId will be assigned (only nationale publication)

### REQ-TPA-001.S3: Concession Notice
**GIVEN** an aanbesteding with:
- `concessie = true`
- geraamde_waarde EUR 10.000.000
- procedure "open"

**WHEN** concept generation runs

**THEN**
- `publicatieType = "concessie"`
- `eformsNoticeSubtype = "25"` (concession notice)
- Drempel check uses EUR 5.538.000 (concession drempel, same as works)
- Validation confirms EUR 10.000.000 > drempel, notice is above-threshold

### REQ-TPA-001.S4: Sociale Specifieke Diensten
**GIVEN** an aanbesteding with:
- `sociale_specifieke_diensten = true`
- geraamde_waarde EUR 600.000
- CPV within Bijlage XIV (social services codes)

**WHEN** concept generation runs

**THEN**
- `publicatieType = "sociale_specifieke_diensten"`
- `eformsNoticeSubtype = "27"` (social and specific services notice)
- Drempel check uses EUR 750.000 (sociale-sector drempel per AW2012 deel 3)
- Validation confirms EUR 600.000 < 750.000, so lighter regime applies

### REQ-TPA-001.S5: Speciale-Sector Aanbesteding
**GIVEN** a speciale-sector-aanbesteder (water authority) with:
- `speciale_sector = "water"`
- geraamde_waarde EUR 500.000 for services
- procedure "niet_openbaar"

**WHEN** concept generation runs

**THEN**
- Drempel check uses speciale-sector services drempel (EUR 443.000 for 2024-2025)
- EUR 500.000 > 443.000, so above-threshold
- `eformsNoticeSubtype = "18"` (open above-threshold speciale sector)
- BT-729 (speciale sector type) set to "water"

---

## REQ-TPA-002: eForms 2.0 Validation (XSD, Schematron, TN-Extension)

**Title:** Validate concept publicaties against eForms 2.0 schema and business rules.

**Description:**
The system MUST serialize concept publicaties to eForms 2.0 SDK-compliant XML and validate them locally against (a) the EU TED-XSD schema for the chosen subtype, (b) the EU Schematron rules pack (eForms business rules), and (c) the TenderNed-specific extension rules before allowing submission.

**Scenarios:**

### REQ-TPA-002.S1: Successful XSD and Schematron Validation
**GIVEN** a concept `tenderned_publicatie` with:
- `eformsNoticeSubtype = "16"` (open above-threshold works)
- All mandatory BT-fields populated (BT-21 title, BT-262 CPV, BT-27 value, BT-105 procedure, etc.)
- `validatieResultaten = []` (no prior errors)

**WHEN** the user requests validation (clicks "Valideer")

**THEN**
- System produces eForms XML conforming to SDK 1.13
- Xerces XSD validation runs against `eForms-2.0.{noticeSubtype}.xsd`
- Saxon-HE runs Schematron rules pack; all BR-* rules pass
- System returns `validatieResultaten = []` (no errors or warnings)
- Status remains `concept`, ready for document upload
- UI shows green checkmark "Gevalideerd"

### REQ-TPA-002.S2: Missing Mandatory BT-Field
**GIVEN** a `tenderned_publicatie` missing BT-21 (title in official language)

**WHEN** validation runs

**THEN**
- Schematron rule BR-OPT-13 fires: "Procurement project must have a title in the official language"
- `validatieResultaten` array receives entry:
  ```json
  {
    "ruleId": "BR-OPT-13",
    "severity": "error",
    "message": "Procurement project must have a title in the official language",
    "xpath": "/cbc:Title/cbc:NameType[@languageID='NL']"
  }
  ```
- Status remains `concept`
- UI surfaces error with deep link to "Titel (NL)" in aanbesteding object
- User must edit aanbesteding, regenerate publicatie, and revalidate

### REQ-TPA-002.S3: TenderNed-Specific Extension Rule (Motivering Niet-Opdelen)
**GIVEN** a `tenderned_publicatie` with:
- `publicatieType = "aankondiging_opdracht"`
- Geen perceelindeling (niet opgedeeld)
- geraamde_waarde EUR 1.500.000

**WHEN** TenderNed-extension validation runs

**THEN**
- TenderNed rule TN-EXT-001 fires: "Opdrachten > EUR 1.000.000 niet opgedeeld vereisen motivering per AW2012 art 2.59"
- If `aanbesteding.motivering_niet_opdelen` is empty:
  - `validatieResultaten` entry with severity "warning" is recorded
  - UI shows yellow warning with deep link to aanbesteding.motivering_niet_opdelen
- Submission is still allowed (warning, not error), but logged for audit
- If `aanbesteding.motivering_niet_opdelen` is populated:
  - No warning is recorded
  - Submission is allowed

### REQ-TPA-002.S4: CPV Check-Digit Validation
**GIVEN** a `tenderned_publicatie` with BT-262 set to CPV "30000000-2" (invalid check digit, should be "30000000-0")

**WHEN** validation runs

**THEN**
- XSD validation catches invalid check digit and returns error
- `validatieResultaten` entry:
  ```json
  {
    "ruleId": "XSD-CPV-CHECK",
    "severity": "error",
    "message": "CPV code 30000000-2 has invalid check digit; expected 30000000-0",
    "xpath": "/cac:MainClassificationCode"
  }
  ```
- Status remains `concept`
- UI shows error with suggestion to correct CPV

### REQ-TPA-002.S5: Multi-Language Title Validation
**GIVEN** a `tenderned_publicatie` with BT-21 title populated in NL only, but not in EN (required for above-threshold notices per eForms 2.0 rules)

**WHEN** validation runs for `eformsNoticeSubtype = "16"` (above-threshold)

**THEN**
- Schematron rule BR-OPT-13 fires again for EN missing
- `validatieResultaten` entry with severity "error"
- User must add EN title (or set EN to same as NL if no translation available)

---

## REQ-TPA-003: Document Upload with Resumable Sessions and Checksum Verification

**Title:** Upload supporting documents to TenderNed with chunking, progress reporting, and SHA-256 verification.

**Description:**
The system MUST upload `tenderned_bijlage` objects to TenderNed via multipart-document endpoints, support files up to 200 MB, report upload progress to the UI, compute and verify SHA-256 checksum end-to-end, and retry idempotently on 5xx errors using the openconnector retry envelope.

**Scenarios:**

### REQ-TPA-003.S1: Successful Large File Upload with Progress
**GIVEN** a `tenderned_bijlage` of 150 MB with:
- `documentType = "bestek"`
- `mimeType = "application/pdf"`
- bestandsnaam `bestek-werken.pdf`
- No prior upload attempt

**WHEN** the user clicks "Upload" from the publicatie detail

**THEN**
- System streams file in 4 MB chunks (150 MB ÷ 4 MB = ~37 chunks)
- After each chunk upload, system emits progress event:
  - Chunk 1: 4/150 MB = 3%
  - Chunk 2: 8/150 MB = 5%
  - ... (every chunk reported)
  - Chunk 37: 150/150 MB = 100%
- UI progress bar updates in real-time for each chunk
- System computes SHA-256 incrementally as chunks stream
- On all chunks sent, system calls POST /documents finalization endpoint with:
  ```json
  {
    "documentId": "TNB-2026-987654",
    "uploadSessionToken": "sess-2026-001",
    "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "fileSize": 157286400
  }
  ```
- TenderNed returns confirmation with `checksumVerified = true`
- `tenderned_bijlage` status transitions to `completed`
- `tendernedBijlageId` is set
- UI shows green checkmark "Geüpload"

### REQ-TPA-003.S2: Checksum Mismatch
**GIVEN** a 50 MB file with local SHA-256 computed as ABC123...

**WHEN** TenderNed finalization returns SHA-256 as XYZ789... (different)

**THEN**
- System detects mismatch and updates `tenderned_bijlage.uploadStatus = "failed_checksum_mismatch"`
- Retry logic engages: system re-uploads the file (max 3 retries)
- After 3 failed retries, status transitions to `failed`
- UI shows error: "Uploaden mislukt: checksum mismatch na 3 pogingen. Controleer bestand en probeer opnieuw."
- User can manually retry

### REQ-TPA-003.S3: HTTP 503 During Upload (Resume from Checkpoint)
**GIVEN** a 100 MB file that fails at chunk 15 (60 MB uploaded) with HTTP 503

**WHEN** openconnector retry logic engages

**THEN**
- openconnector captures the upload session token and last-acknowledged-chunk
- Exponential backoff waits 1s, then retries
- Resume request includes:
  ```
  POST /documents/{documentId}/resume
  uploadSessionToken: sess-2026-001
  resumeFromByte: 62914560 (60 MB = 60 * 1024 * 1024)
  ```
- TenderNed resumes from byte 62914560, not from byte 0
- Remaining 40 MB chunks upload successfully
- SHA-256 finalization succeeds
- `tenderned_bijlage.uploadStatus = "completed"`

### REQ-TPA-003.S4: Confidential Document Upload
**GIVEN** a `tenderned_bijlage` with:
- `documentType = "gunningsleidraad"`
- `vertrouwelijk = true` (confidential, only visible after bid opening)

**WHEN** upload is initiated

**THEN**
- System calls POST /documents with:
  ```json
  {
    "documentType": "gunningsleidraad",
    "confidentialityLevel": "RESTRICTED"
  }
  ```
- TenderNed marks document as confidential
- Document is not exposed via public TenderNed dossier URL
- Document is only visible to bidders after opening date

### REQ-TPA-003.S5: Multiple Languages
**GIVEN** a publicatie with two bestek versions:
- `bestek-nl.pdf` with `taal = "NL"`
- `bestek-en.pdf` with `taal = "EN"` (English summary per AW2012 art 2.78)

**WHEN** both documents are uploaded

**THEN**
- Both `tenderned_bijlage` objects are created with their respective language
- Both upload successfully
- BT-322 (language) in the eForms XML lists both NL and EN
- Both documents are linked in the publicatie.bijlagen array

---

## REQ-TPA-004: Publication Submission and Status Polling

**Title:** Submit validated publicaties to TenderNed and reconcile status transitions.

**Description:**
The system MUST submit validated publicaties to TenderNed, capture the returned `tendernedPublicatieId` plus (for boven-drempel) the `tedPublicatieId`, transition the publicatie status from `ingediend` to `gepubliceerd` once TenderNed confirms publication is live, and record the `publicatieDatum` from the TenderNed response (not local clock).

**Scenarios:**

### REQ-TPA-004.S1: Successful Submission and Publication
**GIVEN** a `tenderned_publicatie` with:
- `status = "concept"`
- `validatieResultaten = []` (no errors)
- All required bijlagen uploaded
- User has goedkeuring (above-threshold publicatie already approved by mandaathouder)

**WHEN** the user clicks "Indienen bij TenderNed"

**THEN**
- System POSTs the eForms XML to TenderNed /notices endpoint
- TenderNed responds with HTTP 202 Accepted:
  ```json
  {
    "correlationId": "550e8400-e29b-41d4-a716-446655440000",
    "statusUrl": "/notices/550e8400-e29b-41d4-a716-446655440000/status"
  }
  ```
- System immediately updates:
  - `status = "ingediend"`
  - `correlatieId = "550e8400-e29b-41d4-a716-446655440000"`
- UI shows "Ingediend bij TenderNed (wacht op publicatie)"
- System starts polling /notices/{correlatieId}/status every 30 seconds
- After ~90 seconds, TenderNed returns status PUBLISHED:
  ```json
  {
    "status": "PUBLISHED",
    "publicationId": "TN-2026-1234567",
    "tedPublicationId": "2026/S 123-456789",
    "publicationDate": "2026-05-23T14:30:00Z",
    "dossierUrl": "https://www.tenderned.nl/tdc/servlet/DCD?cIid=1234567"
  }
  ```
- System updates:
  - `status = "gepubliceerd"`
  - `tendernedPublicatieId = "TN-2026-1234567"`
  - `tedPublicatieId = "2026/S 123-456789"` (if boven-drempel)
  - `publicatieDatum = "2026-05-23T14:30:00Z"`
  - `tendernedDossierUrl = "https://www.tenderned.nl/tdc/servlet/DCD?cIid=1234567"`
- Base `aanbesteding` record gets:
  - `actievePublicatieId = "550e8400-e29b-41d4-a716-446655440000"`
  - Entry added to `publicatieHistorie[]`
- UI shows "Gepubliceerd op TenderNed" with clickable dossier link
- Notification: "Uw aanbesteding is gepubliceerd op TenderNed"

### REQ-TPA-004.S2: Publication Rejected by TenderNed
**GIVEN** a submission that returns status REJECTED:
```json
{
  "status": "REJECTED",
  "errorCode": "INVALID_CPV_HIERARCHY",
  "message": "CPV code 99999999-9 is not valid",
  "violations": [
    {
      "field": "BT-262",
      "message": "Main CPV must be a valid 9-digit code with check digit"
    }
  ]
}
```

**WHEN** polling detects the REJECTED status

**THEN**
- System updates:
  - `status = "concept"` (reverted)
  - `validatieResultaten` array gets new entry:
    ```json
    {
      "ruleId": "TN-INVALID_CPV",
      "severity": "error",
      "message": "CPV code 99999999-9 is not valid",
      "xpath": "/cac:MainClassificationCode",
      "source": "TenderNed"
    }
    ```
- Polling stops
- UI shows "Publicatie geweigerd: CPV-code ongeldig"
- Deep link to aanbesteding.cpv_main_code is shown
- Notification: "Uw publicatie is geweigerd door TenderNed. Controleer het CPV-code."
- User can fix the aanbesteding and regenerate publicatie

### REQ-TPA-004.S3: Polling Timeout
**GIVEN** a submission with correlationId that TenderNed never returns a final status for

**WHEN** polling runs for 10 minutes without receiving PUBLISHED or REJECTED

**THEN**
- System stops polling and logs warning
- `status` remains `"ingediend"` (not reverted)
- UI shows "Publicatie ingediend, wacht nog op bevestiging"
- Manual "Controleer status" button is shown
- User can manually click to check status again
- Ops alert is raised if status stays in "ingediend" for >24 hours

### REQ-TPA-004.S4: Nationale Aankondiging (No tedPublicatieId)
**GIVEN** a `tenderned_publicatie` with:
- `publicatieType = "aankondiging_opdracht"` (nationale)
- `eformsNoticeSubtype` in range T01-T30 (not EU subtype)

**WHEN** submission succeeds and TenderNed returns status PUBLISHED

**THEN**
- System updates:
  - `status = "gepubliceerd"`
  - `tendernedPublicatieId = "TN-2026-1234567"`
  - `tedPublicatieId` remains NULL (no TED publication for nationale)
  - `publicatieDatum` set from TN response

---

## REQ-TPA-005: Vragen-en-Antwoorden Round-Trip and Nota van Inlichtingen

**Title:** Poll TenderNed for incoming questions, expose them for response, and publish answers as a consolidated notice.

**Description:**
The system MUST poll TenderNed for incoming `vragen` during the inlichtingenperiode, expose them in the purchaseq UI for response composition, allow batched publishing of `antwoorden` as a nota van inlichtingen, and republish the consolidated note as a document update to the original publicatie.

**Scenarios:**

### REQ-TPA-005.S1: Poll and Create Vragen Records
**GIVEN** a published `tenderned_publicatie` with:
- `status = "gepubliceerd"`
- `sluitingsDatum = "2026-07-15T23:59:59Z"`
- Current date is 2026-05-25 (within inlichtingenperiode)

**WHEN** the n8n scheduled poll job runs (every 15 minutes)

**THEN**
- System calls TenderNed GET /notices/{tendernedPublicatieId}/questions?since={lastPollTime}
- TenderNed returns:
  ```json
  {
    "questions": [
      {
        "questionId": "Q-2026-001",
        "question": "Kan onderdeel 3.2 ook in kunststof?",
        "askedAt": "2026-05-25T09:15:00Z"
      }
    ]
  }
  ```
- System creates `tenderned_vraag_antwoord` record:
  - `vraagnummer = 1` (auto-incremented)
  - `vraag = "Kan onderdeel 3.2 ook in kunststof?"`
  - `vraagstelDatum = "2026-05-25T09:15:00Z"`
  - `status = "open"`
  - `laat_ontvangen = false` (date is >6 days before sluitingsDatum)
- Notification fires to inkoper: "Nieuwe vraag #1 op uw aanbesteding"
- UI shows new vraag in the publicatie detail under "Vragen & Antwoorden" tab
- Inkoper can click to view and compose an antwoord

### REQ-TPA-005.S2: Late-Received Question (Within 6 Days of Deadline)
**GIVEN** same scenario but vraag arrives on 2026-07-10 (5 days before sluitingsDatum)

**WHEN** poll job creates the vraag record

**THEN**
- System checks: sluitingsDatum - vraagstelDatum = 5 days (< 6 days)
- Sets `laat_ontvangen = true`
- Notification includes warning: "Let op: deze vraag is laat ingediend. Per AW2012 art 2.65 lid 3 kan verlenging van de deadline nodig zijn."
- UI highlights this vraag with yellow warning badge
- Inkoper must review with juridisch adviseur whether deadline extension is required

### REQ-TPA-005.S3: Compose and Publish Nota van Inlichtingen
**GIVEN** three `tenderned_vraag_antwoord` records with status "open"

**WHEN** inkoper marks all three "Klaar" and clicks "Publiceer nota van inlichtingen"

**THEN**
- System generates a PDF documento with:
  - Header: Aanbesteding titel, TenderNed dossier ID
  - Table with:
    - Vraag #1: [anonymised question text]
    - Antwoord #1: [inkoper's response]
    - ... (all three Q&A pairs)
  - Footer: Publication date, signature placeholder
- System creates new `tenderned_bijlage`:
  - `documentType = "nota_van_inlichtingen"`
  - `bestandsnaam = "nota-inlichtingen-{date}.pdf"`
  - File is uploaded to TenderNed
- System checks if any antwoord materially changed the bestek (compared to original):
  - If changes detected (e.g., antwoord clarifies procedure change):
    - System automatically creates and submits a rectificatie-aankondiging
    - User is notified: "Materialele wijziging gedetecteerd. Rectificatie-aankondiging is ingediend."
- Updates vraag statuses: all three transition to `verwerkt_in_nota`
- Updates `notaVanInlichtingenRef` for each vraag to point to the NvI document
- Notification: "Nota van inlichtingen gepubliceerd"

### REQ-TPA-005.S4: Multiple Rounds (FAQ Document)
**GIVEN** antwoorden from round 1 and round 2 (poll ran twice, added more vragen)

**WHEN** inkoper publishes nota with all Q&A pairs

**THEN**
- Single consolidated PDF is generated with all questions and answers from all rounds
- Document is uploaded once; previous NvI is not replaced, but a new version is created
- `tenderned_bijlage` records for both versions exist in the publicatie
- User can view version history

---

## REQ-TPA-006: Award Notice and Post-Award Modifications

**Title:** Generate award notices and support post-award modification publications.

**Description:**
The system MUST publish `aankondigingen van gegunde opdracht` within 30 calendar days of contract signature, support modification notices for in-contract changes (scope, value, term per AW2012 art 2.163a-g), and link them back to the original `opdrachtaankondiging` via the `tendernedPublicatieId` chain.

**Scenarios:**

### REQ-TPA-006.S1: Generate and Submit Award Notice
**GIVEN** a contract object with:
- `contractDate = "2026-05-20"` (signature date)
- `winnaar = { kvk: "12345678", naam: "Bouw BV", adres: "Kerkstraat 1, Amsterdam" }`
- `gunningsBedrag = EUR 8.500.000`
- `aantalInschrijvingen = 7`
- Lowest tender: EUR 8.200.000
- Highest tender: EUR 9.100.000
- Reference to original `tenderned_publicatie` with tendernedPublicatieId "TN-2026-1234567"

**WHEN** user clicks "Genereer gunningsaankondiging" from contract detail

**THEN**
- System creates new `tenderned_publicatie`:
  - `publicatieType = "aankondiging_gunning"`
  - `eformsNoticeSubtype = "26"` (award notice)
  - `status = "concept"`
  - eForms XML pre-filled with:
    - BT-21 (title): "Aankondiging van gegunde opdracht voor [originele titel]"
    - BT-195 (winner name): "Bouw BV"
    - BT-196 (winner KvK): "12345678"
    - BT-194 (contract value): EUR 8.500.000
    - BT-710 (number of tenders): 7
    - BT-711 (lowest tender): EUR 8.200.000
    - BT-712 (highest tender): EUR 9.100.000
    - BT-718 (link to original notice): correlates to TN-2026-1234567
- Submission deadline reminder: "30 dagen voor indiening remain" (calculated as contractDate + 30 days)
- UI shows "Gunningsaankondiging gereed voor goedkeuring"
- Inkoopcoördinator receives notification for approval

### REQ-TPA-006.S2: Award Notice for Multi-Lot Tender
**GIVEN** a multi-perceel aanbesteding with 3 percelen:
- Perceel 1: awarded to Bouw BV
- Perceel 2: awarded to Infra NL
- Perceel 3: not awarded (insufficient suitable offers)

**WHEN** gunningsaankondiging is generated

**THEN**
- System creates award notice covering only percelen 1 and 2
- Perceel 3 is marked with:
  - `status = "niet_gegund"`
  - `reason = "geen_geschikte_inschrijving"` (or alternative reason from enum)
  - Reason is documented in the eForms XML
- Notification: "Perceel 3 niet gegund. U kunt een herhaalde aanbesteding starten voor perceel 3."
- Optional workflow: user can choose to re-tender perceel 3 separately

### REQ-TPA-006.S3: In-Contract Wijziging (Scope Change)
**GIVEN** a contract in execution with:
- Original contract value: EUR 1.000.000
- New scope change: add 5 additional items
- New contract value: EUR 1.080.000 (8% increase)

**WHEN** user clicks "Voeg wijziging toe" and drafts wijzigingsaankondiging

**THEN**
- System creates new `tenderned_publicatie`:
  - `publicatieType = "aankondiging_wijziging"`
  - `eformsNoticeSubtype = "29"` (modification notice)
  - eForms XML includes:
    - Reference to original contract notice via BT-718
    - BT-163 (original contract value): EUR 1.000.000
    - BT-164 (change description): "Toevoeging van 5 vervolgitems"
    - BT-165 (new contract value): EUR 1.080.000
    - BT-166 (change amount): EUR 80.000
    - BT-167 (change percentage): 8%
- System checks AW2012 art 2.163b (cumulative threshold):
  - For supplies/services: 10% cumulative threshold
  - For works: 15% cumulative threshold
  - Original contract was supplies, so 10% threshold = EUR 100.000
  - Current change is 8%, which is < 10%, so single wijziging is under threshold
  - If prior wijzigingen exist, check cumulative sum
- Validation passes, wijziging is ready for submission
- Inkoopcoördinator approves, system submits to TenderNed

### REQ-TPA-006.S4: Wijziging Exceeds Cumulative Threshold
**GIVEN** a contract with:
- Original value: EUR 1.000.000 (supplies)
- Prior wijziging 1: EUR 50.000 (5%)
- Prior wijziging 2: EUR 40.000 (4%)
- Cumulative so far: 9%
- New wijziging requested: EUR 15.000 (1.5%)

**WHEN** system validates the new wijziging

**THEN**
- System calculates cumulative: 5% + 4% + 1.5% = 10.5%
- AW2012 art 2.163b supplies threshold is 10%
- System blocks submission and shows error:
  - "Cumulatieve wijzigingen (10.5%) overschrijden de 10%-drempel voor leveringen per AW2012 art 2.163b"
  - "U moet extra motivering toevoegen of deze wijziging samenbundelen met andere wijzigingen"
- User must:
  - Add `aanbesteding.motivering_wijziging_drempel` field explaining why exceeding threshold is justified, OR
  - Defer this wijziging and bundle it with others for a combined submission

---

## REQ-TPA-007: Status Synchronization and Reconciliation

**Title:** Run daily reconciliation comparing local and TenderNed-side publicatie statuses.

**Description:**
The system MUST run a daily reconciliation job comparing local `publicatie` statuses against TenderNed-side statuses, flag and surface any drift (lokaal `gepubliceerd` but TenderNed `withdrawn`, or vice versa), and auto-correct where the TenderNed side is authoritative.

**Scenarios:**

### REQ-TPA-007.S1: External Withdrawal by TenderNed Admin
**GIVEN** a local `tenderned_publicatie` with:
- `status = "gepubliceerd"`
- `tendernedPublicatieId = "TN-2026-1234567"`
- Last reconciliation: 2026-05-22

**WHEN** reconciliation job runs on 2026-05-23

**THEN**
- System calls TenderNed GET /notices/TN-2026-1234567/status
- TenderNed returns `status = "withdrawn"` (withdrawn by TN admin due to suspected fraud alert)
- System detects drift: local="gepubliceerd" but TN="withdrawn"
- System updates:
  - `status = "ingetrokken"`
  - Audit log entry: "Status updated from gepubliceerd to ingetrokken (reason: external_withdrawal, source: TenderNed reconciliation)"
- Notification sent to inkoper: "Uw publicatie TN-2026-1234567 is ingetrokken op TenderNed"
- Ops log entry: "Drift detected and auto-corrected: TN-2026-1234567"

### REQ-TPA-007.S2: Legal Position Divergence (Critical Alert)
**GIVEN** a local `tenderned_publicatie` with:
- `status = "ingetrokken"` (user withdrew via purchaseq UI)
- `tendernedPublicatieId = "TN-2026-1234567"`

**WHEN** reconciliation queries TenderNed and finds status="published" (still live on TN)

**THEN**
- System detects critical divergence: legal positions are misaligned
  - Authority believes it withdrew (status="ingetrokken")
  - Bidders see an active call on TenderNed (status="published")
  - Bids may still be submitted, creating legal liability
- System raises CRITICAL alert:
  - Email to CISO / ops: "CRITICAL: Legal position divergence detected for TN-2026-1234567. Local status ingetrokken, but TenderNed shows published. Manual intervention required."
  - Ops dashboard shows red alert
- System does NOT auto-correct (too risky; requires human judgment)
- Suggested action: contact PIANOo / TenderNed support to force withdrawal on TN side

### REQ-TPA-007.S3: TenderNed Unreachable for 24+ Hours
**GIVEN** reconciliation job cannot reach TenderNed API for 24+ hours

**WHEN** job continues running nightly

**THEN**
- First attempt (day 1): retry with exponential backoff; log warning
- Subsequent attempts (day 2+): after 24h of unavailability, stop polling
- All publicaties keep their last-known local statuses (do NOT update or assume TN status)
- Ops alert raised: "TenderNed API unreachable for 24+ hours. All publicaties remain at last-known status. Check TenderNed service status."
- When TenderNed becomes reachable again, resume polling from last checkpoint

---

## REQ-TPA-008: Drempelbedrag Validation and Procedure Constraints

**Title:** Validate procedure selection against drempelbedrag and enforce AW2012 procedure rules.

**Description:**
The system MUST validate that the gekozen procedure matches the geraamde_waarde against the active `drempelbedrag_periode`, distinguish centrale overheid vs decentrale overheid vs speciale-sector-aanbesteders, and refuse submission for combinations that violate AW2012.

**Scenarios:**

### REQ-TPA-008.S1: Meervoudig Onderhands Above-Threshold (Violation)
**GIVEN** a centrale overheidsentiteit with:
- `geraamde_waarde = EUR 250.000` (for services)
- `procedure = "meervoudig_onderhands"` (3-party negotiated)
- Active drempelbedrag: centrale diensten EUR 143.000 (above-threshold)

**WHEN** publicatie concept is generated

**THEN**
- System checks procedure rules: for above-threshold diensten, only "open" or "niet_openbaar" are allowed
- "Meervoudig_onderhands" is NOT permitted above-threshold for centrale overheid
- Validation error: "Procedure 'meervoudig onderhands' is niet toegestaan voor diensten boven de Europese drempel (EUR 143.000). Kies 'open' of 'niet openbaar'."
- Status remains `concept`, submission is blocked
- Corrigerend voorstel is shown: "open" of "niet_openbaar"

### REQ-TPA-008.S2: Speciale-Sector Threshold Applied
**GIVEN** a speciale-sector-aanbesteder (water authority) with:
- `geraamde_waarde = EUR 400.000` (for supplies)
- `procedure = "open"`
- Active drempelbedrag speciale-sector supplies: EUR 443.000

**WHEN** publicatie concept validation runs

**THEN**
- System detects `speciale_sector = "water"`
- Applies speciale-sector drempel (EUR 443.000), not klassieke sector drempel (EUR 221.000)
- EUR 400.000 < EUR 443.000, so below-threshold
- `eformsNoticeSubtype` is set to nationale range (T01-T30), NOT EU subtype
- Validation passes; submission is allowed
- Notification: "Deze aanbesteding is geclassificeerd als onderdrempel voor speciale-sector leveringen."

### REQ-TPA-008.S3: Sociale Specifieke Diensten (Verlichted Regime)
**GIVEN** an aanbesteding with:
- `sociale_specifieke_diensten = true`
- CPV codes in Bijlage XIV (social services)
- `geraamde_waarde = EUR 600.000`
- `procedure = "onderhandse_procedure"` (simplified)

**WHEN** publicatie validation runs

**THEN**
- System recognizes sociale specifieke diensten (AW2012 deel 3, verlichted regime)
- Applies sociale-sector drempel: EUR 750.000
- EUR 600.000 < EUR 750.000, so below-threshold
- Verlichted regime allows "onderhandse_procedure" (not normally allowed below-threshold)
- Validation passes
- eForms notice subtype is set to "27" (social services notice)
- Notification: "Deze aanbesteding valt onder het verlichte regime voor sociale en specifieke diensten (AW2012 deel 3)."

### REQ-TPA-008.S4: Procedure Mismatch: Enkelvoudig Onderhands Above-Threshold
**GIVEN** a gemeente with:
- `geraamde_waarde = EUR 200.000` (for works)
- `procedure = "enkelvoudig_onderhands"` (single-negotiated)
- Drempelbedrag werken: EUR 5.538.000
- EUR 200.000 < EUR 5.538.000, so below-threshold

**WHEN** concept is generated

**THEN**
- System checks: EUR 200.000 < 5.538.000, so below-threshold
- For below-threshold, "enkelvoudig_onderhands" is permitted
- Validation passes
- No EU-level publication required (but may publish on TenderNed as transparantie notice if gemeente policy requires)

---

## REQ-TPA-009: CPV and NUTS Code Lookup and Validation

**Title:** Provide typeahead CPV search, validate CPV check-digits, validate NUTS-codes.

**Description:**
The system MUST provide a typeahead-search CPV lookup with hierarchical context, validate the CPV check-digit, support multi-CPV (one `hoofd` + meerdere `bij`-CPV), validate NUTS-codes against the active NUTS-NL register, and reject invalid combinations.

**Scenarios:**

### REQ-TPA-009.S1: CPV Typeahead Search with Hierarchy
**GIVEN** aanbesteding detail page with "CPV-code" field

**WHEN** user types "kant" in the CPV search box

**THEN**
- Typeahead fires and searches `cpv_code` table for matches in label_nl
- Results include:
  - "30000000-9 Kantoor- en computermachines" (level 1)
  - "30100000-0 Kantoormachines en -benodigdheden" (level 2, parent: 30000000-9)
- UI displays results with breadcrumb showing hierarchy:
  - "Kantoor- en computermachines / Kantoormachines"
- User clicks one to select; field is populated with the full code

### REQ-TPA-009.S2: CPV Check-Digit Validation
**GIVEN** user manually pastes CPV "30000000-2" (invalid check digit; should be "30000000-0")

**WHEN** user leaves the field or submits form

**THEN**
- System validates check-digit algorithm per EC Regulation 2195/2002
- Detects: check digit "2" is incorrect for "30000000"
- Validation error: "Ongeldig controlegetal voor CPV-code 30000000-2. Bedoelde u 30000000-0?"
- Suggestion is shown with clickable link to auto-correct

### REQ-TPA-009.S3: Multi-CPV Selection
**GIVEN** aanbesteding for IT services with multiple sub-categories

**WHEN** user selects:
- Hoofd-CPV: "72000000-8" (IT services)
- Bij-CPV: "72200000-1" (Software development)
- Bij-CPV: "72300000-2" (IT consulting)

**THEN**
- System stores:
  - `cpv_main_code = "72000000-8"`
  - `cpv_supplementary_codes = ["72200000-1", "72300000-2"]`
- Validation confirms:
  - Hoofd-CPV and bij-CPV are valid
  - Bij-CPV codes are sub-categories of hoofd (hierarchy check)
- eForms XML includes:
  - BT-262 (main): "72000000-8"
  - BT-263 (supplementary): array of "72200000-1" and "72300000-2"

### REQ-TPA-009.S4: NUTS-Code Lookup from Place of Performance
**GIVEN** aanbesteding with `plaats_van_uitvoering = "Gemeente Utrecht"`

**WHEN** system resolves NUTS code

**THEN**
- System searches `nuts_code` table for "Utrecht"
- Finds: code "NL310", label "Utrecht", level 3
- Associated LAU codes: ["0344"]
- System stores `plaats_van_uitvoering_nuts = "NL310"`
- eForms BT-727 is populated with "NL310"
- UI displays "NL310 (Utrecht, LAU-0344)" for user confirmation

### REQ-TPA-009.S5: Invalid NUTS Rejection
**GIVEN** user manually enters NUTS code "NL999" (invalid)

**WHEN** validation runs

**THEN**
- System searches `nuts_code` table; no match found for "NL999"
- Validation error: "NUTS-code NL999 is ongeldig. Kies uit de aktieve NUTS-NL codes."
- Typeahead suggestions are shown for similar codes (e.g., NL990, NL991)

---

## REQ-TPA-010: Audit Trail and Legal Retention

**Title:** Maintain immutable audit trail and enforce 7-year retention per AW2012 art 4.13.

**Description:**
The system MUST keep an immutable audit trail of every `publicatie`-mutatie, store eForms XML payloads and TenderNed responses for at least 7 years (AW2012 art 4.13 bewaartermijn), expose the audit log to auditors via a read-only API, and ensure no purge job can delete `bewaarplicht`-records before the wettelijke termijn expires.

**Scenarios:**

### REQ-TPA-010.S1: Automatic Audit Logging on Status Transitions
**GIVEN** a `tenderned_publicatie` transitioning from `concept` → `ingediend`

**WHEN** status change occurs

**THEN**
- OpenRegister's AuditTrailService automatically creates log entry:
  ```json
  {
    "timestamp": "2026-05-23T14:30:00Z",
    "actor": "user-12345",
    "objectId": "550e8400-e29b-41d4-a716-446655440000",
    "schema": "tenderned_publicatie",
    "action": "UPDATE",
    "field": "status",
    "oldValue": "concept",
    "newValue": "ingediend",
    "xmlPayloadHash": "sha256:abc123...",
    "tenderNedResponseHash": "sha256:def456...",
    "immutable": true,
    "retention_until": "2033-05-23" (7 years from now)
  }
  ```
- Entry is append-only (no updates or deletes allowed)
- Hashes of eForms XML and TN responses are stored for integrity verification

### REQ-TPA-010.S2: Retention Enforcement
**GIVEN** an audit log entry created on 2019-05-23 (more than 7 years old)

**WHEN** a background purge job attempts to delete it on 2026-05-25

**THEN**
- OpenRegister's DestructionService checks `retention_until` date
- Date is 2026-05-23, which is today → still within retention period
- Deletion is blocked
- Log entry remains in system
- Once 2026-05-24 arrives (7 years + 1 day), destruction can proceed

### REQ-TPA-010.S3: Audit Export for Auditor
**GIVEN** an auditor request: "Alle publicaties voor procedure X tussen 2025-01-01 en 2026-05-31"

**WHEN** auditor calls GET /api/audit-export?procedure=X&from=2025-01-01&to=2026-05-31

**THEN**
- System queries AuditTrailService for matching records
- For each matching `tenderned_publicatie`:
  - Exports `tenderned_publicatie` JSON
  - Exports `tenderned_bijlage` records with checksums
  - Exports all AuditTrailService log entries for state transitions
  - Exports TenderNed API responses (stored as blobs)
- Response is a signed JSON+XML bundle:
  ```json
  {
    "exportDate": "2026-05-23T14:30:00Z",
    "exportedBy": "user-54321",
    "procedure": "X",
    "dateRange": { "from": "2025-01-01", "to": "2026-05-31" },
    "publicaties": [ { ... }, ... ],
    "bijlagen": [ { ... checksums... }, ... ],
    "auditLogEntries": [ { ... immutable logs... }, ... ],
    "tenderNedResponses": [ { ... }, ... ],
    "signature": "sha256:xyz789..." (HMAC-signed for integrity)
  }
  ```
- Auditor can verify integrity by checking signature
- Export is read-only; no modifications are possible

### REQ-TPA-010.S4: Bewaartermijn Workflow
**GIVEN** a `tenderned_publicatie` object that has reached retention expiry (7 years + 1 day)

**WHEN** admin initiates deletion workflow

**THEN**
- System checks `retention_until` date
- Date has passed, so deletion is permitted
- Admin is required to provide explicit `bewaartermijn_verlopen` workflow with second-factor approval (2FA)
- Steps:
  1. Admin clicks "Verwijder publicatie"
  2. System confirms: "Dit record valt buiten de bewaartermijn. Weet u zeker?"
  3. Admin completes 2FA (SMS, authenticator app, etc.)
  4. System sets a deletion flag + timestamp
  5. After 24-hour hold period, deletion is executed
  6. Audit log entry is created: "Record deleted after bewaartermijn expiry, approved by {admin}, 2FA verified"
  7. Record is soft-deleted (marked as deleted, but audit trail remains)

---

## Cross-Functional Requirements

### REQ-TPA-XF-001: Mandaat-Based Approval Gates
**Title:** Integrate approval workflow with purchaseq mandaat system.

For publicaties with `geraamde_waarde` or `procedure` above configured thresholds, system MUST require approval by a mandaathouder before allowing submission to TenderNed.

**GIVEN** a publicatie with:
- `geraamde_waarde = EUR 10.000.000`
- Configured mandaat threshold for Finance Director: EUR 5.000.000

**WHEN** user attempts to submit

**THEN**
- System checks mandaat rules
- Requires approval from Finance Director (or delegate)
- Finance Director receives notification: "Publicatie #550e8400 (EUR 10M) awaits your approval"
- Only after approval (goedgekeurdDoor + goedkeuringsDatum recorded) can submission proceed

### REQ-TPA-XF-002: Integration with openconnector CallLog
**Title:** All TenderNed API calls are logged in openconnector CallLog for audit.

Every HTTP request/response to TenderNed is:
- Logged in CallLog with timestamp, method, URL, request body (sanitized), response code, response body
- Associated with the publicatie correlatieId
- Accessible to CISO for security audits
- Retained per OpenConnector's default policy

### REQ-TPA-XF-003: Notification Service Integration
**Title:** All user-facing notifications route via NotificationService.

- New vragen: immediate notification
- Approval requests: immediate notification
- Publication status changes (gepubliceerd, geweigerd, ingetrokken): immediate notification
- Reconciliation drift alerts: immediate notification (critical: also email to ops)

### REQ-TPA-XF-004: FileService Integration
**Title:** All documents are stored via FileService, not custom upload handlers.

- Bijlagen are stored in OpenRegister's file attachment system
- SHA-256 checksums are computed by FileService (with custom TN verification wrapper)
- Files are accessible via FileService REST API
- Download/share links are managed by FileService

---

## Edge Cases and Error Handling

### Edge Case 1: Publicatie with Expired Drempelbedrag Period
**Scenario:** Active drempelbedrag period changes mid-publication cycle (unlikely but possible in Jan/Feb when periods roll over).

**Handling:** System uses the drempelbedrag period active at **publication creation time**, not current time. If period changes before submission, validation runs against the original period; user is notified that drempelbedrag has changed and invited to regenerate if they wish to use new thresholds.

### Edge Case 2: Concurrent Upload and Status Change
**Scenario:** User is uploading documents while another user transitions publicatie status to `ingediend`.

**Handling:** Status transition is blocked until all uploads complete. System checks for in-flight uploads and prevents status change (UI shows "Uploads in progress; wait before submitting").

### Edge Case 3: TenderNed Accepts Submission but Never Returns Final Status
**Scenario:** System submitted publicatie, received 202 Accepted, but polling never receives PUBLISHED or REJECTED.

**Handling:** After 10 minutes of polling without final status, system stops and leaves publicatie in `ingediend` state. User can manually click "Controleer status" to query TenderNed; if still no response after 24 hours, ops alert is raised and manual intervention is required.

### Edge Case 4: Bitwise Identical Bijlage Upload Twice
**Scenario:** User uploads same PDF twice with same filename.

**Handling:** System checks SHA-256 checksum; if identical to an existing bijlage, warns user that document already exists and asks to confirm duplicate.

---

## Out of Scope (Tested Elsewhere)

- Drempelbedrag calculations and procedure-choice business rules (belong to `aanbesteding-werkproces`; this adapter assumes they are pre-validated)
- eForms SDK version management and updates (assumed to be a settings configuration)
- OpenRegister schema CRUD and store operations (tested in OpenRegister test suite)
- TenderNed API contract changes (assumed TN maintains backwards-compatibility; changes would be documented in TN release notes)
- Multi-language rendering and i18n of UI labels (belongs to nldesign-i18n capability)

---

**Next Steps:** Task breakdown for development and testing phases.
