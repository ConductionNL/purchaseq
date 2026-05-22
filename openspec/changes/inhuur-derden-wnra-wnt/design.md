---
app: purchaseq
spec: inhuur-derden-wnra-wnt
title: Inhuur Derden met Wet DBA en WNT Toetsing — Design
status: draft
version: 1.0
date: 2026-05-22
---

# Design: Inhuur Derden met Wet DBA en WNT Toetsing

## Data Model Overview

All domain objects stored in OpenRegister following ADR-001 (data layer) and ADR-011 (schema standards). No custom Entity/Mapper classes for domain data.

### Entities

#### 1. InhuurOpdracht (Primary Entity)

Core contract record linking department need, vendor selection, budget, compliance status, and financial outcome.

```json
{
  "name": "InhuurOpdracht",
  "slug": "inhuur-opdracht",
  "extends": "schema:Event",
  "properties": {
    "opdrachtId": {
      "type": "string",
      "format": "uuid",
      "required": true,
      "description": "System-generated unique contract ID"
    },
    "opdrachtNummer": {
      "type": "string",
      "required": true,
      "description": "Human-readable contract number (e.g., HDO-2026-001-ZZP)"
    },
    "opdrachtgever": {
      "type": "object",
      "properties": {
        "register": { "type": "string", "const": "employees" },
        "schema": { "type": "string", "const": "Person" },
        "slug": { "type": "string" }
      },
      "required": true,
      "description": "Department/team initiating the hire (reference to HRMQ employee)"
    },
    "opdrachtnemer": {
      "type": "object",
      "properties": {
        "register": { "type": "string", "const": "inhuur-opdrachtnemers" },
        "schema": { "type": "string", "const": "Opdrachtnemer" },
        "slug": { "type": "string" }
      },
      "required": true,
      "description": "Vendor/contractor (internal reference)"
    },
    "opdrachtOmschrijving": {
      "type": "string",
      "required": true,
      "description": "Job title and scope (e.g., 'Interim IT-architect, 6 months, 32 hrs/week')"
    },
    "inhuurType": {
      "type": "string",
      "enum": ["zzp", "uitzend", "detachering", "consultancy", "interim"],
      "required": true,
      "description": "Classification of hire for compliance (impacts DBA/WNT routing)"
    },
    "startDatum": {
      "type": "string",
      "format": "date",
      "required": true,
      "description": "Hire start date"
    },
    "geplande EindDatum": {
      "type": "string",
      "format": "date",
      "required": true,
      "description": "Originally planned end date"
    },
    "actueleEindDatum": {
      "type": "string",
      "format": "date",
      "description": "Actual termination date (set when contract ends)"
    },
    "verlengingsOptie": {
      "type": "boolean",
      "required": true,
      "description": "Whether contract has renewal option"
    },
    "uurtarief": {
      "type": "number",
      "minimum": 0,
      "required": true,
      "description": "Hourly rate excl. VAT (EUR)"
    },
    "geschatTotaalbudget": {
      "type": "number",
      "minimum": 0,
      "required": true,
      "description": "Initial estimated total spend (EUR)"
    },
    "daadwerkelijkeUitgaven": {
      "type": "number",
      "minimum": 0,
      "description": "Cumulative invoiced amount from Shillinq (EUR)"
    },
    "kostenplaats": {
      "type": "string",
      "required": true,
      "description": "Cost center code for accounting"
    },
    "projectCode": {
      "type": "string",
      "description": "Project code if applicable"
    },
    "gezagsrelatieScore": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "description": "DBA assessment score: authority/control dimension"
    },
    "vervangbaarheidScore": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "description": "DBA assessment score: replaceability dimension"
    },
    "dbaConlusie": {
      "type": "string",
      "enum": ["groen", "oranje", "rood", "niet-beoordeeld"],
      "description": "DBA risk classification from latest assessment"
    },
    "wntRelevant": {
      "type": "boolean",
      "required": true,
      "description": "Whether contractor subject to WNT wage cap (org-level flag)"
    },
    "status": {
      "type": "string",
      "enum": ["concept", "in-onderzoek", "goedgekeurd", "actief", "verlengd", "afgerond", "geblokkeerd"],
      "required": true,
      "description": "Contract lifecycle state"
    },
    "red_overrideAutorisatie": {
      "type": "object",
      "properties": {
        "goedgekeurd": { "type": "boolean" },
        "autoriseerderAfdeling": { "type": "string" },
        "autoriseerderController": { "type": "string" },
        "autoriseerderBestuurder": { "type": "string" },
        "datumGoedkeuring": { "type": "string", "format": "date-time" },
        "motivering": { "type": "string" }
      },
      "description": "Three-signature approval record for red DBA overrides"
    }
  }
}
```

