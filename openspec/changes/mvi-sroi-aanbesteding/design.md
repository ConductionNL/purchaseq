---
status: in-review
created: 2026-05-22
version: 1.0
---

# MVI- en SROI-eisen in Aanbestedingen — Design

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Procurement UI (purchaseq)                │
│  Tender Setup → MVI Selection → Scoring → Contract Mgmt     │
└───────────┬─────────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────────┐
│             MVI/SROI Capability Backend                      │
│  ├─ Criteria Management (import, versioning)                 │
│  ├─ MVI Set Recommendation Engine                            │
│  ├─ SROI Obligation Calculator                              │
│  ├─ SROI Realization Workflow & Approval                     │
│  └─ Portfolio Reporting & MVOI Export                        │
└───────────┬──────────────────────────────────────────────────┘
            │
    ┌───────┴──────────────────────────────────────┐
    │                                              │
┌───▼─────────┐  ┌──────────────┐  ┌──────────────┐
│ openregister│  │openconnector │  │docudesk      │
│ (schemas)   │  │(RVO import)  │  │(templates)   │
└─────────────┘  └──────────────┘  └──────────────┘
    │
┌───▼──────────────────┐      ┌───────────────┐
│ mydash (GraphQL)     │      │n8n / pipelinq │
│ Portfolio Dashboard  │      │(scheduled jobs)│
└──────────────────────┘      └───────────────┘
```

## Entity Schemas

### 1. `mvi_criterium` — MVI Criteria Library

**Purpose**: Central registry of MVI criteria from RVO, sector standards, and org-owned sources. Versioned to allow historical lookup.

**Fields**:

```json
{
  "id": "uuid",
  "code": "string (e.g., 'RVO-15-CO2-1' or 'ORG-CIRC-001')",
  "bron": "enum [rvo_criteriatool, cpv_klimaatakkoord, organisatie_eigen, cirkelstad, rijk_mvoi]",
  "categorie": "enum [klimaat_co2, milieu_overig, social_arbeid, social_inclusie, circulair, biodiversiteit, dierenwelzijn, internationaal_iso26000, mkb_friendly, lokale_economie]",
  "inkoopcategorie": "string[] (e.g., ['ICT-hardware', 'ICT-software', 'catering', 'bouw', ...])",
  "titel": "string (e.g., 'CO2-uitstoot per kilogram product')",
  "omschrijving": "text",
  "niveau": "enum [basis, significant, ambitieus]",
  "eisType": "enum [minimumeis, gunningscriterium, contracteis]",
  "meetbaarheid": "enum [ja_meetbaar, kwalitatief, certificaat_bewijs]",
  "bewijsmiddelen": "enum[] [co2_prestatieladder_niveau, pso_trede, milieukeur, msc_label, fsc_label, cradle_to_cradle, isae_3000, eigen_verklaring, zelfverklaring_met_audit_recht]",
  "geldigVanaf": "date",
  "geldigTot": "date or null (null = still active)",
  "versie": "integer",
  "createdAt": "timestamp",
  "createdBy": "uuid (user ref)",
  "organisatieRef": "uuid or null (if org-specific)"
}
```

**Seed Data (Dutch examples)**:

```json
[
  {
    "code": "RVO-15-CO2-1",
    "bron": "rvo_criteriatool",
    "categorie": "klimaat_co2",
    "inkoopcategorie": ["ICT-hardware"],
    "titel": "CO2-voetafdruk per kilogram",
    "omschrijving": "Leverancier dient CO2-Prestatieladder niveau aantoonbaar te hebben met geldig certificaat.",
    "niveau": "significant",
    "eisType": "gunningscriterium",
    "meetbaarheid": "certificaat_bewijs",
    "bewijsmiddelen": ["co2_prestatieladder_niveau"],
    "geldigVanaf": "2026-01-01",
    "geldigTot": null,
    "versie": 3
  },
  {
    "code": "ORG-CIRC-001",
    "bron": "organisatie_eigen",
    "categorie": "circulair",
    "inkoopcategorie": ["catering"],
    "titel": "Minimaal 50% van gebruikte verpakking recyclebaar of composteerbaar",
    "omschrijving": "Cateraar dient minimaal 50% van gebruikte verpakkingmaterialen recyclebaar of composteerbaar te maken.",
    "niveau": "basis",
    "eisType": "contracteis",
    "meetbaarheid": "ja_meetbaar",
    "bewijsmiddelen": ["eigen_verklaring", "zelfverklaring_met_audit_recht"],
    "geldigVanaf": "2025-04-01",
    "geldigTot": null,
    "versie": 1,
    "organisatieRef": "org-gemeenteRotterdam"
  },
  {
    "code": "RVO-20-SOC-1",
    "bron": "rvo_criteriatool",
    "categorie": "social_inclusie",
    "inkoopcategorie": ["schoonmaak", "catering"],
    "titel": "Inzet arbeidsgehandicapten minimaal 3% van uren",
    "omschrijving": "Leverancier dient minimaal 3% van gefactureerde uren in te zetten via arbeidsgehandicapten-betrokken organisaties.",
    "niveau": "basis",
    "eisType": "contracteis",
    "meetbaarheid": "ja_meetbaar",
    "bewijsmiddelen": ["eigen_verklaring"],
    "geldigVanaf": "2025-01-01",
    "geldigTot": null,
    "versie": 1
  }
]
```

---

### 2. `aanbesteding_mvi_set` — Per-Tender MVI Selection

**Purpose**: The specific MVI criteria chosen for a given tender, with customized weightings and proof requirements.

**Fields**:

```json
{
  "id": "uuid",
  "aanbestedingRef": "uuid (ref to tender)",
  "criteriumRef": "uuid (ref to mvi_criterium)",
  "criteriumCode": "string (denormalized for performance)",
  "eisType": "enum [minimumeis, gunningscriterium, contracteis]",
  "weging": "integer or null (only for gunningscriteria, e.g., 25 punten)",
  "minimumNiveau": "enum [basis, significant, ambitieus]",
  "verplichtBewijsmiddel": "enum or null (override default from criterium)",
  "formuleringInBestek": "text (customized text for tender document)",
  "motivering": "text (why this criterion in this tender)",
  "geintegreerdIn": "enum [gunningskader, programma_van_eisen, conceptcontract]",
  "createdAt": "timestamp",
  "createdBy": "uuid (inkoper)",
  "versie": "integer (increments with tender revision)"
}
```

**Seed Data (Dutch example)**:

```json
[
  {
    "aanbestedingRef": "tender-26-055-IT-Hardware",
    "criteriumRef": "RVO-15-CO2-1",
    "criteriumCode": "RVO-15-CO2-1",
    "eisType": "gunningscriterium",
    "weging": 25,
    "minimumNiveau": "significant",
    "verplichtBewijsmiddel": "co2_prestatieladder_niveau",
    "formuleringInBestek": "Leverancier dient CO2-Prestatieladder niveau 3 of hoger aantoonbaar te hebben via geldig SKAO-certificaat, niet ouder dan 24 maanden.",
    "motivering": "Bijdrage aan org-doel 'CO2-reductie 49% in 2030' + kostenreductie via energiebesparing.",
    "geintegreerdIn": "gunningskader"
  }
]
```

---

### 3. `sroi_verplichting` — SROI Obligation per Tender/Contract

**Purpose**: Stores the calculated SROI obligation for a given tender or contract, including percentage, amount, model, and status.

**Fields**:

```json
{
  "id": "uuid",
  "aanbestedingRef": "uuid (ref to tender) or contractRef for updates",
  "contractRef": "uuid or null (filled after award)",
  "opdrachtwaardeEur": "decimal",
  "sroiPercentage": "decimal (e.g., 2.0, 5.0)",
  "sroiBedragEur": "decimal (computed: opdrachtwaardeEur * sroiPercentage / 100)",
  "peilDatum": "date (date of obligation calculation)",
  "regioPolicy": "string (ref to policy document, e.g., 'Regio Rijnmond SROI 2024')",
  "bouwblokkenModel": "enum [bouwblokken_2018, bouwblokken_2023, eigen_model]",
  "realisatieperiode": {
    "start": "date",
    "eind": "date"
  },
  "tussentijdseRapportagesFrequentie": "enum [maandelijks, per_kwartaal, halfjaarlijks]",
  "eindrapportageDatum": "date",
  "status": "enum [concept, vastgelegd, in_uitvoering, gerealiseerd, niet_gerealiseerd, vrijgesteld]",
  "vrijstellingMotivering": "text or null",
  "createdAt": "timestamp",
  "createdBy": "uuid (inkoper/SROI-coördinator)",
  "beheerdDoor": "uuid (SROI-coördinator)"
}
```

**Seed Data (Dutch example)**:

```json
[
  {
    "aanbestedingRef": "tender-26-055-Schoonmaak",
    "opdrachtwaardeEur": 400000,
    "sroiPercentage": 5.0,
    "sroiBedragEur": 20000,
    "peilDatum": "2026-02-15",
    "regioPolicy": "Regio Rijnmond SROI 2024",
    "bouwblokkenModel": "bouwblokken_2023",
    "realisatieperiode": {
      "start": "2026-04-01",
      "eind": "2027-03-31"
    },
    "tussentijdseRapportagesFrequentie": "per_kwartaal",
    "eindrapportageDatum": "2027-04-15",
    "status": "vastgelegd"
  }
]
```

---

### 4. `sroi_bouwblok` — Reference: Uniform SROI Weighting (Bouwblokkenmethode)

**Purpose**: Immutable reference data from TNO/PSO defining the economic value of SROI interventions (e.g., "WW placement 12+ mnd, 32 hrs/wk = €35k/FTE/year").

**Fields**:

```json
{
  "id": "uuid",
  "code": "string (e.g., 'BB-WW-12', 'BB-Wajong', 'BB-Scholing', 'BB-LocaleInkoop')",
  "categorie": "enum [uitkeringsgerechtigden, werkloosheid, werkhervatting, participatiewet, statushouders, ex_gedetineerden, leerwerkplek_bbl, leerwerkplek_bol, sociale_werkvoorziening, kwetsbare_jongeren, scholing_omscholing, lokale_inkoop, mkb_inschakeling]",
  "subcategorie": "string (optional detail, e.g., 'WW 12+ maanden')",
  "weegfactorEur": "decimal (per uur, per FTE-jaar, or per geplaatste persoon; unit specified in 'eenheid')",
  "eenheid": "enum [per_uur, per_fte_jaar, per_persoon]",
  "voorwaarden": "text (e.g., 'Plaatsing minimaal 32 uur/week, duur minimaal 12 weken')",
  "geldigVanaf": "date",
  "geldigTot": "date or null",
  "bouwblokkenVersie": "enum [2018, 2023] (TNO/PSO versie)"
}
```

**Seed Data (Dutch example)**:

```json
[
  {
    "code": "BB-WW-12",
    "categorie": "werkloosheid",
    "subcategorie": "WW 12+ maanden, 32+ uur/week",
    "weegfactorEur": 25.0,
    "eenheid": "per_uur",
    "voorwaarden": "Werkloze gedurende minimaal 12 maanden; plaatsing minimaal 32 uur per week; duur minimaal 12 weken aaneengesloten.",
    "geldigVanaf": "2023-01-01",
    "geldigTot": null,
    "bouwblokkenVersie": 2023
  },
  {
    "code": "BB-Scholing",
    "categorie": "scholing_omscholing",
    "subcategorie": "Omscholing kwetsbare groep",
    "weegfactorEur": 5000,
    "eenheid": "per_persoon",
    "voorwaarden": "Scholingsprogramma minimaal 6 weken; deelnemer uit doelgroep (NEET, laag-alfabetisme, etc.); afronding met certificaat.",
    "geldigVanaf": "2023-01-01",
    "geldigTot": null,
    "bouwblokkenVersie": 2023
  },
  {
    "code": "BB-LocaleInkoop",
    "categorie": "lokale_inkoop",
    "subcategorie": "Inkoop lokale MKB (<50 FTE) voor goederen/diensten",
    "weegfactorEur": 2500,
    "eenheid": "per_persoon",
    "voorwaarden": "Leverancier is MKB (<50 FTE) in dezelfde regio als aanbestedende organisatie; anti-stapeling: geen doorverkoop mogelijk zonder medeweten aanbestedende organisatie.",
    "geldigVanaf": "2023-01-01",
    "geldigTot": null,
    "bouwblokkenVersie": 2023
  }
]
```

---

### 5. `sroi_realisatie` — Actual SROI Realization per Reporting Period

**Purpose**: Vendor reports progress against SROI obligation; system calculates realized value and tracks approval.

**Fields**:

```json
{
  "id": "uuid",
  "verplichtingRef": "uuid (ref to sroi_verplichting)",
  "periode": "string (e.g., '2026-Q2', '2026-H1')",
  "bouwblokRef": "uuid (ref to sroi_bouwblok)",
  "bouwblokCode": "string (denormalized)",
  "aantalUren": "decimal or null",
  "aantalFtes": "decimal or null",
  "aantalPersonen": "integer or null",
  "gerealiseerdeWaardeEur": "decimal (computed: aantal * weegfactor)",
  "bewijsmiddelen": [
    {
      "type": "enum [loonadministratie_extract, plaatsingsverklaring, stagecontract, scholingscertificaat, andere]",
      "reference": "string (file ref or description)",
      "uploadDatum": "date",
      "geldTot": "date or null"
    }
  ],
  "verantwoordingsbron": "enum [zelf_rapportage_leverancier, audit_door_aanbestedingsdienst, audit_door_pso, accountantsverklaring]",
  "goedgekeurd": "boolean",
  "goedgekeurdDoor": "uuid or null (SROI-coördinator)",
  "goedkeuringsDatum": "date or null",
  "opmerkingen": "text or null",
  "createdAt": "timestamp",
  "createdBy": "uuid (vendor or staff)"
}
```

**Seed Data (Dutch example)**:

```json
[
  {
    "verplichtingRef": "sroi-verplichting-26-055",
    "periode": "2026-Q2",
    "bouwblokRef": "BB-WW-12",
    "bouwblokCode": "BB-WW-12",
    "aantalUren": 960,
    "aantalPersonen": null,
    "gerealiseerdeWaardeEur": 24000,
    "bewijsmiddelen": [
      {
        "type": "loonadministratie_extract",
        "reference": "Loonlijst_Q2_2026_ScopIntensief.pdf",
        "uploadDatum": "2026-05-10",
        "geldTot": null
      },
      {
        "type": "plaatsingsverklaring",
        "reference": "Plaatsingsverklaring_JanPieterse_BB-WW-12.pdf",
        "uploadDatum": "2026-05-10",
        "geldTot": null
      }
    ],
    "verantwoordingsbron": "zelf_rapportage_leverancier",
    "goedgekeurd": true,
    "goedgekeurdDoor": "user-sroi-coo",
    "goedkeuringsDatum": "2026-05-15",
    "opmerkingen": "Bewijsmiddelen validatie OK. Hoogte en doelgroepkwalificatie gechecked."
  }
]
```

---

### 6. `mvi_monitoring_rapportage` — MVI Contract Monitoring per Period

**Purpose**: Track compliance with contract MVI obligations (e.g., CO₂ Ladder renewal, circular reporting) and penalties/bonuses.

**Fields**:

```json
{
  "id": "uuid",
  "contractRef": "uuid",
  "mviSetRef": "uuid (ref to aanbesteding_mvi_set, the contract obligation)",
  "periode": "string (e.g., '2026-Q1', '2026-Jaar')",
  "gerapporteerdeWaarde": "string or decimal (e.g., 'CO2-Ladder Level 4', or '45% circulaire inhoud')",
  "bewijsmiddelRef": "string (file ref, cert number, audit report #)",
  "voldoetAanEis": "boolean",
  "afwijking": "text (if not compliant, description)",
  "verbetervoorstel": "text (remedial action)",
  "boeteOpgelegdEur": "decimal or null",
  "bonusToegekendEur": "decimal or null",
  "createdAt": "timestamp",
  "createdBy": "uuid (contractmanager)",
  "beheerdDoor": "uuid (contractmanager)"
}
```

**Seed Data (Dutch example)**:

```json
[
  {
    "contractRef": "contract-26-055-IT-Hardware",
    "mviSetRef": "mvi-set-26-055-CO2",
    "periode": "2026-Q1",
    "gerapporteerdeWaarde": "CO2-Ladder Level 4",
    "bewijsmiddelRef": "SKAO-cert-123456-exp-2027-03-31",
    "voldoetAanEis": true,
    "afwijking": null,
    "verbetervoorstel": null,
    "boeteOpgelegdEur": null,
    "bonusToegekendEur": null
  }
]
```

---

### 7. `mvi_portfolio_doelstelling` — Organizational Portfolio Goals

**Purpose**: Store org-wide goals (CO₂ -49% by 2030, Circular 50% by 2030, etc.) with historical tracking and governance linkage.

**Fields**:

```json
{
  "id": "uuid",
  "organisatieRef": "uuid",
  "thema": "enum [co2_reductie_percentage, circulair_percentage_uitgaven, sroi_realisatie_percentage, mkb_aandeel_percentage, biodiversiteit_index]",
  "doeljaar": "integer (e.g., 2030)",
  "doelwaarde": "decimal (e.g., 49 for 49%)",
  "baselineJaar": "integer (e.g., 1990 for CO₂, 2023 for circulair)",
  "baselineWaarde": "decimal (e.g., 100 for CO₂ index, 5 for % circulair)",
  "huidigeWaarde": "decimal (computed, refreshed quarterly)",
  "voortgang": "decimal (computed: % of trajectory toward doelwaarde from baseline)",
  "bestuurlijkAkkoord": {
    "besluitType": "enum [collegebesluit, raadsbesluit, gedeputeerdenbesluit]",
    "besluitDatum": "date",
    "besluitKenmerk": "string",
    "motivering": "text"
  },
  "createdAt": "timestamp",
  "createdBy": "uuid",
  "versie": "integer"
}
```

**Seed Data (Dutch example)**:

```json
[
  {
    "organisatieRef": "org-gemeenteRotterdam",
    "thema": "co2_reductie_percentage",
    "doeljaar": 2030,
    "doelwaarde": 49,
    "baselineJaar": 1990,
    "baselineWaarde": 100,
    "huidigeWaarde": 68,
    "voortgang": 32,
    "bestuurlijkAkkoord": {
      "besluitType": "raadsbesluit",
      "besluitDatum": "2023-05-20",
      "besluitKenmerk": "RB 2023/1234",
      "motivering": "Uitvoering Klimaatakkoord NL 2019 en Convenant Klimaatakkoord Inkoop."
    }
  },
  {
    "organisatieRef": "org-gemeenteRotterdam",
    "thema": "circulair_percentage_uitgaven",
    "doeljaar": 2030,
    "doelwaarde": 50,
    "baselineJaar": 2023,
    "baselineWaarde": 5,
    "huidigeWaarde": 18,
    "voortgang": 26,
    "bestuurlijkAkkoord": {
      "besluitType": "collegebesluit",
      "besluitDatum": "2024-02-15",
      "besluitKenmerk": "CB 2024/567",
      "motivering": "Alignment met Nationaal Programma Circulaire Economie 2023–2030."
    }
  }
]
```

---

## API Endpoints

### MVI Criteria Management

```
GET  /api/mvi-criteria
     query params: categorie, bron, niveau, inkoopcategorie, organisatieRef
     → List of mvi_criterium records, sorted by relevance

