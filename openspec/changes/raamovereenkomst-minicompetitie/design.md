---
status: design
created: 2026-05-23
---

# Design: Raamovereenkomst en Minicompetitie

## Architecture Overview

The capability spans three layers:

1. **Data layer**: Entity schemas for raamovereenkomst, suppliers, minicompetition lifecycle, scoring, extensions, and audit records
2. **Service layer**: Business logic for selection, validation, scoring consensus, volume tracking, and scheduled triggers
3. **UI/API layer**: REST endpoints for competition management, supplier portal, scoring interface, and audit export

### Data Model Entities

#### Schema: `raamovereenkomst`

Root object representing a framework agreement.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `nummer` | string | yes | Internal reference (e.g., RO-2024-IT-001), unique per organization |
| `titel` | string | yes | Framework title |
| `omschrijving` | text | yes | Detailed description |
| `modaliteit` | enum | yes | Values: `enkele_leverancier`, `meerdere_voorwaarden_vast`, `meerdere_minicompetitie` |
| `categorie` | enum | yes | Values: `leveringen`, `diensten`, `werken`, `sociale_diensten` |
| `cpvHoofdcode` | string | yes | Main CPV code (e.g., 72000000 for IT services) |
| `cpvBijcodes` | string[] | no | Additional CPV codes |
| `aanbesteder_id` | UUID | yes | FK to organisation |
| `startDatum` | date | yes | Framework start date |
| `eindDatum` | date | yes | Framework end date |
| `verlengingsopties` | JSON | no | Array of {duur_maanden, eenmalig_boolean, max_aantal_keer} |
| `looptijdJaren` | int | computed | Calculated; max 4 (klassieke) or 8 (speciale sectoren) |
| `maximumVolumeEur` | decimal | yes | Max budget for entire framework |
| `maximumAantalOpdrachten` | int | no | Max number of minicompetitions (optional) |
| `verbruiktVolumeEur` | decimal | computed | Sum of awarded minicompetitions |
| `geraamdeJaarlijksVolume` | decimal | no | Estimated annual consumption |
| `oorspronkelijkeAanbestedingId` | UUID | no | FK to source aanbesteding (if promoted) |
| `tendernedPublicatieId` | string | no | Reference to TenderNed notice ID |
| `status` | enum | yes | Values: `concept`, `gepubliceerd`, `actief`, `in_verlenging`, `geeindigd`, `opgezegd` |
| `eindigingsReden` | string | no | Reason for termination if status=geeindigd or opgezegd |
| `documentRefs` | JSON | no | Array of {naam, url} for RO document, terms, PvE, award decision |
| `created_at` | datetime | yes | Auto-set on creation |
| `created_by` | UUID | yes | FK to user (inkoper) |
| `updated_at` | datetime | yes | Auto-updated on change |
| `updated_by` | UUID | yes | FK to user who made change |

**Indexes:** `(nummer, aanbesteder_id)`, `(status, eindDatum)`, `(status, verbruiktVolumeEur)`

#### Schema: `raamovereenkomst_leverancier`

One row per active supplier per framework, tracking their participation.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `raamovereenkomst_id` | UUID | yes | FK to raamovereenkomst |
| `leverancier_kvk` | string | yes | KvK number of supplier |
| `leverancier_naam` | string | yes | Legal company name |
| `leverancier_vestigingsadres` | string | yes | Official address |
| `contactpersoon_naam` | string | yes | Procurement contact name |
| `contactpersoon_email` | string | yes | Email for minicompetition invitations |
| `contactpersoon_telefoon` | string | yes | Phone for urgent contact |
| `perceel` | string | no | Perceel code if per-perceel winners (e.g., "Perceel A") |
| `rangorde` | int | no | Rank for cascade modality (1=first choice, 2=fallback, etc.) |
| `aandeelPercentage` | decimal | no | Revenue share for allocation modality (0-100) |
| `geschiktheidsbewijzen_refs` | JSON | no | Array of {type, url} for UEA, VOG, professional cert, financial proof, technical capabilities |
| `verzekeringspolisRefs` | JSON | no | Array of {type, url} for liability, professional indemnity insurance |
| `actief` | boolean | yes | True if currently participating; false if removed |
| `wegvalDatum` | date | no | Date supplier became inactive (e.g., bankruptcy) |
| `wegvalReden` | string | no | Reason for removal (e.g., "faillissement", "opzegging door RO-eigenaar") |
| `verbruiktVolumePerLeverancierEur` | decimal | computed | Sum of awarded minicompetitions to this supplier |
| `maximumVolumePerLeverancierEur` | decimal | no | Optional cap per supplier (e.g., EUR 500k) |
| `gegundeMinicompetities` | int | computed | Count of awarded competitions |
| `created_at` | datetime | yes | Auto-set on creation |

