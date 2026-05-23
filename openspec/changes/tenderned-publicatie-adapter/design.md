# TenderNed Publicatie Adapter — Design

**Status:** Design  
**Version:** 1.0  
**Last Updated:** 2026-05-23

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          purchaseq UI                               │
│  (Beheer > Connectors > TenderNed, aanbesteding detail tabs)       │
└──────────────┬──────────────────────────────────────────────────────┘
               │
               ├─ OpenRegister API (CRUD: publicatie, bijlage, vraag_antwoord, inschrijving)
               │
               ├─ TenderNedAdapter Controller
               │  ├─ PublicatieGenerator (concept → eForms XML)
               │  ├─ ValidatorService (XSD, Schematron, TN-extension)
               │  ├─ SubmissionService (POST /notices, poll /status)
               │  ├─ DocumentUploadService (resumable, chunked, SHA-256)
               │  └─ ReconciliationService (daily drift-check)
               │
               ├─ n8n Workflows (via n8n MCP)
               │  ├─ Poll Vragen (every 15 min)
               │  └─ Daily Reconciliation (nightly)
               │
               └─ openconnector TenderNed Source
                  ├─ OAuth2 client-credentials auth
                  ├─ CallLog audit
                  └─ Retry envelope (circuit-breaker, exponential backoff)