POST /api/mvi-criteria
     body: { code, bron, categorie, ... }
     → Create new mvi_criterium (org-owned)

GET  /api/mvi-criteria/{id}
     → Fetch single criterium with version history

PUT  /api/mvi-criteria/{id}
     → Update criterium (marks old version with geldigTot)

GET  /api/mvi-criteria/{id}/history
     → All versions of a criterium (for audit)
```

### Tender MVI Set

```
POST /api/tenders/{tenderId}/mvi-set/recommend
     query params: ambitieniveau (basis|significant|ambitieus)
     → Auto-generate MVI set for tender based on category & org goals

GET  /api/tenders/{tenderId}/mvi-set
     → List of aanbesteding_mvi_set records

PUT  /api/tenders/{tenderId}/mvi-set/{mviSetId}
     → Update weighting, proof requirement, formulation

DELETE /api/tenders/{tenderId}/mvi-set/{mviSetId}
     → Remove criterium from tender (logged)
```

### SROI Obligation & Realization

```
POST /api/tenders/{tenderId}/sroi-obligation
     body: { opdrachtwaardeEur, regioPolicy, ... }
     → Calculate & create sroi_verplichting

GET  /api/sroi-obligations/{obligationId}
     → Fetch obligation details

PUT  /api/sroi-obligations/{obligationId}/exempt
     body: { motivering }
     → Request exemption (state → concept, awaiting approval)