**Indexes:** `(raamovereenkomst_id, leverancier_kvk)`, `(raamovereenkomst_id, actief)`

#### Schema: `minicompetitie`

One row per individual competition within a framework.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `raamovereenkomst_id` | UUID | yes | FK to raamovereenkomst |
| `nummer` | string | yes | Auto-generated (e.g., MC-2024-IT-001-042) |
| `titel` | string | yes | Competition title |
| `omschrijving` | text | yes | What is being procured |
| `aanvragende_afdeling` | string | yes | Department requesting |
| `aanvragende_user_id` | UUID | yes | FK to user who triggered |
| `triggerType` | enum | yes | Values: `nieuwe_behoefte`, `aflopend_contract`, `vervangingsinvestering`, `projectopdracht` |
| `perceel` | string | no | If framework has percelen, which one (e.g., "Perceel A") |
| `geraamde_waarde_eur` | decimal | yes | Estimated value |
| `gunningsmodel` | enum | yes | Values: `laagste_prijs`, `emvi_prijs_kwaliteit`, `emvi_prijs_kwaliteit_duurzaamheid`, `beste_kwaliteit_binnen_plafond` |
| `prijsplafond_eur` | decimal | no | Cap for "best quality within cap" model |
| `wegingsschema` | JSON | yes | {prijs_percentage, kwaliteit_percentage, duurzaamheid_percentage} (must sum to 100) |
| `kwaliteitscriteria` | JSON | yes | Array of {id, naam, omschrijving, max_punten, weging_percentage} |
| `duurzaamheidscriteria` | JSON | no | Array of {id, naam, omschrijving, max_punten, weging_percentage} (inherited from raamovereenkomst if available) |
| `inschrijvingsTermijnDagen` | int | yes | Days from invitation to deadline (e.g., 21) |
| `publicatieDatum` | datetime | yes | When invitations sent |
| `sluitingsDatum` | datetime | yes | Submission deadline (to the second) |
| `uitgenodige_leveranciers_ids` | UUID[] | yes | FK array to raamovereenkomst_leverancier rows |
| `selectieReden` | text | no | Reason if subset selected (min 50 chars if not all invited) |
| `status` | enum | yes | Values: `concept`, `uitgenodigd`, `ontvangst_inschrijvingen`, `in_beoordeling`, `gegund`, `gestaakt` |
| `beoordeling_commissie_ids` | UUID[] | yes | Min 3 user IDs for scoring committee |
| `winnaar_leverancier_id` | UUID | no | FK to raamovereenkomst_leverancier (null until awarded) |
| `gunnings_bedrag_eur` | decimal | no | Amount awarded to winner |
| `gunningsbeslissing_ref` | string | no | URL to generated award decision PDF |
| `contract_ref` | UUID | no | FK to generated contract (if auto-created) |
| `alcatel_deadline_date` | date | no | Computed 20 days after decision sent |
| `created_at` | datetime | yes | Auto-set on creation |
| `created_by` | UUID | yes | FK to user who created |
| `updated_at` | datetime | yes | Auto-updated |
| `updated_by` | UUID | yes | FK to user who updated |

**Indexes:** `(raamovereenkomst_id, status)`, `(raamovereenkomst_id, sluitingsDatum)`, `(aanvragende_user_id, status)`

#### Schema: `minicompetitie_inschrijving`

One row per bid submission per supplier.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `minicompetitie_id` | UUID | yes | FK to minicompetitie |
| `leverancier_id` | UUID | yes | FK to raamovereenkomst_leverancier |
| `ingediend_datum` | datetime | yes | Submission timestamp (to validate before deadline) |
| `inschrijvings_bedrag_eur` | decimal | yes | Price offered |
| `prijs_opbouw` | JSON | no | Line items [{ beschrijving, hoeveelheid, eenheidsprijs, subtotaal }] |
| `kwaliteitsscores` | JSON | no | Array of {criterium_id, score, motivering} (filled by scoring process) |
| `duurzaamheidscores` | JSON | no | Array of {criterium_id, score, motivering} |
| `totaalScore` | decimal | computed | Weighted final score after consensus |
| `documentRefs` | JSON | no | Array of {type, url} for quotation, price breakdown, quality answers, methodology |
| `tijdig` | boolean | yes | True if submitted before deadline |
| `geldigVerklaard` | boolean | no | True after committee formally declares it complete and valid |
| `geldigheidsReden` | text | no | If invalid, reason (e.g., "incomplete pricing", "late submission") |
| `vertrouwelijk_opslag` | boolean | yes | True = encrypted before opening ceremony; false = already decrypted |
| `created_at` | datetime | yes | Auto-set on submission |