#### 2. Opdrachtnemer (Contractor Entity)

Natural person (ZZP) or legal entity (agency/vendor).

```json
{
  "name": "Opdrachtnemer",
  "slug": "opdrachtnemer",
  "extends": "schema:Person|schema:Organization",
  "properties": {
    "opdrachtnemerId": {
      "type": "string",
      "format": "uuid",
      "required": true,
      "description": "Unique vendor/contractor ID"
    },
    "naam": {
      "type": "string",
      "required": true,
      "description": "Full name (person) or org name (company)"
    },
    "kvkNummer": {
      "type": "string",
      "pattern": "^[0-9]{8}$",
      "required": true,
      "description": "Dutch Trade Register number"
    },
    "btwnummer": {
      "type": "string",
      "pattern": "^NL[0-9]{10}B[0-9]{2}$",
      "description": "Dutch VAT number"
    },
    "geboorteDatum": {
      "type": "string",
      "format": "date",
      "description": "Birth date (natural person only, for age verification if applicable)"
    },
    "woonAdres": {
      "type": "object",
      "properties": {
        "straat": { "type": "string" },
        "huisnummer": { "type": "string" },
        "postcode": { "type": "string", "pattern": "^[1-9][0-9]{3}[A-Z]{2}$" },
        "plaats": { "type": "string" }
      },
      "description": "Residential address (natural person)"
    },
    "zakelijkAdres": {
      "type": "object",
      "properties": {
        "straat": { "type": "string" },
        "huisnummer": { "type": "string" },
        "postcode": { "type": "string", "pattern": "^[1-9][0-9]{3}[A-Z]{2}$" },
        "plaats": { "type": "string" }
      },
      "description": "Business address"
    },
    "iban": {
      "type": "string",
      "pattern": "^NL[0-9]{2}[A-Z]{4}[0-9]{10}$",
      "description": "Dutch IBAN for payment"
    },
    "modelovereenkomstType": {
      "type": "string",
      "enum": ["belastingdienst-standaard", "branche-specifiek", "individueel"],
      "description": "Classification of applicable model contract"
    },
    "modelovereenkomstKenmerk": {
      "type": "string",
      "description": "Belastingdienst reference number for model contract"
    },
    "datumLaatsteModelovereenkomstCheck": {
      "type": "string",
      "format": "date",
      "description": "Last validation date of model agreement; triggers warning if >5 yrs"
    },
    "waadiRegistratie": {
      "type": "boolean",
      "description": "Whether vendor has valid WAADI registration (temp staffing/detachment only)"
    },
    "waadiNummer": {
      "type": "string",
      "description": "WAADI registration number from KvK"
    },
    "gRekeningnummer": {
      "type": "string",
      "description": "G-account number (temp staffing agencies only)"
    },
    "aantalOpdrachtgeversLaatste12mnd": {
      "type": "integer",
      "minimum": 0,
      "description": "Number of different clients in past 12 months (ZZP self-employment indicator)"
    }
  }
}
```

#### 3. DbaAssessment (Risk Assessment Entity)

12-question DBA risk scoring per contract, repeatable.

```json
{
  "name": "DbaAssessment",
  "slug": "dba-assessment",
  "properties": {
    "assessmentId": {
      "type": "string",
      "format": "uuid",
      "required": true
    },
    "opdrachtId": {
      "type": "object",
      "properties": {
        "register": { "type": "string", "const": "inhuur-opdrachten" },
        "schema": { "type": "string", "const": "InhuurOpdracht" },
        "slug": { "type": "string" }
      },
      "required": true,
      "description": "Reference to contract"
    },
    "assessmentDatum": {
      "type": "string",
      "format": "date-time",
      "required": true,
      "description": "When assessment was completed"
    },
    "beoordelaar": {
      "type": "string",
      "required": true,
      "description": "User ID / email of assessor"
    },
    "vragen": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "vraagNummer": { "type": "integer", "minimum": 1, "maximum": 12 },
          "vraagTekst": { "type": "string" },
          "antwoord": { "type": "string", "enum": ["ja", "nee", "deels", "onbekend"] },
          "toelichting": { "type": "string" }
        }
      },
      "description": "12 standard DBA questions per Belastingdienst guidance"
    },
    "risicoscore": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "required": true,
      "description": "Calculated risk score: 0-100"
    },
    "conclusie": {
      "type": "string",
      "enum": ["groen", "oranje", "rood"],
      "required": true,
      "description": "Green: <30, Orange: 30-60, Red: >60"
    },
    "advies": {
      "type": "string",
      "description": "Assessor's narrative advice (e.g., 'High risk due to control of daily tasks')"
    },
    "escalatieVereist": {
      "type": "boolean",
      "required": true,
      "description": "Whether escalation to controller/board required"
    }
  }
}
```