POST /api/sroi-obligations/{obligationId}/realization
     body: { periode, bouwblokRef, aantal*, bewijsmiddelen[] }
     → Vendor submits SROI realization report

GET  /api/sroi-obligations/{obligationId}/realizations
     query params: periode
     → List all realization reports for obligation

PUT  /api/sroi-realizations/{realizationId}/approve
     body: { opmerkingen }
     → SROI Coordinator approves realization

PUT  /api/sroi-realizations/{realizationId}/reject
     body: { opmerkingen }
     → SROI Coordinator rejects & requests revision
```

### Portfolio Reporting

```
GET  /api/portfolio/dashboard
     → Aggregated KPIs: % MVI-compliant, SROI realized, portfolio goal progress

GET  /api/portfolio/goals
     → List all mvi_portfolio_doelstelling records

PUT  /api/portfolio/goals/{goalId}
     body: { doelwaarde, bestuurlijkAkkoord, ... }
     → Update goal (versioning + audit trail)

GET  /api/portfolio/mvoi-export
     query params: jaar
     → CSV+JSON in PIANOo/RIVM format

POST /api/portfolio/college-report
     query params: periode (Q or Jaar)
     → Generate PDF report for governance
```

---

## Data Flow Diagrams

### 1. Tender Setup with MVI Recommendation

```
Inkoper creates tender (CPV code) 
    ↓