```

## Schema Definitions (OpenRegister)

All schemas are defined in `lib/Settings/purchaseq_register.json` under the `purchaseq` register.

### Schema: tenderned_publicatie

**Purpose:** Records every publication transaction to TenderNed.

**Entity Definition:**
```json
{
  "name": "tenderned_publicatie",
  "title": "TenderNed Publicatie",
  "description": "A publication to TenderNed representing a single notice (vooraankondiging, aankondiging_opdracht, etc.)",
  "type": "object",
  "required": ["aanbestedingId", "publicatieType", "status"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "title": "Publicatie ID"
    },
    "aanbestedingId": {
      "type": "string",
      "description": "Reference to the base aanbesteding object (register: purchaseq, schema: aanbesteding, objectId)"
    },
    "publicatieType": {
      "type": "string",
      "enum": [
        "vooraankondiging",
        "aankondiging_opdracht",
        "aankondiging_gunning",
        "aankondiging_wijziging",
        "rectificatie",
        "vrijwillige_transparantievooraf",
        "concessie",
        "sociale_specifieke_diensten"
      ],
      "title": "Publication Type"
    },
    "eformsNoticeSubtype": {
      "type": "string",
      "pattern": "^[0-9]{2}$",
      "title": "eForms Notice Subtype",
      "description": "40+ eForms subtype codes per EU Regulation 2019/1780, e.g., '16' for open above-threshold works"
    },
    "status": {
      "type": "string",
      "enum": ["concept", "gevalideerd", "ingediend", "gepubliceerd", "ingetrokken", "geweigerd"],
      "title": "Publication Status"
    },
    "tendernedPublicatieId": {
      "type": "string",
      "description": "TenderNed-assigned publication ID, set on successful publish"
    },
    "tedPublicatieId": {
      "type": "string",
      "description": "TED publication number in format YYYY/S NNN-NNNNNN (only for boven-drempel notices)"
    },
    "publicatieDatum": {
      "type": "string",
      "format": "date-time",
      "description": "Publication timestamp from TenderNed response (not local clock)"
    },
    "sluitingsDatum": {
      "type": "string",
      "format": "date-time",
      "description": "Deadline for bid submission"
    },
    "eformsXml": {
      "type": "string",
      "description": "Canonical eForms 2.0 XML payload submitted to TenderNed"
    },
    "validatieResultaten": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "ruleId": { "type": "string" },
          "severity": { "type": "string", "enum": ["error", "warning"] },
          "message": { "type": "string" },
          "xpath": { "type": "string" }
        }
      },
      "title": "Validation Results"
    },
    "bijlagen": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Array of refs to tenderned_bijlage objects"
    },
    "correlatieId": {
      "type": "string",
      "format": "uuid",
      "description": "UUID for idempotency"
    },
    "gepubliceerdDoor": {
      "type": "string",
      "description": "User ID who initiated publication"
    },
    "goedgekeurdDoor": {
      "type": "string",
      "description": "Mandaat holder ID who approved publication (if above threshold)"
    },
    "goedkeuringsDatum": {
      "type": "string",
      "format": "date-time"
    },
    "tendernedDossierUrl": {
      "type": "string",
      "format": "uri",
      "description": "Public URL to the dossier on TenderNed"
    }
  }
}
```

### Schema: tenderned_bijlage

**Purpose:** Tracks document uploads to TenderNed.

**Entity Definition:**
```json
{
  "name": "tenderned_bijlage",
  "title": "TenderNed Document Attachment",
  "type": "object",
  "required": ["publicatieId", "bestandsnaam"],
  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "publicatieId": {
      "type": "string",
      "description": "Reference to tenderned_publicatie"
    },
    "documentRef": {
      "type": "string",
      "description": "OpenRegister file attachment reference (register: openregister-files)"
    },
    "bestandsnaam": { "type": "string" },
    "mimeType": { "type": "string" },
    "groottebytes": { "type": "integer" },
    "documentType": {
      "type": "string",
      "enum": [
        "selectieleidraad",
        "gunningsleidraad",
        "bestek",
        "beschrijvend_document",
        "programma_van_eisen",
        "conceptovereenkomst",
        "nota_van_inlichtingen",
        "proces_verbaal_opening",
        "gunningsbeslissing"
      ]
    },
    "taal": {
      "type": "string",
      "enum": ["NL", "EN", "FR", "DE"],
      "title": "Language (per AW2012 art 2.78)"
    },
    "tendernedBijlageId": {
      "type": "string",
      "description": "Set on successful upload"
    },
    "checksumSha256": { "type": "string" },
    "vertrouwelijk": {
      "type": "boolean",
      "description": "If true, document is only visible after bid opening"
    },
    "uploadStatus": {
      "type": "string",
      "enum": ["pending", "in_progress", "completed", "failed", "failed_checksum_mismatch"],
      "title": "Upload Status"
    }
  }
}
```

### Schema: tenderned_vraag_antwoord

**Purpose:** Q&A round during inlichtingenperiode.

**Entity Definition:**
```json
{
  "name": "tenderned_vraag_antwoord",
  "title": "Question and Answer",
  "type": "object",
  "required": ["publicatieId"],
  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "publicatieId": {
      "type": "string",
      "description": "Reference to tenderned_publicatie"
    },
    "vraagnummer": {
      "type": "integer",
      "description": "Auto-incremented per aanbesteding"
    },
    "vraag": {
      "type": "string",
      "description": "Question text (anonymised)"
    },
    "vraagstelDatum": { "type": "string", "format": "date-time" },
    "antwoord": {
      "type": "string",
      "description": "Answer text"
    },
    "antwoordDatum": { "type": "string", "format": "date-time" },
    "bijlagen": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Refs to supporting documents"
    },
    "betreftDocument": {
      "type": "string",
      "description": "Ref to tenderned_bijlage (e.g., bestek) that question concerns"
    },
    "betreftHoofdstuk": {
      "type": "string",
      "description": "Chapter/section reference if applicable"
    },
    "status": {
      "type": "string",
      "enum": ["open", "beantwoord", "verwerkt_in_nota"],
      "title": "Q&A Status"
    },
    "laat_ontvangen": {
      "type": "boolean",
      "description": "True if question arrived within 6 days of sluitingsDatum (may require deadline extension per AW2012 art 2.65 lid 3)"
    },
    "notaVanInlichtingenRef": {
      "type": "string",
      "description": "Ref to the nota van inlichtingen document that contains this answer"
    }
  }
}
```

### Schema: tenderned_inschrijving

**Purpose:** Bid receipt and opening data from TenderNed secure depot.

**Entity Definition:**
```json
{
  "name": "tenderned_inschrijving",
  "title": "Bid Submission",
  "type": "object",
  "required": ["publicatieId"],
  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "publicatieId": {
      "type": "string",
      "description": "Reference to tenderned_publicatie"
    },
    "tendernedInschrijvingId": {
      "type": "string",
      "description": "TenderNed-assigned bid ID"
    },
    "ingediendDatum": {
      "type": "string",
      "format": "date-time",
      "description": "Must be before sluitingsDatum"
    },
    "openingsDatum": { "type": "string", "format": "date-time" },
    "documentRefs": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Encrypted until opening"
    },
    "inschrijverNaam": {
      "type": "string",
      "description": "Visible only post-opening"
    },
    "inschrijverKvK": {
      "type": "string",
      "description": "KvK number, visible post-opening"
    },
    "inschrijvingsBedrag": {
      "type": "number",
      "format": "decimal",
      "description": "Visible post-opening"
    },
    "combinatie": {
      "type": "boolean",
      "description": "True if bid is a consortium"
    },
    "deelnemers": {
      "type": "array",
      "items": { "type": "string" },
      "description": "KvK refs for consortium members"
    },
    "onderaanneming": {
      "type": "boolean",
      "description": "True if subcontracting is used"
    },
    "onderaannemers": {
      "type": "array",
      "items": { "type": "string" },
      "description": "KvK refs for subcontractors"
    },
    "geldigVerklaard": {
      "type": "boolean",
      "description": "Validated by awarding authority"
    },
    "motivering": {
      "type": "string",
      "description": "Reason for validity decision"
    }
  }
}
```

### Schema: cpv_code (Reference Data)

**Purpose:** Common Procurement Vocabulary codes (imported quarterly).

```json
{
  "name": "cpv_code",
  "title": "CPV Code",
  "type": "object",
  "required": ["code"],
  "properties": {
    "code": {
      "type": "string",
      "pattern": "^[0-9]{8}-[0-9]$",
      "title": "CPV Code",
      "description": "9-digit code with verification digit per EC Regulation 2195/2002"
    },
    "label_nl": { "type": "string" },
    "label_en": { "type": "string" },
    "parent_code": { "type": "string" },
    "level": { "type": "integer", "enum": [1, 2, 3] },
    "active": { "type": "boolean" }
  }
}
```

### Schema: nuts_code (Reference Data)

**Purpose:** NUTS-NL codes (imported quarterly).

```json
{
  "name": "nuts_code",
  "title": "NUTS Code",
  "type": "object",
  "required": ["code"],
  "properties": {
    "code": {
      "type": "string",
      "pattern": "^NL[0-9]{3}$",
      "title": "NUTS Code"
    },
    "label": { "type": "string" },
    "level": { "type": "integer", "enum": [1, 2, 3] },
    "lau_codes": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Associated LAU (Local Administrative Unit) codes"
    }
  }
}
```

### Schema: drempelbedrag_periode (Reference Data)

**Purpose:** EU and Dutch threshold values (refreshed bi-annually).

```json
{
  "name": "drempelbedrag_periode",
  "title": "Threshold Period",
  "type": "object",
  "required": ["startDatum", "einddatum"],
  "properties": {
    "startDatum": { "type": "string", "format": "date" },
    "einddatum": { "type": "string", "format": "date" },
    "klassieke_werken_eur": { "type": "number" },
    "klassieke_leveringen_centraal_eur": { "type": "number" },
    "klassieke_leveringen_decentraal_eur": { "type": "number" },
    "klassieke_diensten_centraal_eur": { "type": "number" },
    "klassieke_diensten_decentraal_eur": { "type": "number" },
    "speciale_sector_werken_eur": { "type": "number" },
    "speciale_sector_leveringen_eur": { "type": "number" },
    "speciale_sector_diensten_eur": { "type": "number" },
    "sociale_specifieke_diensten_eur": { "type": "number" },
    "concessie_eur": { "type": "number" },
    "nationale_onderdrempel_indication_eur": { "type": "number" },
    "source": {
      "type": "string",
      "enum": ["EU_regulation_2023_2497", "EU_regulation_2023_2495", "EU_regulation_2023_2496"]
    }
  }
}
```

## eForms Mapping Tables

### Mapping: Aanbesteding → eForms Notice (Subtype Selection)

```
Aanbesteding Fields → eForms Subtype & BT-Fields