**Indexes:** `(minicompetitie_id, ingediend_datum)`, `(minicompetitie_id, leverancier_id)`

#### Schema: `minicompetitie_beoordeling`

Scoring record per reviewer per submission per criterion.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `inschrijving_id` | UUID | yes | FK to minicompetitie_inschrijving |
| `beoordelaar_user_id` | UUID | yes | FK to scoring committee member |
| `criterium_id` | string | yes | Matches kwaliteitsscriteria[].id or duurzaamheidscriteria[].id |
| `score` | decimal | yes | 0-10 scale or max_punten from criterion |
| `motivering` | text | yes | Justification for score (required) |
| `consensus_score` | decimal | no | Final agreed score after discussion (null until consensus) |
| `consensus_motivering` | text | no | Final agreed justification |
| `flagged_for_discussion` | boolean | computed | True if divergence >2 points from other reviewers |
| `created_at` | datetime | yes | Auto-set on first score |
| `updated_at` | datetime | yes | Auto-updated |

**Indexes:** `(inschrijving_id, criterium_id)`, `(inschrijving_id, beoordelaar_user_id)`

#### Schema: `raamovereenkomst_verlenging`

One row per extension of a framework agreement.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `raamovereenkomst_id` | UUID | yes | FK to raamovereenkomst |
| `verlengings_nummer` | int | yes | Sequence number (1st, 2nd, etc.) |
| `voorgenomen_ingangsdat` | date | yes | Proposed start of extension |
| `voorgenomen_duur_maanden` | int | yes | Length of extension |
| `motivering` | text | yes | Why extend (contract performing well, market analysis, etc.) |
| `besluit` | enum | no | Values: `in_voorbereiding`, `akkoord`, `geweigerd`, `met_voorwaarden` |
| `besluit_door_user_id` | UUID | no | FK to user who approved/rejected |
| `besluit_datum` | date | no | When decision was made |
| `voorwaarden` | text | no | Conditions if `met_voorwaarden` |
| `nieuwe_eind_datum` | date | computed | Calculated as start + duur_maanden |
| `prijsindexatie_percentage` | decimal | no | CPI adjustment (e.g., 2.3) |
| `wijzigingsaankondiging_tendernedId` | string | no | If TenderNed notice published for wijziging |
| `created_at` | datetime | yes | Auto-set on creation |
| `created_by` | UUID | yes | FK to creator |

**Indexes:** `(raamovereenkomst_id, verlengings_nummer)`, `(raamovereenkomst_id, besluit)`

#### Schema: `minicompetitie_proces_verbaal`

Digitally-signed record of key ceremonial moments.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | UUID | yes | Primary key |
| `minicompetitie_id` | UUID | yes | FK to minicompetitie |
| `pvType` | enum | yes | Values: `opening_inschrijvingen`, `beoordeling`, `gunningsadvies` |
| `datum` | datetime | yes | When the ceremony occurred |
| `aanwezigen_user_ids` | UUID[] | yes | Attendees (committee members) |
| `inhoud` | text | yes | Full text of PV (auto-generated) |
| `bijlage_ref` | string | yes | URL to digitally-signed PDF |
| `ondertekenaars_sigs` | JSON | yes | Array of {user_id, signature, signedAt} |
| `created_at` | datetime | yes | Auto-set |
| `immutable` | boolean | yes | Always true; PVs cannot be edited once signed |

**Indexes:** `(minicompetitie_id, pvType)`, `(minicompetitie_id, datum)`

---

## Service Layer Logic

### Volume Calculation Service

```
CalculateFrameworkVolume(raamovereenkomst_id):
  verbruikt = SELECT SUM(gunnings_bedrag_eur)
              FROM minicompetitie m
              JOIN raamovereenkomst_leverancier rl ON m.winnaar_leverancier_id = rl.id
              WHERE m.raamovereenkomst_id = :id AND m.status = 'gegund'
  
  IF verbruikt >= 0.95 * RO.maximumVolumeEur:
    SEND ALERT to RO.created_by with percentage
  
  RETURN verbruikt
```

### Volume Validation on Award

