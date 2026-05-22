---
status: proposed
app: purchaseq
spec: bibob-toetsing-leveranciers
version: 1.0
---

# Design — BIBOB Toetsing Leveranciers en Opdrachtnemers

## Entity Schemas

All entities stored in OpenRegister with schema.org vocabulary. Schemas defined in `lib/Settings/bibob_register.json`.

### BibobDossier

Core entity per investigation.

```json
{
  "$schema": "https://schema.org/Thing",
  "type": "object",
  "title": "BIBOB Dossier",
  "description": "Investigation file for supplier integrity assessment",
  "properties": {
    "dossiernummer": {
      "type": "string",
      "title": "Dossier Number",
      "pattern": "^BIBOB-[0-9]{4}-[0-9]{6}$",
      "description": "Unique identifier (BIBOB-YYYY-XXXXXX)"
    },
    "aanleiding": {
      "type": "string",
      "enum": ["aanbesteding", "vergunning", "subsidie"],
      "title": "Trigger Type",
      "description": "What triggered the investigation"
    },
    "aanleiding_referentie": {
      "type": "string",
      "title": "Trigger Reference",
      "description": "Purchase order ID, permit ID, or subsidy ID"
    },
    "risico_sector_hoog": {
      "type": "boolean",
      "title": "High-Risk Sector",
      "description": "Bouw, horeca, afval, vastgoed, transport"
    },
    "risico_opvallende_prijs": {
      "type": "boolean",
      "title": "Unusual Price Risk",
      "description": "Price significantly below market"
    },
    "risico_atypische_structuur": {
      "type": "boolean",
      "title": "Atypical Structure Risk",
      "description": "Complex ownership or shell structure"
    },
    "risico_recente_overdracht": {
      "type": "boolean",
      "title": "Recent Transfer Risk",
      "description": "Ownership changed <24 months"
    },
    "onderzoeksfase": {
      "type": "string",
      "enum": ["geen", "formulier-1", "formulier-2", "lbb-advies", "besluit"],
      "title": "Investigation Phase"
    },
    "startdatum": {
      "type": "string",
      "format": "date",
      "title": "Start Date"
    },
    "deadline_datum": {
      "type": "string",
      "format": "date",
      "title": "Target Completion Date"
    },
    "status": {
      "type": "string",
      "enum": ["open", "onderzoek", "advies", "besloten", "gesloten"],
      "title": "Status",
      "description": "Workflow state of the dossier"
    },
    "conclusie": {
      "type": "string",
      "enum": ["geen-bezwaar", "lichte-mate", "ernstige-mate-van-gevaar"],
      "title": "Conclusion",
      "description": "Final integrity assessment"
    },
    "besluit_link": {
      "type": "string",
      "format": "uri",
      "title": "Decision Link",
      "description": "Reference to BibobBesluit entity"
    },
    "toegang_restricted": {
      "type": "string",
      "enum": ["all", "integriteit_jurist_bestuur", "jurist_bestuur"],
      "title": "Access Restriction",
      "description": "Privacy-level for sensitive data"
    }
  },
  "required": ["dossiernummer", "aanleiding", "startdatum"]
}
```

### BibobBetrokkene

Person or organization under investigation.

```json
{
  "$schema": "https://schema.org/Person",
  "type": "object",
  "title": "BIBOB Involved Party",
  "description": "Natural or legal person subject of investigation",
  "properties": {
    "type": {
      "type": "string",
      "enum": ["aanvrager", "bestuurder", "aandeelhouder", "ubo", "financier", "leidinggevende", "zakelijke-relatie"],
      "title": "Party Type"
    },
    "naam": {
      "type": "string",
      "title": "Name",
      "minLength": 1
    },
    "kvk_nummer": {
      "type": "string",
      "pattern": "^[0-9]{8}$",
      "title": "Chamber of Commerce Number",
      "description": "Dutch KvK number for legal persons"
    },
    "bsn": {
      "type": "string",
      "title": "Citizen ID Number",
      "description": "Encrypted; highly restricted access"
    },
    "geboortedatum": {
      "type": "string",
      "format": "date",
      "title": "Date of Birth"
    },
    "woonadres": {
      "type": "string",
      "title": "Address",
      "description": "Encrypted storage"
    },
    "relatie_tot_aanvrager": {
      "type": "string",
      "title": "Relation to Applicant"
    },
    "mate_van_zeggenschap_percentage": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "title": "Control Percentage",
      "description": "Ownership/control share"
    },
    "dossier_id": {
      "type": "string",
      "title": "Dossier Reference"
    }
  },
  "required": ["type", "naam"]
}
```