#### 4. WntToets (WNT Wage-Cap Tracker)

Per-contract WNT obligation tracking for capped-salary organizations.

```json
{
  "name": "WntToets",
  "slug": "wnt-toets",
  "properties": {
    "toetsId": {
      "type": "string",
      "format": "uuid",
      "required": true
    },
    "opdrachtId": {
      "type": "object",
      "properties": {
        "register": { "type": "string", "const": "inhuur-opdrachten" },
        "schema": { "type": "string", "const": "InhuurOpdracht" },
        "slug": { "type": "string" }
      },
      "required": true
    },
    "kalenderJaar": {
      "type": "integer",
      "minimum": 2020,
      "maximum": 2099,
      "required": true,
      "description": "Fiscal year being tracked"
    },
    "totaleuiutkeringTotNuToe": {
      "type": "number",
      "minimum": 0,
      "required": true,
      "description": "Cumulative payments to contractor in calendar year (EUR)"
    },
    "wntNormJaar": {
      "type": "number",
      "minimum": 0,
      "required": true,
      "description": "Applicable WNT salary cap for the year (EUR)"
    },
    "percentageNormBereikt": {
      "type": "number",
      "minimum": 0,
      "maximum": 300,
      "required": true,
      "description": "% of WNT norm reached (e.g., 85)"
    },
    "vrijstellingVanToepassing": {
      "type": "boolean",
      "required": true,
      "description": "Whether short-term exemption applies (<12 months)"
    },
    "publicatieplichtGetriggerd": {
      "type": "boolean",
      "required": true,
      "description": "Whether >100% of norm, triggering publication requirement"
    }
  }
}
```

#### 5. InhuurRatio (Aggregated Metric)

Per cost-center per quarter: external vs. internal hiring proportion.

```json
{
  "name": "InhuurRatio",
  "slug": "inhuur-ratio",
  "properties": {
    "ratioId": {
      "type": "string",
      "format": "uuid",
      "required": true
    },
    "kostenplaats": {
      "type": "string",
      "required": true,
      "description": "Cost center code"
    },
    "kwartaal": {
      "type": "string",
      "enum": ["Q1", "Q2", "Q3", "Q4"],
      "required": true
    },
    "jaar": {
      "type": "integer",
      "minimum": 2020,
      "required": true
    },
    "aantalFteIntern": {
      "type": "number",
      "minimum": 0,
      "description": "Internal FTE from HRMQ"
    },
    "aantalFteExtern": {
      "type": "number",
      "minimum": 0,
      "description": "External FTE from active InhuurOpdrachten"
    },
    "kostenIntern": {
      "type": "number",
      "minimum": 0,
      "description": "Payroll cost (EUR)"
    },
    "kostenExtern": {
      "type": "number",
      "minimum": 0,
      "description": "Total invoice cost for external hires (EUR)"
    },
    "ratioPercentage": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "External / (internal + external) * 100"
    },
    "drempelwaarde": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Organizational norm max % external"
    },
    "overschredenJaNee": {
      "type": "boolean",
      "description": "Ratio > threshold"
    }
  }
}
```

## Schema Relationships

```
InhuurOpdracht 1 ── n DbaAssessment (assessment history)
InhuurOpdracht 1 ── 1 Opdrachtnemer (hire vendor)
InhuurOpdracht 1 ── n WntToets (one per calendar year if WNT-relevant)
InhuurOpdracht 1 ── n Shillinq Invoice (cross-app, via cost center + vendor)
Opdrachtnemer 1 ── n InhuurOpdracht (vendor has multiple active/past contracts)
InhuurRatio n ── 1 kostenplaats (derived from active InhuurOpdracht set)
```