concessie=true
  → publicatieType: "concessie"
  → eformsNoticeSubtype: "25" (Concession notice)
  → drempel: 5.538.000 EUR (2024-2025)

procedure="open" OR "niet_openbaar" AND geraamde_waarde >= drempel
  AND cpv_major IN (45.000.000-7 | 48.000.000-2) -- works
  → eformsNoticeSubtype: "16" (Open above-threshold works)

procedure="open" OR "niet_openbaar" AND geraamde_waarde >= drempel
  AND cpv_major NOT IN works
  → eformsNoticeSubtype: "17" (Open above-threshold supplies/services)

geraamde_waarde < drempel AND NOT concessie
  → publicatieType: "aankondiging_opdracht" (nationale)
  → eformsNoticeSubtype: "T01-T30" (nationale range, TN-specific)

sociale_specifieke_diensten=true AND geraamde_waarde < 750.000 EUR
  → publicatieType: "sociale_specifieke_diensten"
  → eformsNoticeSubtype: "27" (Social and other specific services notice)

speciale_sector=true AND geraamde_waarde >= speciale_sector_drempel
  → eformsNoticeSubtype: "18" (Open above-threshold speciale sector)
```

### eForms BT-Field Mappings (Excerpt)

```
BT-21   → aanbesteding.titel (EN, NL mandatory)
BT-262  → aanbesteding.cpv_main_code
BT-263  → aanbesteding.cpv_supplementary_codes[]
BT-27   → aanbesteding.geraamde_waarde
BT-105  → aanbesteding.procedure (enum mapping: open→1, niet_openbaar→2, etc.)
BT-36   → aanbesteding.looptijd_in_maanden (as duration)
BT-301  → aanbesteding.plaats_van_uitvoering (NUTS code)
BT-539  → aanbesteding.gunningscriteria[] (price weight, quality weight, etc.)
BT-137  → aanbesteding.percelen[] (lot identifiers for multi-lot tenders)
BT-02   → tenderned_publicatie.publicatieDatum (set by system)
BT-701  → aanbesteding.contact_persoon (email, phone)