### BibobFormulier1

Self-assessment questionnaire.

```json
{
  "$schema": "https://schema.org/Form",
  "type": "object",
  "title": "BIBOB Form 1 (Self-Assessment)",
  "description": "Initial integrity assessment by government body",
  "properties": {
    "dossier_id": {
      "type": "string",
      "title": "Dossier Reference"
    },
    "ingevuld_door": {
      "type": "string",
      "title": "Completed By",
      "description": "User identifier of coordinator"
    },
    "ingevuld_op": {
      "type": "string",
      "format": "date-time",
      "title": "Completed Date"
    },
    "antwoorden_json": {
      "type": "object",
      "title": "Responses",
      "description": "Standardized questionnaire responses (financiering, strafbare feiten, schulden, structuur, relaties)",
      "additionalProperties": true
    },
    "bijgevoegde_stukken": {
      "type": "array",
      "title": "Attached Documents",
      "description": "File references (KvK extract, financial statements, financing agreements)",
      "items": {
        "type": "object",
        "properties": {
          "bestand_id": {"type": "string"},
          "titel": {"type": "string"},
          "upload_datum": {"type": "string", "format": "date"}
        }
      }
    },
    "eerste_indicatie": {
      "type": "string",
      "enum": ["geen-twijfel", "twijfel", "sterke-twijfel"],
      "title": "Initial Assessment"
    }
  },
  "required": ["dossier_id", "ingevuld_op", "eerste_indicatie"]
}
```

### BibobFormulier2

In-depth investigation.

```json
{
  "$schema": "https://schema.org/Thing",
  "type": "object",
  "title": "BIBOB Form 2 (In-Depth Assessment)",
  "description": "Extended investigation when initial assessment shows risk",
  "properties": {
    "dossier_id": {
      "type": "string",
      "title": "Dossier Reference"
    },
    "aanvullende_vragen": {
      "type": "array",
      "title": "Supplementary Questions",
      "items": {"type": "string"}
    },
    "openbare_bronnen_geraadpleegd": {
      "type": "array",
      "title": "Public Sources Consulted",
      "description": "Bankruptcy Register, BIG register, convictions, media sources",
      "items": {
        "type": "object",
        "properties": {
          "bron": {"type": "string"},
          "datum": {"type": "string", "format": "date"},
          "bevindingen": {"type": "string"}
        }
      }
    },
    "zienswijze_betrokkene": {
      "type": "object",
      "title": "Party's Position",
      "description": "Response from involved party to draft conclusions",
      "properties": {
        "ontvangen_datum": {"type": "string", "format": "date"},
        "document": {"type": "string"}
      }
    },
    "voorlopige_conclusie": {
      "type": "string",
      "enum": ["geen-bezwaar", "lichte-mate", "ernstige-mate-van-gevaar"],
      "title": "Preliminary Conclusion"
    }
  },
  "required": ["dossier_id", "voorlopige_conclusie"]
}
```

### LbbAdviesaanvraag

Request to National Bureau for Integrity Assessment (LBB).