## Seed Data

### Example 1: ZZP Data Architect (Green Assessment)

**Opdrachtnemer**:
```json
{
  "@self": {
    "register": "inhuur-opdrachtnemers",
    "schema": "Opdrachtnemer",
    "slug": "zzp-data-arch-001"
  },
  "naam": "Ir. Jan de Wit",
  "kvkNummer": "82234567",
  "btwnummer": "NL821234567B01",
  "geboorteDatum": "1978-06-15",
  "woonAdres": {
    "straat": "Kruisplein",
    "huisnummer": "42",
    "postcode": "3012AB",
    "plaats": "Rotterdam"
  },
  "zakelijkAdres": {
    "straat": "Meelbaan",
    "huisnummer": "3",
    "postcode": "3821EA",
    "plaats": "Amersfoort"
  },
  "iban": "NL38ABNA0504431926",
  "modelovereenkomstType": "belastingdienst-standaard",
  "modelovereenkomstKenmerk": "BD-2023-001-SER",
  "datumLaatsteModelovereenkomstCheck": "2023-09-15",
  "aantalOpdrachtgeversLaatste12mnd": 3
}
```

**InhuurOpdracht**:
```json
{
  "@self": {
    "register": "inhuur-opdrachten",
    "schema": "InhuurOpdracht",
    "slug": "hdo-2026-001-zzp"
  },
  "opdrachtNummer": "HDO-2026-001-ZZP",
  "opdrachtgever": "emp-012345",
  "opdrachtnemer": "zzp-data-arch-001",
  "opdrachtOmschrijving": "Data architect, enterprise data governance, 6 months 32 hrs/week",
  "inhuurType": "zzp",
  "startDatum": "2026-03-01",
  "geplande EindDatum": "2026-09-01",
  "verlengingsOptie": false,
  "uurtarief": 95.00,
  "geschatTotaalbudget": 59520.00,
  "kostenplaats": "CC-IT-INFRA",
  "projectCode": "PRJ-DATA-2026",
  "dbaConlusie": "groen",
  "wntRelevant": false,
  "status": "actief"
}
```

**DbaAssessment**:
```json
{
  "@self": {
    "register": "dba-assessments",
    "schema": "DbaAssessment",
    "slug": "dba-hdo-2026-001"
  },
  "opdrachtId": "hdo-2026-001-zzp",
  "assessmentDatum": "2026-02-20T14:30:00Z",
  "beoordelaar": "user-inkoper-maria@org.nl",
  "risicoscore": 18,
  "conclusie": "groen",
  "advies": "Low risk: independent architect with own tools, multiple clients, time-limited project scope, no direct supervision.",
  "escalatieVereist": false
}
```

### Example 2: Temporary Staffing (Red Assessment, Requires Authorization)

**Opdrachtnemer** (Agency):
```json
{
  "@self": {
    "register": "inhuur-opdrachtnemers",
    "schema": "Opdrachtnemer",
    "slug": "uitzend-bureau-001"
  },
  "naam": "StaffForce Interim BV",
  "kvkNummer": "67345678",
  "btwnummer": "NL673456789B02",
  "zakelijkAdres": {
    "straat": "Wibautstraat",
    "huisnummer": "120",
    "postcode": "1091GR",
    "plaats": "Amsterdam"
  },
  "iban": "NL42RABO0300065264",
  "waadiRegistratie": true,
  "waadiNummer": "UWV-WG/20245/1",
  "gRekeningnummer": "G-REK-001234567"
}
```

**InhuurOpdracht**:
```json
{
  "@self": {
    "register": "inhuur-opdrachten",
    "schema": "InhuurOpdracht",
    "slug": "hdo-2026-002-uitzend"
  },
  "opdrachtNummer": "HDO-2026-002-UIT",
  "opdrachtgever": "emp-012346",
  "opdrachtnemer": "uitzend-bureau-001",
  "opdrachtOmschrijving": "Junior administrateur, 3 maanden, 40 hrs/week, daily management",
  "inhuurType": "uitzend",
  "startDatum": "2026-04-15",
  "geplande EindDatum": "2026-07-15",
  "verlengingsOptie": true,
  "uurtarief": 28.50,
  "geschatTotaalbudget": 19080.00,
  "kostenplaats": "CC-HR-ADMIN",
  "dbaConlusie": "rood",
  "wntRelevant": true,
  "status": "in-onderzoek",
  "red_overrideAutorisatie": {
    "goedgekeurd": true,
    "autoriseerderAfdeling": "user-afdeling-sjaak@org.nl",
    "autoriseerderController": "user-controller-anna@org.nl",
    "autoriseerderBestuurder": "user-bestuurder-dirk@org.nl",
    "datumGoedkeuring": "2026-04-01T09:15:00Z",
    "motivering": "Urgent operational need; staffing agency has established compliance track record; 3-month term limits exposure."
  }
}
```