For MVI/SROI (injected from mvi-sroi-aanbesteding capability):
BT-805  → MVI flag (green procurement)
BT-755  → Accessibility flag
BT-738  → Social criteria

For speciale-sector:
BT-729  → Speciale sector type (water/energie/vervoer/post)
```

## n8n Workflow Definitions

### Workflow 1: Poll Vragen (Every 15 Minutes)

```yaml
Trigger: Cron (0 */15 * * * * *)
Steps:
  1. Get all tenderned_publicatie with status="gepubliceerd"
     and publicatieDatum < 7 days ago (inlichtingenperiode active)
  
  2. For each publicatie:
     a. Call TenderNed GET /notices/{publicatieId}/questions
     b. Filter to questions with vraagstelDatum > laatstePollMoment
     c. For each new question:
        - Create tenderned_vraag_antwoord record with status="open"
        - Check if vraagstelDatum is within 6 days of sluitingsDatum
          → set laat_ontvangen=true if so
        - Send notification to inkoper: "Nieuwe vraag #{vraagnummer}"
  
  3. Log poll event with count of new vragen
```

### Workflow 2: Daily Reconciliation (Nightly at 02:00 UTC)

```yaml
Trigger: Cron (0 2 * * * *)
Steps:
  1. Get all tenderned_publicatie with status IN
     ("gepubliceerd", "ingediend", "ingetrokken")
  
  2. For each publicatie:
     a. Call TenderNed GET /notices/{publicatieId}/status
     b. Compare local status vs TN status:
        - If local="gepubliceerd" AND TN="withdrawn":
          → Update local to "ingetrokken"
          → Log: external_withdrawal
          → Notify inkoper: "Publicatie #{publicatieId} is ingetrokken op TenderNed"
        
        - If local="ingetrokken" AND TN="published":
          → Raise CRITICAL alert: "Legal position divergence: local ingetrokken but TN published"
          → Do NOT auto-correct (requires manual investigation)
          → Escalate to ops@purchaseq
        
        - If TenderNed unreachable:
          → Increment retry counter
          → After 24h of unreachability, stop polling and raise ops alert
  
  3. Log reconciliation summary with drift count