```json
{
  "$schema": "https://schema.org/Thing",
  "type": "object",
  "title": "LBB Advice Request",
  "description": "Request to National Bureau for Integrity Assessment when unclear",
  "properties": {
    "dossier_id": {
      "type": "string",
      "title": "Dossier Reference"
    },
    "aanvraagdatum": {
      "type": "string",
      "format": "date",
      "title": "Request Date"
    },
    "aanvraag_tekst": {
      "type": "string",
      "title": "Request Text",
      "description": "Structured justification for LBB advice"
    },
    "verzending_bevestiging": {
      "type": "string",
      "format": "date-time",
      "title": "Sending Confirmation"
    },
    "lbb_zaaknummer": {
      "type": "string",
      "title": "LBB Case Number"
    },
    "advies_datum": {
      "type": "string",
      "format": "date",
      "title": "Advice Date"
    },
    "advies_tekst": {
      "type": "string",
      "title": "Advice Text",
      "description": "Encrypted, highly restricted access"
    },
    "advies_conclusie": {
      "type": "string",
      "enum": ["geen-bezwaar", "lichte-mate", "ernstige-mate-van-gevaar"],
      "title": "Advice Conclusion"
    },
    "geldigheidsduur": {
      "type": "integer",
      "title": "Validity Period (Months)",
      "description": "Max 24 months"
    }
  },
  "required": ["dossier_id", "aanvraagdatum"]
}
```

### BibobBesluit

Formal administrative decision.

```json
{
  "$schema": "https://schema.org/Thing",
  "type": "object",
  "title": "BIBOB Decision",
  "description": "Formal administrative decision with Awb Article 3:46 reasoning",
  "properties": {
    "dossier_id": {
      "type": "string",
      "title": "Dossier Reference"
    },
    "besluit_type": {
      "type": "string",
      "enum": ["gunning", "niet-gunning", "weigering", "intrekking", "voorwaardelijk"],
      "title": "Decision Type"
    },
    "feitelijke_grondslag": {
      "type": "string",
      "title": "Factual Basis",
      "description": "Facts on which decision rests (min. 250 chars)"
    },
    "juridische_grondslag": {
      "type": "string",
      "title": "Legal Basis",
      "description": "Legal grounds (Wet Bibob articles, Awb)",
      "minLength": 50
    },
    "belangenafweging": {
      "type": "string",
      "title": "Weighing of Interests",
      "description": "How competing interests were balanced",
      "minLength": 100
    },
    "proportionaliteitstoets": {
      "type": "string",
      "title": "Proportionality Test",
      "description": "Why decision is proportionate to the risk",
      "minLength": 100
    },
    "motivering": {
      "type": "string",
      "title": "Full Reasoning",
      "description": "Complete Awb-compliant explanation"
    },
    "ondertekenaar": {
      "type": "string",
      "title": "Signatory",
      "description": "Mandated decision-maker"
    },
    "datum_uitreiking": {
      "type": "string",
      "format": "date",
      "title": "Date of Service"
    },
    "ontvangstbevestiging": {
      "type": "string",
      "title": "Receipt Confirmation",
      "description": "Proof of delivery to party"
    },
    "bezwaartermijn_startdatum": {
      "type": "string",
      "format": "date",
      "title": "Objection Period Start",
      "description": "6-week countdown for objections"
    }
  },
  "required": ["dossier_id", "besluit_type", "feitelijke_grondslag", "juridische_grondslag", "proportionaliteitstoets", "ondertekenaar"]
}
```

### BibobBewaarregel

Compliance-derived retention schedule.

```json
{
  "$schema": "https://schema.org/Thing",
  "type": "object",
  "title": "BIBOB Retention Rule",
  "description": "Calculated destruction date and anonymization schedule",
  "properties": {
    "dossier_id": {
      "type": "string",
      "title": "Dossier Reference"
    },
    "bewaarduur_basis": {
      "type": "string",
      "enum": ["5-jaar-na-sluiting", "5-jaar-na-onherroepelijk-besluit"],
      "title": "Retention Base",
      "description": "Legal basis for retention period"
    },
    "vernietigingsdatum": {
      "type": "string",
      "format": "date",
      "title": "Destruction Date",
      "description": "Calculated end-of-life date"
    },
    "anonimisering_geplanned": {
      "type": "string",
      "format": "date",
      "title": "Anonymization Date",
      "description": "When PII removal begins"
    },
    "archivaris_tweede_handtekening": {
      "type": "string",
      "title": "Archivist Second Signature",
      "description": "Required confirmation for destruction"
    },
    "vernietigingslog": {
      "type": "string",
      "title": "Destruction Log",
      "description": "Indelible record of what was destroyed when"
    }
  },
  "required": ["dossier_id", "bewaarduur_basis", "vernietigingsdatum"]
}
```