```
ValidateAwardDoesNotExceedFrameworkMax(minicompetitie_id, gunningsAmount):
  RO = GET raamovereenkomst WHERE id = minicompetitie.raamovereenkomst_id
  currentVerbruikt = CalculateFrameworkVolume(RO.id)
  
  IF currentVerbruikt + gunningsAmount > RO.maximumVolumeEur:
    RETURN error("Award would exceed maximum. Options: (1) Cancel this competition, (2) File TenderNed wijziging, (3) Reduce scope/budget")
  
  RETURN success()
```

### Supplier Selection Validation

```
ValidateSupplierSelection(minicompetitie_id, selectedSupplierIds):
  selected_count = LENGTH(selectedSupplierIds)
  active_count = SELECT COUNT(*) FROM raamovereenkomst_leverancier
                 WHERE raamovereenkomst_id = MC.raamovereenkomst_id AND actief = true
  
  IF selected_count < active_count:
    # Subset selected; require reason
    IF NOT MC.selectieReden OR LENGTH(MC.selectieReden) < 50:
      RETURN error("Deselecting suppliers requires reason ≥50 characters")
  
  # Ensure no inactive suppliers selected
  inactive = SELECT * FROM raamovereenkomst_leverancier
             WHERE id IN (selectedSupplierIds) AND actief = false
  
  IF COUNT(inactive) > 0:
    RETURN error("Cannot invite inactive suppliers without admin override")
  
  RETURN success()
```

### Scoring Consensus Detection

```
DetectScoringDivergence(inschrijving_id, criterium_id):
  scores = SELECT score FROM minicompetitie_beoordeling
           WHERE inschrijving_id = :id AND criterium_id = :crit
           ORDER BY score
  
  divergence = MAX(scores) - MIN(scores)
  
  IF divergence > 2:
    FLAG inschrijving for discussion on this criterion
    SET consensus_score = NULL until agreement reached via dedicated UI
  
  RETURN divergence
```

### Gunning Model Validation

```
ValidateGunningsmodelWithinFrameworkMarges(minicompetitie_id):
  MC = GET minicompetitie WHERE id = :id
  RO = GET raamovereenkomst WHERE id = MC.raamovereenkomst_id
  
  # Check if chosen model is allowed by RO
  IF MC.gunningsmodel NOT IN RO.allowed_gunningsmodels:
    RETURN error(sprintf("Model %s not allowed by this framework", MC.gunningsmodel))
  
  # Check weighting sums to 100%
  total = MC.wegingsschema.prijs_percentage 
        + MC.wegingsschema.kwaliteit_percentage 
        + MC.wegingsschema.duurzaamheid_percentage
  
  IF total != 100:
    RETURN error("Weights must sum to 100%")
  
  # Check minimum 10% per criterion (proportionality)
  FOR EACH pct IN [prijs, kwaliteit, duurzaamheid]:
    IF pct > 0 AND pct < 10:
      RETURN error(sprintf("%s is %d%% but must be ≥10%% if included", pct.name, pct.value))
  
  RETURN success()
```

### Scheduled Extension Reminder

```
ScheduledJob_ExtensionReminders():
  expiring_frameworks = SELECT * FROM raamovereenkomst
                        WHERE status = 'actief'
                        AND DATE_DIFF(eindDatum, TODAY()) IN (180, 120, 60)
  
  FOR EACH ro IN expiring_frameworks:
    available_extensions = FILTER(ro.verlengingsopties 
                                   WHERE already_used_count < max_aantal_keer)
    
    SEND notification to ro.created_by:
      subject: "Framework {ro.nummer} expiring in {days} days"
      body: "Create extension or plan new procurement"
      cta: "Create extension" (pre-fills form with defaults from verlengingsopties)
```

### Auto-trigger on Contract Expiry

```
ScheduledJob_ContractExpiryTriggers():
  expiring_contracts = SELECT * FROM contract c
                       WHERE c.is_from_minicompetitie = true
                       AND DATE_DIFF(c.eindDatum, TODAY()) = 90
  
  FOR EACH contract IN expiring_contracts:
    mc = GET minicompetitie WHERE contract_ref = contract.id
    ro = GET raamovereenkomst WHERE id = mc.raamovereenkomst_id
    
    new_mc = CREATE minicompetitie:
      titel: sprintf("Renewal of %s", contract.titel)
      triggerType: "aflopend_contract"
      aanvragende_user_id: contract.created_by (original requester)
      status: "concept"
      raamovereenkomst_id: ro.id
    
    SEND notification to ro.created_by and contract.created_by
```