System detects: ICT-hardware, Rotterdam region, org goal "Circulair 50%"
    ↓
Recommendation engine queries mvi_criteria:
  - RVO criteria for ICT-hardware (niveau ∈ [basis, significant, ambitieus])
  - Criteria that map to org goals (boost circulair weight)
    ↓
System returns: 
  [CO2-Ladder (gunn, 25pt), Energy-Star (min), Repair-right (contract), Cradle-to-Cradle (gunn, 15pt)]
    ↓
Inkoper reviews & customizes:
  - Remove Repair-right (not applicable)
  - Boost Cradle-to-Cradle weight to 20pt
    ↓
System saves aanbesteding_mvi_set with versioning & audit trail
```

### 2. SROI Obligation Calculation

```
Tender value entered: €800,000
Tender category: ICT-Software (cloud-only)
    ↓
System looks up regio-policy: "Rotterdam SROI 2024"
    ↓
Policy rule: Diensten ≥ €250k → 5% SROI required, EXCEPT cloud-only exempt
    ↓
Inkoper chooses: "Request exemption (cloud-only, no NL labor)"
    ↓
System creates sroi_verplichting with status=concept, freedom_motivering set
    ↓
SROI-coördinator reviews exemption request
    ↓
If approved → sroiBedragEur = 0, status=vrijgesteld
If rejected → status=vastgelegd, sroiBedragEur=€40,000
```

### 3. Vendor SROI Reporting & Approval

```
Q2 deadline approaches (7 days before)
    ↓