## Seed Data

Example objects per schema with realistic Dutch values.

### BibobDossier Examples

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobDossier",
    "slug": "dossier-2024-bouwbedrijf-midden"
  },
  "dossiernummer": "BIBOB-2024-001234",
  "aanleiding": "aanbesteding",
  "aanleiding_referentie": "INK-2024-4567",
  "risico_sector_hoog": true,
  "risico_opvallende_prijs": false,
  "risico_atypische_structuur": false,
  "risico_recente_overdracht": false,
  "onderzoeksfase": "formulier-1",
  "startdatum": "2024-03-15",
  "deadline_datum": "2024-05-15",
  "status": "onderzoek",
  "conclusie": null,
  "toegang_restricted": "integriteit_jurist_bestuur"
}
```

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobDossier",
    "slug": "dossier-2024-horeca-amsterdam"
  },
  "dossiernummer": "BIBOB-2024-001235",
  "aanleiding": "vergunning",
  "aanleiding_referentie": "VGM-2024-8901",
  "risico_sector_hoog": true,
  "risico_opvallende_prijs": true,
  "risico_atypische_structuur": true,
  "risico_recente_overdracht": true,
  "onderzoeksfase": "formulier-2",
  "startdatum": "2024-02-10",
  "deadline_datum": "2024-04-10",
  "status": "onderzoek",
  "conclusie": null,
  "toegang_restricted": "jurist_bestuur"
}
```

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobDossier",
    "slug": "dossier-2024-transport-rotterdam"
  },
  "dossiernummer": "BIBOB-2024-001236",
  "aanleiding": "aanbesteding",
  "aanleiding_referentie": "INK-2024-5678",
  "risico_sector_hoog": true,
  "risico_opvallende_prijs": false,
  "risico_atypische_structuur": true,
  "risico_recente_overdracht": false,
  "onderzoeksfase": "besluit",
  "startdatum": "2023-11-20",
  "deadline_datum": "2024-02-20",
  "status": "gesloten",
  "conclusie": "lichte-mate",
  "toegang_restricted": "all"
}
```

### BibobBetrokkene Examples

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobBetrokkene",
    "slug": "betrokkene-construct-bv"
  },
  "type": "aanvrager",
  "naam": "Construct B.V.",
  "kvk_nummer": "12345678",
  "bsn": null,
  "geboortedatum": null,
  "woonadres": "Bouwlaan 42, 3054 XH Rotterdam",
  "relatie_tot_aanvrager": "Hoofdaanvrager",
  "mate_van_zeggenschap_percentage": 100,
  "dossier_id": "BIBOB-2024-001234"
}
```

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobBetrokkene",
    "slug": "betrokkene-jan-pieterzoon"
  },
  "type": "bestuurder",
  "naam": "Jan Pieterzoon",
  "kvk_nummer": null,
  "bsn": "ENCRYPTED",
  "geboortedatum": "1968-05-12",
  "woonadres": "ENCRYPTED",
  "relatie_tot_aanvrager": "Directeur Construct B.V.",
  "mate_van_zeggenschap_percentage": 75,
  "dossier_id": "BIBOB-2024-001234"
}
```

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobBetrokkene",
    "slug": "betrokkene-cafe-zonnegracht"
  },
  "type": "aanvrager",
  "naam": "Café Zonnegracht",
  "kvk_nummer": "23456789",
  "bsn": null,
  "geboortedatum": null,
  "woonadres": "Zonnegracht 15, 1015 SJ Amsterdam",
  "relatie_tot_aanvrager": "Permitaanvrager",
  "mate_van_zeggenschap_percentage": 100,
  "dossier_id": "BIBOB-2024-001235"
}
```

### BibobFormulier1 Example