---

## Seed Data

### Sample Framework Agreement: RO-2024-INHUUR-001

```json
{
  "id": "ro-2024-001-uuid",
  "nummer": "RO-2024-INHUUR-001",
  "titel": "Raamovereenkomst Java-Ontwikkelaars 2024-2026",
  "omschrijving": "Framework for temporary staffing of experienced Java developers (min. 5 years), supporting core application development and legacy system maintenance.",
  "modaliteit": "meerdere_minicompetitie",
  "categorie": "diensten",
  "cpvHoofdcode": "72000000",
  "cpvBijcodes": ["72200000", "72400000"],
  "aanbesteder_id": "org-gemeente-xs",
  "startDatum": "2024-03-01",
  "eindDatum": "2026-03-01",
  "verlengingsopties": [
    {"duur_maanden": 12, "eenmalig": false, "max_aantal_keer": 2}
  ],
  "looptijdJaren": 2,
  "maximumVolumeEur": 2000000,
  "maximumAantalOpdrachten": null,
  "verbruiktVolumeEur": 1200000,
  "geraamdeJaarlijksVolume": 1000000,
  "oorspronkelijkeAanbestedingId": "aab-2023-034-uuid",
  "tendernedPublicatieId": "TN-2024-001234",
  "status": "actief",
  "created_at": "2024-02-15T10:00:00Z",
  "created_by": "user-inkoper-001"
}
```

### Sample Suppliers on RO-2024-INHUUR-001

```json
[
  {
    "id": "rl-001-uuid",
    "raamovereenkomst_id": "ro-2024-001-uuid",
    "leverancier_kvk": "12345678",
    "leverancier_naam": "TechStaff Uitzendbureau B.V.",
    "leverancier_vestigingsadres": "Techpark 42, 3600 ZX Utrecht",
    "contactpersoon_naam": "Jan Pieterzoon",
    "contactpersoon_email": "jan.pieterzoon@techstaff.nl",
    "contactpersoon_telefoon": "+31-30-2345678",
    "perceel": null,
    "rangorde": null,
    "aandeelPercentage": null,
    "geschiktheidsbewijzen_refs": [
      {"type": "UEA", "url": "https://register.uea.nl/techstaff-uitzend"},
      {"type": "VOG", "url": "https://assets.example.com/vog-techstaff-2024.pdf"}
    ],
    "actief": true,
    "verbruiktVolumePerLeverancierEur": 480000,
    "gegundeMinicompetities": 4
  },
  {
    "id": "rl-002-uuid",
    "raamovereenkomst_id": "ro-2024-001-uuid",
    "leverancier_kvk": "87654321",
    "leverancier_naam": "CodeForce Consulting",
    "leverancier_vestigingsadres": "Innovation Avenue 7, 1012 AB Amsterdam",
    "contactpersoon_naam": "Maria Sandberg",
    "contactpersoon_email": "maria@codeforce.nl",
    "contactpersoon_telefoon": "+31-20-9876543",
    "perceel": null,
    "rangorde": null,
    "aandeelPercentage": null,
    "geschiktheidsbewijzen_refs": [
      {"type": "UEA", "url": "https://register.uea.nl/codeforce"}
    ],
    "actief": true,
    "verbruiktVolumePerLeverancierEur": 720000,
    "gegundeMinicompetities": 3
  },
  {
    "id": "rl-003-uuid",
    "raamovereenkomst_id": "ro-2024-001-uuid",
    "leverancier_kvk": "13579246",
    "leverancier_naam": "Staffing Solutions Oost",
    "leverancier_vestigingsadres": "Bedrijventerrein 'De Zijdebaan' 101, 6900 AA Zevenaar",
    "contactpersoon_naam": "Peter Maas",
    "contactpersoon_email": "peter@staffoost.nl",
    "contactpersoon_telefoon": "+31-316-555444",
    "perceel": null,
    "rangorde": null,
    "aandeelPercentage": null,
    "geschiktheidsbewijzen_refs": [
      {"type": "UEA", "url": "https://register.uea.nl/staffoost"}
    ],
    "actief": true,
    "verbruiktVolumePerLeverancierEur": 0,
    "gegundeMinicompetities": 0
  }
]
```

### Sample Minicompetition: MC-2024-INHUUR-001-007