**DbaAssessment**:
```json
{
  "@self": {
    "register": "dba-assessments",
    "schema": "DbaAssessment",
    "slug": "dba-hdo-2026-002"
  },
  "opdrachtId": "hdo-2026-002-uitzend",
  "assessmentDatum": "2026-03-28T10:45:00Z",
  "beoordelaar": "user-inkoper-tom@org.nl",
  "risicoscore": 72,
  "conclusie": "rood",
  "advies": "High risk: staffing agency employee; direct daily management; no independent business risk; client determines work schedule. Strong indicators of hidden employment. Requires three-signature override and active monitoring.",
  "escalatieVereist": true
}
```

### Example 3: Consultancy (Orange Assessment)

**InhuurOpdracht**:
```json
{
  "@self": {
    "register": "inhuur-opdrachten",
    "schema": "InhuurOpdracht",
    "slug": "hdo-2026-003-cons"
  },
  "opdrachtNummer": "HDO-2026-003-CONS",
  "opdrachtgever": "emp-012347",
  "opdrachtnemer": "consultant-firm-002",
  "opdrachtOmschrijving": "Change management consulting, digital transformation, 4 months",
  "inhuurType": "consultancy",
  "startDatum": "2026-05-01",
  "geplande EindDatum": "2026-09-01",
  "verlengingsOptie": false,
  "uurtarief": 150.00,
  "geschatTotaalbudget": 96000.00,
  "kostenplaats": "CC-STRATEGY",
  "dbaConlusie": "oranje",
  "wntRelevant": false,
  "status": "goedgekeurd"
}
```

### Example 4: WNT Tracking

**WntToets**:
```json
{
  "@self": {
    "register": "wnt-toetsen",
    "schema": "WntToets",
    "slug": "wnt-2026-hdartsen-001"
  },
  "opdrachtId": "hdo-2026-002-uitzend",
  "kalenderJaar": 2026,
  "totaleuiutkeringTotNuToe": 38160.00,
  "wntNormJaar": 150000.00,
  "percentageNormBereikt": 25,
  "vrijstellingVanToepassing": true,
  "publicatieplichtGetriggerd": false
}
```

### Example 5: Hiring Ratio Dashboard

**InhuurRatio** (Q1 2026):
```json
{
  "@self": {
    "register": "inhuur-ratios",
    "schema": "InhuurRatio",
    "slug": "ratio-cc-it-infra-q1-2026"
  },
  "kostenplaats": "CC-IT-INFRA",
  "kwartaal": "Q1",
  "jaar": 2026,
  "aantalFteIntern": 12.5,
  "aantalFteExtern": 2.0,
  "kostenIntern": 625000.00,
  "kostenExtern": 119040.00,
  "ratioPercentage": 16.0,
  "drempelwaarde": 25,
  "overschredenJaNee": false
}
```

## Service Integration Points

### OpenRegister Usage
- **ObjectService**: CRUD for InhuurOpdracht, Opdrachtnemer, DbaAssessment, WntToets, InhuurRatio.
- **SchemaService**: Validation for all incoming data (model agreements, IBAN, postcode format).
- **ImportService**: Seed data loading on app install.
- **AuditTrailService**: Immutable logging of all changes (REQ-010).

### Shillinq Integration
- Webhook listener on invoice creation/payment.
- Extract ZZP flag from Shillinq dba-compliance-marker.
- Aggregate to InhuurOpdracht.daadwerkelijkeUitgaven.
- Trigger WNT cap check (REQ-003) and ratio recalculation.

### OpenConnector KvK Adapter
- Lookup on Opdrachtnemer creation (vendor selection form).
- WAADI validation for inhuurType="uitzend" or "detachering" (REQ-005).
- Annual re-check on contract anniversary.
- Return: name, address, status, WAADI registration flag.