```json
{
  "@self": {
    "register": "bibob",
    "schema": "BibobFormulier1",
    "slug": "formulier1-dossier-2024-001234"
  },
  "dossier_id": "BIBOB-2024-001234",
  "ingevuld_door": "coordinator-001",
  "ingevuld_op": "2024-03-20T14:30:00Z",
  "antwoorden_json": {
    "financiering_bronnen": "Leningsfonds ABN AMRO, bedrag €500.000, afgesloten 2023-06",
    "strafbare_feiten": "Nee, geen eerdere veroordelingen bekend",
    "fiscale_schulden": "Nee, fiscale situatie schoon",
    "ondernemingsstructuur": "B.V., geopend 2019, één directeur, twee aandeelhouders",
    "zakelijke_relaties": "Reguliere toeleveranciers bouw, geen opvallende partners"
  },
  "bijgevoegde_stukken": [
    {
      "bestand_id": "file-001",
      "titel": "KvK-uittreksel Construct B.V.",
      "upload_datum": "2024-03-20"
    },
    {
      "bestand_id": "file-002",
      "titel": "Jaarrekening 2023",
      "upload_datum": "2024-03-20"
    }
  ],
  "eerste_indicatie": "geen-twijfel"
}
```

## UI & Integration Points

### Risk Detection Workflow

**Trigger:** New purchase order in purchaseq with value > €50k and sector in [bouw, horeca, afval, vastgoed, transport] OR supplier registered <24 months OR ownership change <24 months.

**Action:** Automatic notification to integrity coordinator with:
- Risk factors identified
- Supplier details (KvK number, ownership age, sector)
- Quick-link to create BIBOB dossier (auto-populates form-1)
- Link to relevant purchase order

### Form-1 Editor

**Component:** CnDetailPage with CnFormDialog for schema-driven form

**Fields:**
- Dossier metadata (trigger type, reference, risk indicators)
- Standardized question blocks (financing, criminal history, tax debts, structure, relations) with toggle/textarea fields
- File upload section for KvK extract, financial statements, financing agreements
- First-indication assessment (radio: geen-twijfel / twijfel / sterke-twijfel)
- Auto-populated KvK data from openconnector kvk-adapter

**Privacy:** No sensitive personal data exposed to regular purchaser users.

### Escalation & Form-2

**Component:** CnDetailPage with modal-driven state transition (ADR-004 compliant — modals in `src/modals/`)

**Decision Point:** If form-1 conclusion is "twijfel" or "sterke-twijfel":
- Option A: Escalate to Form-2 with mandatory detail text
- Option B: Close with detailed reasoning (recorded in auditlog)

**Form-2 Content:**
- Supplementary questions (extensible list)
- Consulted sources checklist (Bankruptcy Register, BIG, public convictions, media)
- Draft letter to involved party (Awb 3:46 zienswijze-procedure), 2-week response window
- Response capture (uploaded document or meeting notes)
- Preliminary conclusion selection

### Decision Workflow

**Component:** CnDetailPage with CnAdvancedFormDialog for decision template

**Mandatory fields:**
- Factual basis (min. 250 chars)
- Legal grounds (min. 50 chars)
- Weighing of interests (min. 100 chars)
- Proportionality test (min. 100 chars)
- Decision type: award / no-award / refuse / revoke / conditional
- Signatory (mandated organ from decidesk mandaatregister)

**Submission:** Digital signature via decidesk, calculates 6-week objection period, logs service date.

### Privacy-Aware Views

**Field-level RBAC:**
- Regular purchaser: dossier exists (count), final conclusion
- Integrity coordinator: all data
- Lawyer: form-1, form-2, drafts, decision prep
- Mandated official: all + signature authority
- Archivist: retention/destruction log

**Implementation:** PropertyRbacHandler per user role, encrypted-at-rest BSN/address fields.

### LBB Advice Request

**Component:** CnFormDialog (Form building)

**Required sections (enforced by min-length validation):**
- Investigation summary (own findings to date)
- Specific questions for LBB
- All involved parties and their roles
- Desired response date
- Proposed risk level if advice received