```json
{
  "id": "mc-2024-007-uuid",
  "raamovereenkomst_id": "ro-2024-001-uuid",
  "nummer": "MC-2024-INHUUR-001-007",
  "titel": "Java Developer - Backend Services Migration (8 FTE voor 6 maanden)",
  "omschrijving": "Staffing for backend services team: experienced Java developers (5+ years) to support migration from monolithic to microservices architecture. Preferred: Spring Boot, Kubernetes, CI/CD pipeline experience.",
  "aanvragende_afdeling": "IT Operations",
  "aanvragende_user_id": "user-it-ops-manager",
  "triggerType": "projectopdracht",
  "perceel": null,
  "geraamde_waarde_eur": 720000,
  "gunningsmodel": "emvi_prijs_kwaliteit",
  "prijsplafond_eur": null,
  "wegingsschema": {
    "prijs_percentage": 50,
    "kwaliteit_percentage": 40,
    "duurzaamheid_percentage": 10
  },
  "kwaliteitscriteria": [
    {
      "id": "crit-001",
      "naam": "Relevant experience",
      "omschrijving": "Years of Java development, microservices architecture, and deployment pipeline experience",
      "max_punten": 30,
      "weging_percentage": 20
    },
    {
      "id": "crit-002",
      "naam": "Technical methodology",
      "omschrijving": "Description of how team will approach code review, testing, and documentation standards",
      "max_punten": 25,
      "weging_percentage": 15
    },
    {
      "id": "crit-003",
      "naam": "Communication & escalation",
      "omschrijving": "How team will coordinate with internal stakeholders, weekly reporting, issue resolution",
      "max_punten": 20,
      "weging_percentage": 5
    }
  ],
  "duurzaamheidscriteria": [
    {
      "id": "durz-001",
      "naam": "Knowledge transfer & sustainability",
      "omschrijving": "Plan for handing over code, documentation, and training internal team for long-term maintainability",
      "max_punten": 15,
      "weging_percentage": 10
    }
  ],
  "inschrijvingsTermijnDagen": 21,
  "publicatieDatum": "2024-05-10T10:00:00Z",
  "sluitingsDatum": "2024-05-31T17:00:00Z",
  "uitgenodige_leveranciers_ids": ["rl-001-uuid", "rl-002-uuid", "rl-003-uuid"],
  "selectieReden": null,
  "status": "in_beoordeling",
  "beoordeling_commissie_ids": ["user-inkoper-001", "user-it-ops-manager", "user-hr-advisor", "user-it-director"],
  "winnaar_leverancier_id": "rl-002-uuid",
  "gunnings_bedrag_eur": 650000,
  "gunningsbeslissing_ref": "https://assets.example.com/MC-2024-INHUUR-001-007-award.pdf",
  "contract_ref": "contract-mc-2024-007-uuid",
  "alcatel_deadline_date": "2024-06-20",
  "created_at": "2024-05-08T14:30:00Z",
  "created_by": "user-it-ops-manager"
}
```

### Sample Submissions for MC-2024-INHUUR-001-007

```json
[
  {
    "id": "insch-001-uuid",
    "minicompetitie_id": "mc-2024-007-uuid",
    "leverancier_id": "rl-001-uuid",
    "ingediend_datum": "2024-05-25T09:15:00Z",
    "inschrijvings_bedrag_eur": 720000,
    "prijs_opbouw": [
      {"beschrijving": "8 FTE Java developers @ €9000/maand", "hoeveelheid": 8, "eenheidsprijs": 9000, "subtotaal": 432000},
      {"beschrijving": "Project management & coordination @ €3000/maand", "hoeveelheid": 1, "eenheidsprijs": 3000, "subtotaal": 18000},
      {"beschrijving": "Infrastructure & tooling", "hoeveelheid": 1, "eenheidsprijs": 270000, "subtotaal": 270000}
    ],
    "tijdig": true,
    "geldigVerklaard": true,
    "geldigheidsReden": null,
    "created_at": "2024-05-25T09:15:00Z"
  },
  {
    "id": "insch-002-uuid",
    "minicompetitie_id": "mc-2024-007-uuid",
    "leverancier_id": "rl-002-uuid",
    "ingediend_datum": "2024-05-28T14:45:00Z",
    "inschrijvings_bedrag_eur": 650000,
    "prijs_opbouw": [
      {"beschrijving": "8 FTE Senior Java developers @ €8000/maand", "hoeveelheid": 8, "eenheidsprijs": 8000, "subtotaal": 384000},
      {"beschrijving": "Scrum master & stakeholder engagement", "hoeveelheid": 1, "eenheidsprijs": 6000, "subtotaal": 36000},
      {"beschrijving": "Knowledge transfer sessions & documentation", "hoeveelheid": 1, "eenheidsprijs": 230000, "subtotaal": 230000}
    ],
    "tijdig": true,
    "geldigVerklaard": true,
    "geldigheidsReden": null,
    "created_at": "2024-05-28T14:45:00Z"
  },
  {
    "id": "insch-003-uuid",
    "minicompetitie_id": "mc-2024-007-uuid",
    "leverancier_id": "rl-003-uuid",
    "ingediend_datum": "2024-05-29T16:20:00Z",
    "inschrijvings_bedrag_eur": 695000,
    "prijs_opbouw": [
      {"beschrijving": "8 FTE Java developers @ €8500/maand", "hoeveelheid": 8, "eenheidsprijs": 8500, "subtotaal": 408000},
      {"beschrijving": "Team lead & daily coordination", "hoeveelheid": 1, "eenheidsprijs": 4500, "subtotaal": 27000},
      {"beschrijving": "Tools, equipment & logistics", "hoeveelheid": 1, "eenheidsprijs": 260000, "subtotaal": 260000}
    ],
    "tijdig": true,
    "geldigVerklaard": true,
    "geldigheidsReden": null,
    "created_at": "2024-05-29T16:20:00Z"
  }
]
```