System sends email + portal reminder to vendor + CC contractmanager
    ↓
Vendor logs in, selects sroi_obligation, creates sroi_realisatie:
  - Periode: 2026-Q2
  - Bouwblok: BB-WW-12 (WW 12+ mnd plaatsing)
  - Aantal uren: 960 (2 persons, 15 weeks, 32 hrs/week)
  - Gerealiseerde waarde: 960 * €25/hr = €24,000
    ↓
Vendor uploads: loonadministratie_extract.pdf, plaatsingsverklaring.pdf
    ↓
SROI-coördinator reviews:
  - Validates proof docs (expiry, signatures, doelgroep check)
  - Runs anti-stacking check for lokale_inkoop
    ↓
If OK → goedgekeurd=true, system updates cumulatieve realisatie
If ≥15% lag vs. plan → escalation ticket created
```

---

## Deployment Notes

### Database Migrations

- **mvi_criterium**: Index on (bron, categorie, niveau, geldigTot desc) for efficient search.
- **sroi_verplichting**: Index on (contractRef, status) for filtering active obligations.
- **sroi_realisatie**: Index on (verplichtingRef, periode) for period-based reporting.
- **mvi_portfolio_doelstelling**: Index on (organisatieRef, thema) for dashboard queries.

### Scheduled Jobs (n8n/pipelinq)

1. **RVO Import Job** (runs weekly Mon 03:00 UTC):
   - Fetch updated criteria from mvicriteria.nl API
   - Diff against local db; mark old versions with geldigTot
   - Log changes; alert if import fails

2. **SROI Deadline Reminders** (runs daily 09:00 local):
   - Find all sroi_verplichting with next deadline ≤ 7 days
   - Filter by tussentijdseRapportagesFrequentie
   - Send email + portal notification to vendor + CC contractmanager

3. **Escalation Check** (runs daily after 18:00 local):
   - For each active sroi_verplichting in_uitvoering
   - Calculate cumulative realisatie vs. expected (pro-rata of elapsed time)
   - If lag > 15%, create escalation ticket + notify SROI-coördinator

4. **Annual MVOI Export** (runs 2026-12-20 13:00 UTC):
   - Aggregate all contracts: totaal_uitgaven, mvi_conformiteit_pct, sroi_realized, co2_baseline/realization, circulair_index
   - Format as CSV+JSON per PIANOo/RIVM spec
   - Push to RIVM API (if configured)

5. **Portfolio Dashboard Refresh** (runs every 4 hours):
   - Compute huidigeWaarde for each mvi_portfolio_doelstelling
   - Refresh mydash widget (GraphQL mutation)

### Audit Trail

All mutations logged to `openregister` audit table with:
- Timestamp
- User (uuid + email)
- Entity & ID
- Operation (create, update, delete)
- Old & new values (JSON diff)
- Reason code (e.g., "exemption_approved", "tender_version_bump")

---

## Integration Points

| System | Direction | Trigger | Payload |
|--------|-----------|---------|---------|
| **purchaseq** (tender UI) | ← | Tender published | aanbesteding_mvi_set + SROI obligation |
| **tenderned-adapter** | ← | Tender published | eForms BT-755 (Accessibility), BT-805/806 (Green), BT-756 (SROI) |
| **docudesk** | ← | Tender published | MVI bestek-fragment from library + gunning scoring instructions |
| **openconnector** | → | Weekly | RVO import job result (success/fail + delta) |
| **mydash** | ← | Every 4h | Portfolio dashboard data (GraphQL mutation) |
| **n8n / pipelinq** | ← | Per schedule | Reminders, escalations, exports |

---

## Security & Privacy Considerations

- **Audit trail immutable**: All decisions logged; no deletion, only soft-deletion + reason.
- **Proof document encryption**: Bewijsmiddelen uploaded to secure store; access logged.
- **Role-based access**:
  - Inkoper: view/edit tender MVI setup, view portfolio dashboard
  - SROI-coördinator: approve/reject realizations, manage exemptions, escalate
  - Contractmanager: view contract MVI status, report compliance
  - MVI-adviseur: curate criteria library, manage portfolio goals
  - Bestuurder: read-only dashboard + reports
- **Data minimization**: Only contract-active vendor contacts exposed; vendor portal is separate authentication realm.