**Transmission:** Secure channel (OpenZaak SOAP API or dedicated LBB-connector ExApp).

**Receipt:** Advice stored encrypted, visible only to coordinator/lawyer/mandated official. Every view logged.

### Retention & Destruction Workflow

**Component:** CnDetailPage (Archival section)

**Lifecycle:**
- Dossier closed → BibobBewaarregel auto-created with destruction date (5 years post-close or post-unappealable decision)
- 60 days before destruction: archivist receives notification
- Destruction interface: confirm anonymization, second-sign, log action
- Post-destruction: statistics retained, PII purged, LBB advice deleted

**Integration:** docudesk destruction workflow; openzaak zaak lifecycle; audit trail indelible.

## Reuse Analysis (OpenRegister Abstractions)

This spec leverages and does NOT duplicate:

1. **ObjectService (CRUD)** — all BibobDossier, BibobBetrokkene, BibobFormulier1/2, LbbAdviesaanvraag, BibobBesluit creation/read/update
2. **CnDetailPage + CnFormDialog** — form-driven UI for all schemas
3. **PropertyRbacHandler** — field-level RBAC per user role (coordinator/lawyer/mandated/archivist)
4. **FileService** — document upload/download (KvK extracts, financial statements, LBB advice)
5. **AuditTrailService** — automatically tracks all state changes, decisions, approvals
6. **NotificationService** — automatic alerts to coordinator/lawyer on escalations and deadline approaches
7. **ArchivalService** — legal hold and destruction schedules for retention rules
8. **ImportService/ExportService** — bulk import of legacy BIBOB dossiers (CSV), export for audit/reporting
9. **openconnector kvk-adapter** — auto-enrich KvK number with company details, UBO register, sector classification
10. **decidesk** — BibobBesluit digital signature, mandaatregister lookup
11. **docudesk** — dossier file management, retention lifecycle, anonymization

No custom services are required for CRUD, state transitions, notifications, audit, retention, or file management. Custom integrations (openconnector adapter hand-off, decidesk signature flow, docudesk lifecycle timing) are documented in tasks.md.

## Declarative-vs-Imperative Decision

Per ADR-031, business logic is schema-declarative where OpenRegister extensions fit:

| Behaviour | Approach | Reason |
|---|---|---|
| Dossier status transitions (geen → formulier-1 → formulier-2 → lbb-advies → besluit) | `x-openregister-lifecycle` in BibobDossier schema | State machine + audit trail + RBAC-per-state + CloudEvents |
| Escalation guards (may escalate only if first-indicatie is "twijfel" or "sterke-twijfel") | PHP guard class `BibobEscalationGuard`, invoked by lifecycle `requires` | Complex business rule: check form-1 conclusion + optional motivering-if-declining |
| Retention-schedule calculation (destruction-date = close-date + 5 years or decision-date + 5 years) | `x-openregister-calculations` field `vernietigingsdatum` on BibobBewaarregel | Derived, always-current, no service round-trip |
| Deadline reminders (6-week objection period, 60-day pre-destruction alerts) | ScheduledWorkflow + n8n adapter (external) | Scheduled side-effects beyond what lifecycle provides |
| Document anonymization & PII removal | `ArchivalService.anonymizeObject()` call from destruction workflow | Uses platform's PII-detection and redaction; not duplicating custom logic |

No custom service classes are written. Coordinator actions (form-1 entry, escalation decision, decision drafting) are workflows, not automations.

## Security & Compliance Notes

- **BSN, woonadres:** Encrypted at rest, encrypted in transit. Field-level RBAC enforces read-only for authorized roles.
- **LBB advies:** Stored in encrypted container separate from dossier. Every access logged.
- **Awb Article 3:46:** Decision template enforces four mandatory reasoning blocks; minimum text lengths prevent hollow compliance.
- **Zienswijze-procedure:** Automated draft letter + termijn-blokking ensure Awb compliance.
- **Bewaarregels:** Destruction date calculated by law, second-signature required, action indelibly logged.
- **AVG compliance:** No sensitive data exported except in anonymized statistical form.