```

## Seed Data (OpenRegister components.objects)

### tenderned_publicatie Examples

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "tenderned_publicatie",
    "slug": "aankondiging-werken-gemeente-amsterdam-2026-001"
  },
  "aanbestedingId": "aanbesteding-werken-gemeente-amsterdam-2026-001",
  "publicatieType": "aankondiging_opdracht",
  "eformsNoticeSubtype": "16",
  "status": "gepubliceerd",
  "tendernedPublicatieId": "TN-2026-1234567",
  "tedPublicatieId": "2026/S 123-456789",
  "publicatieDatum": "2026-05-23T14:30:00Z",
  "sluitingsDatum": "2026-07-15T23:59:59Z",
  "eformsXml": "[XML content...]",
  "validatieResultaten": [],
  "bijlagen": ["bijlage-bestek-werken-2026-001"],
  "correlatieId": "550e8400-e29b-41d4-a716-446655440000",
  "gepubliceerdDoor": "user-12345",
  "goedgekeurdDoor": "user-54321",
  "goedkeuringsDatum": "2026-05-23T10:00:00Z",
  "tendernedDossierUrl": "https://www.tenderned.nl/tdc/servlet/DCD?cIid=1234567"
}
```

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "tenderned_publicatie",
    "slug": "aankondiging-leveringen-provincie-zuid-holland-2026-001"
  },
  "aanbestedingId": "aanbesteding-leveringen-provincie-zuid-holland-2026-001",
  "publicatieType": "aankondiging_opdracht",
  "eformsNoticeSubtype": "17",
  "status": "concept",
  "validatieResultaten": [
    {
      "ruleId": "BR-OPT-13",
      "severity": "error",
      "message": "Procurement project must have a title in the official language",
      "xpath": "/cbc:Title/cbc:NameType[@languageID='NL']"
    }
  ],
  "bijlagen": [],
  "correlatieId": "660e8400-e29b-41d4-a716-446655440111"
}
```

### tenderned_bijlage Examples

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "tenderned_bijlage",
    "slug": "bijlage-bestek-werken-2026-001"
  },
  "publicatieId": "aankondiging-werken-gemeente-amsterdam-2026-001",
  "documentRef": "file-attachment-2026-001",
  "bestandsnaam": "bestek-werken-amsterdam.pdf",
  "mimeType": "application/pdf",
  "groottebytes": 2457600,
  "documentType": "bestek",
  "taal": "NL",
  "tendernedBijlageId": "TNB-2026-987654",
  "checksumSha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "vertrouwelijk": false,
  "uploadStatus": "completed"
}
```

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "tenderned_bijlage",
    "slug": "bijlage-leidraad-amsterdam-2026-001"
  },
  "publicatieId": "aankondiging-werken-gemeente-amsterdam-2026-001",
  "documentRef": "file-attachment-2026-002",
  "bestandsnaam": "selectieleidraad-amsterdam.pdf",
  "mimeType": "application/pdf",
  "groottebytes": 1048576,
  "documentType": "selectieleidraad",
  "taal": "NL",
  "tendernedBijlageId": "TNB-2026-987655",
  "checksumSha256": "b3c0d44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b856",
  "vertrouwelijk": false,
  "uploadStatus": "completed"
}
```

### tenderned_vraag_antwoord Examples

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "tenderned_vraag_antwoord",
    "slug": "qa-werken-amsterdam-2026-001-001"
  },
  "publicatieId": "aankondiging-werken-gemeente-amsterdam-2026-001",
  "vraagnummer": 1,
  "vraag": "Kan het bestekonderdeel 3.2 ook in kunststof worden uitgevoerd?",
  "vraagstelDatum": "2026-05-25T09:15:00Z",
  "antwoord": "Nee, onderdeel 3.2 moet in duurzaam hout worden uitgevoerd per AW2012 eisen.",
  "antwoordDatum": "2026-05-26T14:30:00Z",
  "betreftDocument": "bijlage-bestek-werken-2026-001",
  "betreftHoofdstuk": "3.2 Materialen",
  "status": "verwerkt_in_nota",
  "laat_ontvangen": false,
  "notaVanInlichtingenRef": "bijlage-nota-inlichtingen-amsterdam-2026-001"
}
```

### cpv_code Examples

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "cpv_code",
    "slug": "cpv-45000000-7"
  },
  "code": "45000000-7",
  "label_nl": "Bouwwerken",
  "label_en": "Construction works",
  "level": 1,
  "active": true
}
```

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "cpv_code",
    "slug": "cpv-45100000-8"
  },
  "code": "45100000-8",
  "label_nl": "Voorbereidingswerkzaamheden en grondwerk",
  "label_en": "Site preparation and excavation",
  "parent_code": "45000000-7",
  "level": 2,
  "active": true
}
```

### nuts_code Examples

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "nuts_code",
    "slug": "nuts-nl310"
  },
  "code": "NL310",
  "label": "Utrecht",
  "level": 3,
  "lau_codes": ["0344"]
}
```

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "nuts_code",
    "slug": "nuts-nl325"
  },
  "code": "NL325",
  "label": "Amsterdam",
  "level": 3,
  "lau_codes": ["0363"]
}
```

### drempelbedrag_periode Example

```json
{
  "@self": {
    "register": "purchaseq",
    "schema": "drempelbedrag_periode",
    "slug": "drempel-2024-2025"
  },
  "startDatum": "2024-12-01",
  "einddatum": "2025-11-30",
  "klassieke_werken_eur": 5538000,
  "klassieke_leveringen_centraal_eur": 143000,
  "klassieke_leveringen_decentraal_eur": 221000,
  "klassieke_diensten_centraal_eur": 143000,
  "klassieke_diensten_decentraal_eur": 221000,
  "speciale_sector_werken_eur": 5538000,
  "speciale_sector_leveringen_eur": 443000,
  "speciale_sector_diensten_eur": 443000,
  "sociale_specifieke_diensten_eur": 750000,
  "concessie_eur": 5538000,
  "nationale_onderdrempel_indication_eur": 50000,
  "source": "EU_regulation_2023_2497"
}
```