### Sample Scoring Records

```json
[
  {
    "id": "beoor-001-uuid",
    "inschrijving_id": "insch-001-uuid",
    "beoordelaar_user_id": "user-inkoper-001",
    "criterium_id": "crit-001",
    "score": 24,
    "motivering": "TechStaff has 8+ years Java experience, demonstrated microservices work on 3 production projects.",
    "consensus_score": 25,
    "consensus_motivering": "After discussion, commission agreed score should be 25 given the additional detail on K8s certification.",
    "flagged_for_discussion": false
  },
  {
    "id": "beoor-002-uuid",
    "inschrijving_id": "insch-002-uuid",
    "beoordelaar_user_id": "user-inkoper-001",
    "criterium_id": "crit-001",
    "score": 28,
    "motivering": "CodeForce has 10+ years Java, explicit microservices leadership, Kubernetes certified lead developers.",
    "consensus_score": 28,
    "consensus_motivering": "Strong experience profile, clear methodology outlined.",
    "flagged_for_discussion": false
  }
]
```

---

## API Contracts

### POST /raamovereenkomsten

Create a new framework agreement (typically from promotion).

**Request:**
```json
{
  "nummer": "RO-2024-IT-001",
  "titel": "IT Services Framework 2024-2026",
  "omschrijving": "...",
  "modaliteit": "meerdere_minicompetitie",
  "categorie": "diensten",
  "cpvHoofdcode": "72000000",
  "aanbesteder_id": "org-uuid",
  "startDatum": "2024-01-01",
  "eindDatum": "2026-01-01",
  "verlengingsopties": [{"duur_maanden": 12, "eenmalig": false, "max_aantal_keer": 2}],
  "maximumVolumeEur": 2000000,
  "oorspronkelijkeAanbestedingId": "aab-uuid" (optional, if promoted)
}
```

**Response:** 201 Created
```json
{
  "id": "ro-uuid",
  "nummer": "RO-2024-IT-001",
  "status": "concept",
  "links": {
    "self": "/raamovereenkomsten/ro-uuid",
    "leveranciers": "/raamovereenkomsten/ro-uuid/leveranciers"
  }
}
```

### POST /raamovereenkomsten/{id}/minicompetities

Create a new minicompetition within a framework.

**Request:**
```json
{
  "titel": "Java Developers Q3 2024",
  "omschrijving": "...",
  "aanvragende_afdeling": "IT Operations",
  "triggerType": "nieuwe_behoefte",
  "geraamde_waarde_eur": 720000,
  "gunningsmodel": "emvi_prijs_kwaliteit",
  "wegingsschema": {"prijs_percentage": 50, "kwaliteit_percentage": 40, "duurzaamheid_percentage": 10},
  "kwaliteitscriteria": [...],
  "inschrijvingsTermijnDagen": 21,
  "uitgenodige_leveranciers_ids": ["rl-1", "rl-2", "rl-3"],
  "selectieReden": null (required if subset < all active)
}
```

**Response:** 201 Created
```json
{
  "id": "mc-uuid",
  "nummer": "MC-2024-IT-001-042",
  "status": "concept",
  "sluitingsDatum": "2024-06-20T17:00:00Z",
  "links": {
    "self": "/raamovereenkomsten/{ro-id}/minicompetities/mc-uuid",
    "publish": "/raamovereenkomsten/{ro-id}/minicompetities/mc-uuid/publish"
  }
}
```

### POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/publish

Send invitations to suppliers.

**Request:**
```json
{
  "inschrijvingsTermijnDagen": 21
}
```

**Response:** 200 OK
```json
{
  "status": "uitgenodigd",
  "invitations_sent": 3,
  "publicatieDatum": "2024-05-10T10:00:00Z",
  "sluitingsDatum": "2024-05-31T17:00:00Z"
}
```

### POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/submit-bid

Supplier submits a bid (via supplier portal, authenticated).

**Request:**
```json
{
  "inschrijvings_bedrag_eur": 650000,
  "prijs_opbouw": [...],
  "kwaliteitsscores": [
    {"criterium_id": "crit-001", "score": "...", "motivering": "Our team has 10+ years experience..."}
  ],
  "documents": [
    {"type": "offerte", "file": "data:application/pdf;base64,..."}
  ]
}
```

**Response:** 201 Created
```json
{
  "id": "insch-uuid",
  "ingediend_datum": "2024-05-25T09:15:00Z",
  "status": "accepted",
  "message": "Your bid has been received and stored securely. You will be notified of the award decision on or after 2024-06-20."
}
```

### POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/open-bids

Committee opens bids for review (dual-signature gate).

**Request:**
```json
{
  "authorizers": ["user-id-1", "user-id-2"]
}
```

**Response:** 200 OK
```json
{
  "status": "in_beoordeling",
  "bids_unsealed": 3,
  "pv_generated": "https://assets/MC-2024-001-opening-pv.pdf",
  "message": "Bids are now visible to the scoring committee. Process verbal of opening has been generated and signed."
}
```

### POST /minicompetities/{mc-id}/scores

Committee member submits scores for one or more criteria.

**Request:**
```json
{
  "reviewer_id": "user-uuid",
  "scores": [
    {
      "inschrijving_id": "insch-001",
      "criterium_id": "crit-001",
      "score": 24,
      "motivering": "Strong experience profile..."
    }
  ]
}
```

**Response:** 201 Created
```json
{
  "scores_recorded": 3,
  "divergences_detected": [
    {"inschrijving_id": "insch-001", "criterium_id": "crit-001", "flagged": true}
  ]
}
```

### POST /minicompetities/{mc-id}/consensus

Record consensus score after committee discussion.

**Request:**
```json
{
  "inschrijving_id": "insch-001",
  "criterium_id": "crit-001",
  "consensus_score": 25,
  "consensus_motivering": "After discussion, agreed on 25 due to..."
}
```

**Response:** 200 OK

### POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/award

Record the award decision.

**Request:**
```json
{
  "winnaar_leverancier_id": "rl-002",
  "gunnings_bedrag_eur": 650000,
  "motivering": "Highest weighted score of 87.5 points based on price and quality criteria."
}
```

**Response:** 201 Created
```json
{
  "status": "gegund",
  "gunningsbeslissing_pdf": "https://assets/MC-2024-award-decision.pdf",
  "alcatel_deadline": "2024-06-20",
  "message": "Award decision generated, signed, and sent to all bidders. Dispute period runs until 2024-06-20."
}
```

### GET /raamovereenkomsten/{ro-id}/audit-export

Export complete audit trail for 7-year retention (ZIP file).

**Query params:**
- `from_date`: ISO date
- `to_date`: ISO date

**Response:** 200 OK (application/zip)
```
MC-2024-001-042/
  invitations.pdf
  submissions/
    insch-001.pdf
    insch-002.pdf
  scores/
    scores_reviewer1.xlsx
    scores_reviewer2.xlsx
  pvs/
    pv_opening.pdf (signed)
    pv_scoring.pdf (signed)
    pv_award.pdf (signed)
  award_decision.pdf
  audit_log.json
```

---

## Integration Points

- **aanbesteding-werkproces**: Promotion flow creates raamovereenkomst from gegunde aanbesteding
- **tenderned-publicatie-adapter**: Wijzigingsaankondigingen when framework volume is exceeded; publication of award notices for above-threshold competitions
- **mvi-sroi-aanbesteding**: Sustainability criteria defaults copied to minicompetition
- **openconnector**: Email delivery of invitations (SMTP with DKIM/DMARC)
- **docudesk**: PV generation and template management
- **mydash**: KPI widgets for framework portfolio overview
- **opentalk** (optional): Video call recording for consensus meetings, attached to scoring PV