### HRMQ Personnel Register
- Query personnel FTE and salaries for InhuurRatio calculation (REQ-006).
- Filter by kostenplaats to isolate cost-center hiring data.

### Decidesk Decision Workflow
- Three-signature approval chain for red DBA (REQ-007).
- Store approvals in InhuurOpdracht.red_overrideAutorisatie.
- Lock contract status until all three signatures obtained.

### Docudesk Dossier Vault
- Archive completed InhuurOpdracht + attachments (contracts, assessments, WNT disclosures).
- Retention policy: 7 years (Belastingdienst audit window).

## Reuse Analysis

**Existing Platform Services Leveraged**:
- **ObjectService** + **CnDetailPage** for contract CRUD (standard detail form).
- **CnFormDialog** for inline DBA assessment questions (schema-driven).
- **ImportService** + **ExportService** for annual WNT report (JSON/XML export).
- **AuditTrailService** for immutable change log (REQ-010, standard audit tab).
- **CnChartWidget** + **CnStatsBlock** for hiring ratio dashboard (line chart, KPI card).
- **NotificationService** for WNT threshold warnings (REQ-003).
- **AuthorizationService** for role-based access (controller, executive, auditor roles).

**No Custom Duplication**: All CRUD, list, search, and export handled by platform. No custom controllers, forms, or data layers needed beyond domain service logic (DBA scoring, WNT aggregation, ratio calculation).

## Deduplication Check

- **DBA Scoring Logic**: No existing DBA assessment or risk-scoring service in OpenRegister. Custom scoring engine required (domain-specific, Belastingdienst 12-question framework).
- **WNT Wage-Cap Tracking**: No existing WNT thresholding or publication record generation. Custom WNT service required.
- **WAADI Validation**: Leverage existing OpenConnector KvK adapter; no duplication.
- **Hiring Ratio Aggregation**: Relies on ObjectService queries + HRMQ cross-app lookup; no custom aggregation framework needed (SQL view + scheduled job suffices).
- **Three-Signature Approval**: Leverage existing Decidesk workflow engine; no custom approval logic.
- **Immutable Audit Log**: Fully provided by AuditTrailService (standard).

**Conclusion**: No overlap with existing OpenRegister, Shillinq, HRMQ, Decidesk, or Docudesk services. All required custom logic is domain-specific (DBA, WNT) and scoped to this spec.

## Architectural Decisions

1. **OpenRegister-First Data**: All hiring, assessment, and WNT data in OpenRegister (ADR-001). No custom persistence layer.

2. **Event-Driven Aggregation**: Shillinq invoice webhook triggers WNT and ratio recalculation (eventual consistency, not real-time locks).

3. **Schema-Driven Validation**: Postcode, IBAN, date formats validated via OpenRegister schema types (ADR-011). No custom string validators.

4. **Three-Signature as Decidesk Workflow**: Rather than custom approval logic, use existing Decidesk decision engine (ADR-023 action authorization).

5. **Immutable Audit as AuditTrailService**: No custom audit logging; rely on platform AuditTrailService (automatic on save).

6. **KvK Adapter for WAADI**: Outsource vendor validation to existing OpenConnector service; no embedded KvK client.

7. **Hiring Ratio as Scheduled Job**: Recalculate InhuurRatio records nightly via a scheduled service (not real-time widget calculation) to avoid expensive joins on every page load.

8. **Annual Export via Docudesk**: After WNT report generation, archive contract + assessment + WNT record to Docudesk for 7-year retention and auditor access.

## Migration & Rollout

### Phase 1: Foundation (Weeks 1-6)
- Schemas and seed data in OpenRegister.
- DBA assessment service (12-question scoring).
- Contract CRUD forms (use CnDetailPage + CnFormDialog).

### Phase 2: Compliance Checks (Weeks 7-12)
- Shillinq webhook integration (invoice aggregation).
- WNT cap monitoring (threshold alerts).
- Model agreement validation (KvK + form UX).
- WAADI validation on vendor selection.

### Phase 3: Governance & Export (Weeks 13-20)
- Three-signature flow (Decidesk integration).
- Hiring ratio dashboard (CnChartWidget).
- End-of-contract checklist (CnDetailPage task section).
- Annual board report export (template + PDF/XML).

### Phase 4: Audit Hardening (Weeks 21-26)
- Immutable audit trail verification (log export).
- Docudesk archival workflow.
- Tax audit response playbook (docs + training).