## TenderNed API Contracts

### POST /notices (Submit Publication)

**Request:**
```
POST https://api.tenderned.nl/notices
Authorization: Bearer {oauth2_token}
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<Notice>
  <!-- eForms XML per SDK version -->
  ...
</Notice>
```

**Response (202 Accepted):**
```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "statusUrl": "/notices/550e8400-e29b-41d4-a716-446655440000/status"
}
```

**Error Response (400, 422, 503):**
```json
{
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

### GET /notices/{correlationId}/status (Poll Publication Status)

**Request:**
```
GET https://api.tenderned.nl/notices/550e8400-e29b-41d4-a716-446655440000/status
Authorization: Bearer {oauth2_token}
```

**Response (200 OK):**
```json
{
  "status": "PUBLISHED",
  "publicationId": "TN-2026-1234567",
  "tedPublicationId": "2026/S 123-456789",
  "publicationDate": "2026-05-23T14:30:00Z",
  "dossierUrl": "https://www.tenderned.nl/tdc/servlet/DCD?cIid=1234567"
}
```

### POST /notices/{publicationId}/documents (Upload Document)

**Request:**
```
POST https://api.tenderned.nl/notices/TN-2026-1234567/documents
Authorization: Bearer {oauth2_token}
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="bestek.pdf"
Content-Type: application/pdf
[binary file content, up to 200 MB]
--boundary
Content-Disposition: form-data; name="documentType"
bestek
--boundary
Content-Disposition: form-data; name="confidentialityLevel"
PUBLIC
--boundary
Content-Disposition: form-data; name="sha256"
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
--boundary--
```

**Response (202 Accepted, resumable session):**
```json
{
  "documentId": "TNB-2026-987654",
  "uploadSessionToken": "sess-2026-001",
  "uploadStatus": "IN_PROGRESS",
  "bytesReceived": 524288,
  "checksumVerified": false
}
```

### GET /notices/{publicationId}/questions (Poll Questions)

**Request:**
```
GET https://api.tenderned.nl/notices/TN-2026-1234567/questions?since=2026-05-25T09:00:00Z
Authorization: Bearer {oauth2_token}
```

**Response (200 OK):**
```json
{
  "questions": [
    {
      "questionId": "Q-2026-001",
      "question": "Kan onderdeel 3.2 ook in kunststof?",
      "askedAt": "2026-05-25T09:15:00Z",
      "askedByEmail": "bidder@example.nl",
      "relatedDocument": "bestek.pdf",
      "status": "OPEN"
    }
  ]
}
```

## Exception Handling and Retry Logic

All TenderNed API calls use openconnector's retry envelope:

```
HTTP 5xx (500, 502, 503, 504)
  → Exponential backoff: retry at 1s, 2s, 4s, 8s, 16s, 32s (max 6 retries)
  → Resume from last checkpoint (for chunked uploads)

HTTP 4xx (400, 422)
  → Do NOT retry (validation error; requires user action)
  → Capture error detail in publicatie.validatieResultaten

HTTP 429 (Too Many Requests)
  → Exponential backoff with jitter (respect Retry-After header)

Connection timeout / network error
  → Retry with exponential backoff (max 5 retries)

Timeout after max retries
  → Transition publicatie status to "concept"
  → Log error with TenderNed correlation ID
  → Notify user to retry or contact support
```

## Reuse Analysis

This capability leverages existing OpenRegister patterns:

- **ObjectService** (CRUD): All publicatie, bijlage, vraag_antwoord, inschrijving records.
- **AuditTrailService** (automatic): Every status transition is logged; 7-year retention enforced via OpenRegister's destruction schedules.
- **FileService** (multipart uploads): Document upload infrastructure (with custom SHA-256 verification wrapper).
- **NotificationService**: Alerts for new vragen, approvals, rejections.
- **AuthorizationService**: Mandaat-based approval gates.
- **OpenConnector**: TenderNed source definition, OAuth2, retry envelope, CallLog.
- **ScheduledWorkflowController**: n8n integration for polling and reconciliation jobs.

No duplicate logic is needed; all adapters reuse openconnector's standard patterns.

---

**Next Steps:** Specification phase to detail REQ-XXX validation logic, error scenarios, and edge cases.
