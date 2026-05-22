---
status: draft
created: 2026-05-22
version: 1.0
---

# MVI- en SROI-eisen in Aanbestedingen — Specifications

## Functional Requirements

---

## REQ-MVI-001: MVI Criteria Library Import & Versioning

**Objective**: The system MUST import RVO MVI-criteriatool quarterly (minimum), MUST allow organization-specific criteria without polluting RVO criteria, and MUST maintain version tracking so active tenders retain their original criteria version.

### Acceptance Criteria

#### AC-MVI-001-01: Quarterly RVO Import with Versioning
- **Given** the RVO publishes a new version of the CO₂ criterion for ICT-hardware (from 250g CO₂/kWh baseline → 200g baseline)
- **When** the import job runs
- **Then**:
  - The old criterium record is marked with `geldigTot = import_timestamp`
  - A new `mvi_criterium` record is created with the updated baseline and `versie` incremented
  - All active tenders referencing the old criterion continue to use `versie` N (unchanged)
  - Future new tenders are offered `versie` N+1 by default
  - Import log records: timestamp, delta count, API endpoint version, status (success/failure)

#### AC-MVI-001-02: Organization-Specific Criteria Registration
- **Given** a procurement officer wants to register an organization-specific criterion "Glasvezel-leverancier with green energy guarantee"
- **When** the officer submits it via the criteria admin UI
- **Then**:
  - System auto-assigns `bron = organisatie_eigen`, `code = ORG-{ORG_ID}-{SERIAL}` (e.g., ORG-RTD-001)
  - Record is saved with `organisatieRef` set to the current organization
  - Criterion appears alongside RVO criteria in the same category dropdown
  - Versioning starts at `versie = 1`
  - Audit log records: who created, timestamp, fields

#### AC-MVI-001-03: Import Failure Fallback
- **Given** the import job fails (RVO API unreachable after 3 retries)
- **When** retry limit is exhausted
- **Then**:
  - The last successful import version remains active
  - Ops alert is raised (Slack/email to MVI-team)
  - Log records: error details, last-good timestamp, retry count
  - All active tenders continue with their already-locked versions (no interruption)
  - Manual re-run can be triggered by admin without re-importing successful-already tenders

#### AC-MVI-001-04: Duplicate Prevention
- **Given** the import job runs and the RVO publishes a criterion with code "RVO-15-CO2-1" (version 3)
- **When** a local "RVO-15-CO2-1" version 2 exists with `geldigTot = null`
- **Then**:
  - Diff algorithm detects the new version and marks v2 with `geldigTot = now()`
  - New v3 record is created
  - If content is identical (hash match), no new record is created; just update `geldigTot` on old record if needed

---

## REQ-MVI-002: Auto-Generate Standard MVI Set per Tender

**Objective**: For each new tender, the system MUST suggest a standard MVI set based on (a) procurement category, (b) org portfolio goals, and (c) chosen ambition level (basis, significant, ambitieus).

### Acceptance Criteria

#### AC-MVI-002-01: Category-Based Recommendation
- **Given** a new tender for ICT-Hardware (CPV 30200000), ambition level "significant"
- **When** the officer clicks "Suggest MVI Set"
- **Then** system returns:
  - CO₂-Ladder nivel Significant (gunning criterion, weight ∈ [15, 25] pts, suggested 20)
  - Energy Star (minimum requirement, binary pass/fail)
  - 5-year spare parts guarantee (contract obligation)
  - Repair right clause (contract obligation)
  - Transport CO₂ footprint (gunning, weight ∈ [5, 10] pts, suggested 8)
  - All text taken from RVO public recommendations for this category
  - All records created as `aanbesteding_mvi_set` with `geintegreerdIn = null` (waiting for officer assignment)

#### AC-MVI-002-02: Ambition Level Scaling
- **Given** the same tender, but officer selects ambition "ambitieus"
- **When** recommendation is generated
- **Then** system returns all "significant" criteria PLUS:
  - Cradle-to-Cradle certification (gunning, weight ∈ [10, 20] pts, suggested 15)
  - Minimum 30% refurbished content (minimum requirement, measurable %)
  - Take-back guarantee (contract obligation, text specifies 5+ year warranty on components)
  - Life-cycle cost analysis (contract obligation, annual report)

#### AC-MVI-002-03: Portfolio Goal Alignment
- **Given** the organization has an active `mvi_portfolio_doelstelling` with `thema = circulair_percentage_uitgaven`, `doeljaar = 2030`, `doelwaarde = 50`, `huidigeWaarde = 18`
- **When** recommendation is generated for any tender
- **Then** system boosts weight on all "circulair" category criteria by +10% (e.g., a 15pt criterion → 16.5pt suggested) and adds motivering text "Contributes to Circular 50% goal (currently 18%, target 50% by 2030)"

#### AC-MVI-002-04: Recommendation Lockable
- **Given** officer reviews the suggested set and clicks "Accept as-is"
- **When** acceptance is confirmed
- **Then**:
  - All recommended `aanbesteding_mvi_set` records are saved with `geintegreerdIn = gunningskader` or appropriate value
  - Tender moves to next workflow step (bestek drafting, etc.)
  - If officer wants to customize, they can add/remove/adjust weightings before accepting

---

## REQ-MVI-003: MVI Scoring in EMVI Gunning Framework

**Objective**: MVI gunning criteria are formulated for EMVI (multi-criteria) scoring, the system provides scoring matrix guidance, and awarded scores are locked into the contract for monitoring.

### Acceptance Criteria

#### AC-MVI-003-01: Score Matrix Generation
- **Given** a tender with MVI gunning criterion "CO₂-Prestatieladder niveau"
- **When** the officer opens the scoring matrix builder
- **Then** system suggests:
  - Level 5 = 100 points (or weighted proportionally if max weight is <100)
  - Level 4 = 80 points
  - Level 3 = 60 points
  - Level 2 = 40 points
  - Level 1 = 20 points
  - No ladder / no proof = 0 points
  - Officer can override each threshold (logged as "manual override")

#### AC-MVI-003-02: Score Validation Against Proof
- **Given** three bidders submit proofs:
  - Bidder A: SKAO CO₂-Ladder Level 4 certificate (expires 2027-06-30)
  - Bidder B: SKAO Level 3 (expires 2026-08-15)
  - Bidder C: No proof submitted
- **When** the scoring commission enters scores via the portal
- **Then** system:
  - Validates certificate expiry (must not be expired or expiring <180 days)
  - Auto-fills scores: A→80, B→60, C→0 (unless manually overridden)
  - Shows confidence indicators (green = auto-filled from valid proof, orange = overridden, red = no proof)
  - Logs any manual override with motivering (mandatory text field)

#### AC-MVI-003-03: Proof Document Verification
- **Given** Bidder A uploads a certificate claiming SKAO Level 4
- **When** the system receives the document
- **Then**:
  - System extracts: issuer, level, expiry date via OCR or manual parsing
  - Validates certificate format (SKAO format v2+)
  - Checks expiry: if expired, mark as "invalid" (red); if expiring <180d, mark as "at-risk" (orange)
  - Score matrix honors this status: valid = score applied, invalid/at-risk = lower tier or 0 (configurable)

#### AC-MVI-003-04: Score Lock-in to Contract
- **Given** the award decision is finalized with Bidder A at 80 points for CO₂-Ladder
- **When** the contract is created
- **Then**:
  - The MVI gunning criterion becomes a contract obligation: "Maintain CO₂-Prestatieladder Level 4 throughout contract term"
  - `mvi_monitoring_rapportage` record is pre-created with `periode = contract_year_1`, `gerapporteerdeWaarde = null`, `voldoetAanEis = null`
  - System schedules annual renewal reminder (first reminder ≥60 days before contract anniversary or level-expiry, whichever is earlier)
  - Scoring worksheet is archived with tender files (immutable record)

---

## REQ-MVI-004: Auto-Calculate SROI Obligation per Tender

**Objective**: The system MUST determine SROI obligation per tender based on (a) value, (b) category, (c) regio-policy, and (d) exemption grounds, and MUST display EUR amount and filling options.

### Acceptance Criteria

#### AC-MVI-004-01: Standard SROI Determination
- **Given** a tender for cleaning services with estimated value €400,000 in Rotterdam region
- **When** the officer enters tender details and clicks "Calculate SROI"
- **Then** system:
  - Looks up "Regio Rijnmond SROI 2024" policy
  - Policy rule: Diensten ≥ €250k → 5% SROI required
  - Creates `sroi_verplichting` with:
    - `sroiPercentage = 5.0`
    - `sroiBedragEur = €400,000 * 0.05 = €20,000`
    - `bouwblokkenModel = bouwblokken_2023` (from policy)
    - `realisatieperiode.start = contract_start_date`
    - `realisatieperiode.eind = contract_start_date + 12 months`
    - `tussentijdseRapportagesFrequentie = per_kwartaal` (default, from policy)
    - `status = concept`
  - Officer reviews and clicks "Confirm" → status becomes "vastgelegd"

#### AC-MVI-004-02: Exemption Request (Cloud Software Example)
- **Given** a tender for specialized ICT cloud software with value €800,000
- **When** initial obligation calculation shows 5% SROI = €40,000, but officer recognizes "no NL physical labor / cloud-only"
- **When** officer clicks "Request Exemption"
- **Then**:
  - System creates exemption request record (linked to sroi_verplichting, status = concept)
  - Officer provides motivering: "Cloud-only SaaS, no NL headcount impact"
  - Request is routed to SROI-coördinator for approval (workflow step)
  - If approved: `status = vrijgesteld`, `sroiBedragEur = 0`
  - If rejected: `status = vastgelegd`, obligation remains at €40,000

#### AC-MVI-004-03: Below-Threshold Suggestion
- **Given** a tender for office supplies with estimated value €100,000 in Rotterdam region
- **When** SROI calculation is triggered
- **Then** system:
  - Detects: value <€250k threshold for diensten
  - Creates `sroi_verplichting` with `status = exempt` (auto-exempt, no obligation)
  - BUT shows UI suggestion: "Voluntary SROI available? Default 2%?" with clickable button
  - If officer clicks "Apply voluntary SROI": obligation recalculated at 2% = €2,000

#### AC-MVI-004-04: Category-Specific Rules
- **Given** a tender for construction/public works with value €1,500,000
- **When** SROI determination runs
- **Then** system:
  - Looks up werken (not diensten) policy rule: "€500k+ → 5% SROI"
  - OR identifies category as "bouw" and applies bouw-specific percentage (may differ, e.g., 3% if labor-intensive bouwwerk)
  - Sets obligation accordingly (log which rule applied for audit)

#### AC-MVI-004-05: Obligation Persistence Across Tender Versions
- **Given** a tender with SROI obligation €20,000 is published
- **When** officer revises tender (changes scope, extends budget to €500,000)
- **Then**:
  - Officer can recalculate SROI (triggers AC-MVI-004-01 logic anew)
  - System shows: "Old obligation €20k (locked into 15 bids). New calculation suggests €25k. Confirm to update?"
  - If confirmed, new `sroi_verplichting` is created (linked to tender v2), old one marked with `geldigTot`
  - All existing bids remain under old obligation; future bids under new

---

## REQ-MVI-005: SROI Realization Reporting & Approval Workflow

**Objective**: Vendors report SROI via bouwblokkenmethode, system calculates realized value, collects proof, and SROI-coördinator approves.

### Acceptance Criteria

#### AC-MVI-005-01: Vendor Realization Report Submission
- **Given** SROI obligation: €20,000, period 2026 Q2, bouwblokkenmodel 2023
- **Given** vendor places 2 WW-eligible workers (12+ mnd unemployed) for 13 weeks, 32 hrs/week each
- **When** vendor logs into vendor portal and creates realization report
- **Then** system:
  - Offers bouwblok selector filtered to applicable blocks (WW-kategorie)
  - Vendor selects "BB-WW-12" (€25/hr weighting)
  - System auto-calculates: 2 persons × 32 hrs/week × 13 weeks = 832 hours; realized value = 832 × €25 = €20,800
  - Vendor uploads: loonadministratie_extract.pdf, two plaatsingsverklaringen (one per person)
  - Vendor sets `verantwoordingsbron = zelf_rapportage_leverancier`
  - System creates `sroi_realisatie` with `goedgekeurd = false` (pending)
  - Email confirmation sent to vendor + CC contractmanager

#### AC-MVI-005-02: Proof Validation & Anti-Stacking
- **Given** the realization report from AC-MVI-005-01 is submitted with proof documents
- **When** SROI-coördinator reviews in the approval UI
- **Then** system:
  - Extracts candidate names + employment dates from payroll and placement docs
  - Cross-checks: WW duration ≥12 mnd? Placement ≥32 hrs/wk? ≥13 weeks continuous?
  - Validates document signatures (mandatory for plaatsingsverklaring per Dutch law)
  - For "lokale_inkoop" bouwblok: checks anti-stacking rule (no subcontracting without org knowledge)
  - Shows green checkmarks for valid docs, orange for partial (e.g., 11.5 mnd WW), red for invalid (e.g., no signature)
  - Displays final validated count (e.g., "2 persons confirmed, 1 awaiting payroll confirmation")

#### AC-MVI-005-03: Coordinator Approval & Cumulatief Tracking
- **Given** the validated realization report shows €20,800 with 2 confirmed persons
- **When** SROI-coördinator clicks "Approve" (with optional opmerkingen: "All docs valid, anti-stacking check OK")
- **Then** system:
  - Sets `goedgekeurd = true`, `goedgekeurdDoor = coordinator_id`, `goedkeuringsDatum = now()`
  - Updates contract's cumulative SROI: e.g., Q1: €0, Q2: €20,800 → Total to date: €20,800
  - Compares to plan: obligation €20k, realized €20,800 → 104% (green indicator)
  - Sends confirmation email to vendor + contractmanager
  - Logs approval action to audit trail

#### AC-MVI-005-04: Rejection & Revision Request
- **Given** the coordinator finds invalid proof (e.g., payroll extract is from Q1, not Q2)
- **When** coordinator clicks "Request Revision" with motivering text
- **Then** system:
  - Sets `goedgekeurd = false`
  - Vendor receives email: "Your Q2 SROI report (€20.8k) was rejected. Reason: Payroll extract dated Q1, not Q2. Please resubmit with correct documents by [deadline]."
  - Realization record stays in `realisatie` queue (not finalized)
  - If vendor resubmits with corrected docs within deadline, system allows re-approval

---

## REQ-MVI-006: SROI Deadline Monitoring & Escalation

**Objective**: System enforces reporting schedules, sends reminders, and escalates if realization lags >15% vs. plan.

### Acceptance Criteria

#### AC-MVI-006-01: Deadline Reminder Automation
- **Given** SROI obligation with `tussentijdseRapportagesFrequentie = per_kwartaal`, next Q2 deadline 2026-05-31
- **When** today = 2026-05-24 (7 days before)
- **When** the scheduled reminder job runs (daily 09:00 local)
- **Then** system:
  - Finds all obligations due within 7 days
  - Sends email to vendor with subject: "Q2 2026 SROI Report due 2026-05-31 (7 days)"
  - Includes portal link, deadline, last-reported amount (if any)
  - CC's contractmanager on reminder
  - Logs reminder sent to audit trail
  - If vendor still hasn't reported 3 days before deadline (2026-05-28), sends second email with escalation warning

#### AC-MVI-006-02: Lag Detection & Risk Flagging
- **Given** annual SROI obligation €20,000, reporting per Q (4 periods), linear plan = €5k/Q
- **Given** current date = 2026-05-31 (end of Q2), vendor has reported:
  - Q1: €3,000 (60% of plan)
  - Q2: no report
- **When** scheduled escalation job runs (daily 18:00 local)
- **Then** system:
  - Calculates: expected to date = €10,000 (Q1+Q2 50% plan)
  - Actual: €3,000 (30% of plan)
  - Lag = 30% actual vs. 50% expected = 15% shortfall (at threshold)
  - Marks obligation as `risicovol = true`
  - Creates escalation ticket: "Q2 SROI lag: 30% of plan (expected 50%). Recommend phone call to vendor by [tomorrow]."
  - Notifies SROI-coördinator via email + portal notification
  - Log entry: timestamp, lag %, risk level (green/orange/red)

#### AC-MVI-006-03: Escalation Severity Bands
- **Given** escalation rules based on lag severity
- **When** lag is detected at different thresholds
- **Then** system triggers:
  - >15% lag: orange flag, email reminder to vendor, CC contractmanager
  - >25% lag: red flag, create internal escalation ticket, notify SROI-coord + contractmanager
  - >40% lag: red + high-priority, schedule forced review call, loop in category manager
  - Log all escalations with timestamp, reason, assigned-to (person), expected resolution date

#### AC-MVI-006-04: End-of-Period Shortfall & Penalty Calculation
- **Given** SROI obligation €20,000, realization deadline 2027-03-31, obligation status in_uitvoering
- **Given** final reported realization = €16,000 (80% of obligation)
- **When** end-of-period arrives and last realization report is finalized
- **Then** system:
  - Calculates shortfall: €20,000 - €16,000 = €4,000 (20% unmet)
  - Applies penalty per contract terms (e.g., "25% surcharge on shortfall"): €4,000 × 1.25 = €5,000
  - Creates `mvi_monitoring_rapportage` record with `boeteOpgelegdEur = €5,000` (if penalty clause is in contract)
  - Generates invoice draft for vendor collection (links to contract financials)
  - Sends notice to vendor + contractmanager: "SROI obligation 80% met. Penalty: €5,000. Invoice #[XYZ] issued."
  - Updates status to `gerealiseerd` or `niet_gerealiseerd` (depending on pass/fail threshold, typically ≥95%)
  - Logs completion action to audit trail

---

## REQ-MVI-007: Post-Award MVI Contract Monitoring

**Objective**: System monitors contract MVI obligations (e.g., CO₂ Ladder maintenance, circular reporting), enforces annual renewal, and triggers penalty/bonus calculations.

### Acceptance Criteria

#### AC-MVI-007-01: Annual MVI Renewal Reminder
- **Given** contract with MVI obligation: "CO₂-Prestatieladder Level 4 to be maintained, yearly renewal required"
- **Given** contract anniversary = 2026-09-15
- **When** today = 2026-07-15 (60 days before anniversary)
- **When** the scheduled reminder job runs
- **Then** system:
  - Sends email to vendor: "Your CO₂-Ladder certificate expires in 60 days (2026-09-15). Please renew and upload new certificate by [2026-09-01]."
  - Creates portal task for vendor
  - CC's contractmanager
  - Sets `mvi_monitoring_rapportage` to `status = pending_renewal`

#### AC-MVI-007-02: Certificate Expiry Validation
- **Given** vendor uploads a CO₂-Ladder certificate with expiry 2025-08-31 (already expired by 8 months)
- **When** system receives the document (auto-parse via OCR or manual entry)
- **Then** system:
  - Detects expiry < today
  - Marks as `voldoetAanEis = false`
  - Sets `afwijking = "Certificate expired 2025-08-31"`
  - Creates contract-monitoring alert: "CO₂-Ladder OVERDUE. Immediate action required."
  - Triggers boete-berekening workflow (linked to contract penalty clause)

#### AC-MVI-007-03: Over-Performance & Bonus Activation
- **Given** contract requires "CO₂-Prestatieladder Level 4"
- **Given** vendor reports renewal with Level 5 certificate
- **When** contractmanager receives and validates the Level 5 cert
- **Then** system:
  - Sets `voldoetAanEis = true` (exceeds requirement)
  - Checks contract for bonus clause (if present, e.g., "Level 5 = €5k bonus")
  - If bonus clause exists: creates `bonusToegekendEur = 5000`
  - Logs over-performance to portfolio reporting (for executive summary: "X contracts exceeded MVI targets this year")
  - Sends positive feedback email to vendor (with bonus detail if applicable)

#### AC-MVI-007-04: Multiple-Year Obligation Tracking
- **Given** multi-year contract with annual MVI monitoring
- **When** contractmanager logs compliance for Year 1, Year 2, Year 3
- **Then** system:
  - Creates separate `mvi_monitoring_rapportage` records per periode (e.g., "2026-Jaar", "2027-Jaar", "2028-Jaar")
  - Accumulates penalties/bonuses per year
  - Portfolio dashboard shows YoY compliance trends (e.g., "Compliance improved Year 1→Year 2: 80%→95%")

---

## REQ-MVI-008: Portfolio Dashboard for MVI/SROI Performance

**Objective**: Real-time portfolio view showing % MVI-compliant spend, cumulative SROI, and progress toward portfolio goals with drill-down capability.

### Acceptance Criteria

#### AC-MVI-008-01: Portfolio KPI Aggregation
- **Given** organization has:
  - Portfolio goal: "Circular 50% by 2030, baseline 5% in 2023"
  - Active contracts: 50 (€20M total spend)
  - 35 contracts are MVI-compliant (defined as ≥1 MVI gunning criterion + passing scores)
  - Cumulative circular-spend 2026 to date: €4.2M (21% of €20M)
- **When** dashboard loads for Q2 2026
- **Then** system displays:
  - "Portfolio MVI Compliance: 70% (35/50 contracts)" — green indicator (target ≥80%)
  - "Circular Spend: 21% (€4.2M / €20M) — orange indicator (target 50% by 2030, on track for 30%)"
  - "Progress to 2030 goal: 26% of trajectory (21% current vs. 5% baseline, 50% goal → 26 of 100 steps)" — orange
  - "SROI Realized YTD: €15.2M (95% of plan €16M)" — green

#### AC-MVI-008-02: Goal Tracking Visualization
- **Given** organization with CO₂ goal "49% reduction by 2030, baseline 1990"
- **Given** 2026 realization: 32% reduction (68% of 1990 emissions)
- **When** user clicks on the CO₂ tile in the dashboard
- **Then** system displays:
  - Line chart: baseline 100 (1990) → 68 (2026 actual) → target 51 (2030 goal)
  - Shows linear path (2026-2030: 4 years, ~4.25% reduction/year needed)
  - Actual 2026: 32% reduced, ~7.4% reduced vs. prior year (exceeds linear pace)
  - Traffic light: green (on track)
  - Text: "At current pace (32% by 2026), target 49% by 2030 is achievable."

#### AC-MVI-008-03: Drill-Down to Category Details
- **Given** user on portfolio dashboard sees "Circular Spend: 21%"
- **When** user clicks the tile
- **Then** system transitions to category-level view:
  - List of categories: IT (42% circular), Catering (8% circular), Cleaning (5% circular), etc.
  - Filter options: show only MVI-compliant, show by status (active/ended), show by risk level
  - Each row clickable to contract list

#### AC-MVI-008-04: Contract-Level Drill-Down
- **Given** user has drilled to Catering category (8% circular, 5 contracts)
- **When** user clicks on the category row
- **Then** system displays list of 5 catering contracts:
  - Contract name, start date, end date, spend, MVI status (pass/fail), SROI obligation, SROI realized, penalty/bonus YTD
  - Each contract clickable to full detail view (scores, monitored criteria, etc.)

#### AC-MVI-008-05: Dashboard Refresh Frequency
- **Given** dashboard is active in browser
- **When** 4+ hours pass since last load
- **Then** system auto-refreshes data (silent, no interruption) from openregister GraphQL queries
- **Timestamp** in dashboard footer shows "Last refresh: 2026-05-22 14:30 UTC"

---

## REQ-MVI-009: Annual MVOI Monitor Export & Governance Reports

**Objective**: System generates MVOI-monitor export (PIANOo/RIVM format), college governance report (PDF), and audit bundle for external accountability.

### Acceptance Criteria

#### AC-MVI-009-01: MVOI-Monitor CSV Export
- **Given** end-of-year (2026-12-31), organizations aggregated contracts
- **When** system generates "MVOI-Monitor 2026" export
- **Then** output CSV contains per procurement category:
  - Totaal uitgaven (EUR)
  - MVI-conform percentage (%)
  - SROI-realisatie (EUR + %)
  - CO₂ baseline (year) + realization (year) + reduction (%)
  - Circulairatied-index (%)
  - Compliant with PIANOo/RIVM schema (column names, units, decimal places)
- **Validation**: Manual spot-check against underlying openregister data; ±2% tolerance for rounding

#### AC-MVI-009-02: College Governance Report (PDF)
- **Given** Q4 2026 reporting cycle
- **When** MVI-adviseur clicks "Generate College Report" with parameters: periode="2026-Q4", format="PDF"
- **Then** system generates PDF with:
  - **Executive Summary** (1 page): key KPIs, compliance %, SROI realized, portfolio goal progress (CO₂, circular, SROI %)
  - **Goal Progress** (1 page): 3-5 visualization tiles (line charts) showing baseline→current→2030 target for each goal
  - **Top 10 Performers** (1 page): contracts that exceeded MVI targets or realized >150% SROI, with brief narrative
  - **Risk Items** (1 page): contracts <80% SROI realized, pending MVI renewals, upcoming penalties
  - **Policy Recommendations** (1-2 pages): text suggestions for next year (e.g., "Recommend stricter Circular criteria for IT next round based on current 42% compliance")
  - **Audit Trail Summary** (1 page): # of decisions logged, # of proof docs validated, exemptions granted, appeals processed
  - Footer: Generated by [system name] on [date], digital signature from org (optional PKI)

#### AC-MVI-009-03: Auditor Export Bundle
- **Given** external accountant requests audit evidence for MVO annual report verification
- **When** system generates auditor export (admin function, locked to audit-role)
- **Then** deliverables:
  - CSV of all contracts + MVI scores + SROI obligations + realizations (filtered to contract period 2026)
  - List of all mvi_portfolio_doelstelling records + bestuursbesluiten (decision records + dates)
  - Sample of 5 scored tenders: scoring sheets + uploaded proofs + signed award decision
  - SROI sample (5 vendors): submitted realizations + proof docs + coordinator approval signature + calculation worksheets
  - Audit trail excerpt (all MVI/SROI mutations for 2026)
  - Signed attestation: "This export is certified to be complete and accurate as of [date] and matches openregister source data."

#### AC-MVI-009-04: RIVM API Push (If Configured)
- **Given** system is configured with RIVM API endpoint + API key
- **When** MVOI export is finalized (post-validation)
- **When** admin clicks "Push to RIVM"
- **Then** system:
  - Calls RIVM API endpoint with CSV payload
  - Logs response (success/error)
  - Updates status: "Exported to RIVM [date] [API response code]"
  - Sends confirmation email to MVI-adviseur + leadership
  - Retries on transient failure (3x, exponential backoff)

---

## REQ-MVI-010: Integration with Tender Strategy & Governance Decisions

**Objective**: Portfolio goals are linked to governance decisions; users see goal context while creating tenders; goal changes require new decision records.

### Acceptance Criteria

#### AC-MVI-010-01: Governance Decision Linkage
- **Given** a new collegebesluit (city council decision): "CO₂-reductie 49% in 2030 t.o.v. 1990" dated 2023-05-20, kenmerk RB-2023/1234
- **When** MVI-adviseur creates `mvi_portfolio_doelstelling` with:
  - `thema = co2_reductie_percentage`
  - `doelwaarde = 49`
  - `baselineJaar = 1990`
  - `doeljaar = 2030`
- **When** advisor specifies `bestuurlijkAkkoord`:
  - `besluitType = raadsbesluit`
  - `besluitDatum = 2023-05-20`
  - `besluitKenmerk = RB-2023/1234`
- **Then** system:
  - Validates that the decision record exists in decidesk (or logs a link to external doc)
  - Stores goal with immutable governance link
  - Creates audit log: "Portfolio goal created, linked to decision RB-2023/1234"

#### AC-MVI-010-02: Tender UI Goal Context
- **Given** active portfolio goal: "CO₂ -49% by 2030, current 32%, baseline 1990"
- **When** procurement officer creates a new tender for energy supply (estimated 150 MWh)
- **When** officer selects "Energia" (energy) procurement category
- **Then** system displays goal banner in tender UI:
  - "This tender contributes to: CO₂ Reduction 49% by 2030 (currently 32% achieved, 17% to go)"
  - Banner includes clickthrough to goal detail (baseline, current, target, decision record)
  - System auto-suggests MVI criteria tagged with "klimaat_co2" category with boosted weighting

#### AC-MVI-010-03: Goal Revision Requires New Decision
- **Given** existing portfolio goal: "Circular 50% by 2030" (linked to CB-2024/567)
- **Given** new collegebesluit: "Increased ambition: Circular 55% by 2030" (CB-2025/891, dated 2025-06-15)
- **When** MVI-adviseur updates the goal:
  - Changes `doelwaarde = 55`
  - Enters new `bestuurlijkAkkoord` with link to CB-2025/891
- **Then** system:
  - Does NOT overwrite old goal; instead creates new version of record
  - Old version marked with `geldigTot = update_date`
  - New version has `versie = 2`
  - Historical reporting can still reference baseline goal (55% revised to 55%, no impact, but decision chain preserved)
  - Active tenders can optionally be re-evaluated under new goal (optional workflow step: "Reweight MVI criteria to align with new 55% target?")

#### AC-MVI-010-04: Audit Trail of Goal Changes
- **Given** multiple goal revisions over 3-year period
- **When** auditor requests "Goal change history for Circular goal"
- **Then** system returns:
  - Version 1: 50% by 2030, baseline 5%, decision CB-2024/567, created 2024-02-15
  - Version 2: 55% by 2030, baseline 5%, decision CB-2025/891, created 2025-06-15 (revision reason: "Increased ambition per college memo")
  - Timeline view showing which tenders used which goal version (tenders published before version change = old goal, after = new goal)

---

## Non-Functional Requirements

### NFR-001: Performance

- Dashboard loads within 3 seconds (p95)
- SROI obligation calculation completes within 500ms
- Category recommendation engine (AC-MVI-002) completes within 1 second
- RVO import job (weekly) completes within 5 minutes (handles ~2000 criteria)
- MVOI export generation (annual) completes within 10 minutes (handles 500+ contracts)

### NFR-002: Reliability

- System uptime: 99.5% (monthly)
- RVO import job retry logic: 3 retries with exponential backoff, fallback to last-good version
- Database backups: daily, with tested restore procedure
- Data loss: zero tolerance (all financial decisions must be audit-loggable and recoverable)

### NFR-003: Security

- Role-based access control (RBAC): Inkoper, SROI-Coördinator, MVI-Adviseur, Contractmanager, Bestuurder, Audit
- Audit trail: immutable logs of all mutations (no deletion)
- Proof documents encrypted at rest, access logged
- Vendor portal: separate auth realm (SSO or dedicated credentials)
- API authentication: OAuth 2.0 or equivalent

### NFR-004: Usability

- Inkoper onboarding: <4 hours to competency on tender MVI setup
- SROI-Coördinator: portal navigation <2 clicks to approval queue
- Dashboard: single-page load, responsive design (mobile + desktop)
- Error messages: actionable, in Dutch, with remediation suggestions

### NFR-005: Interoperability

- openregister GraphQL schema: versioned, backward compatible
- TenderNed eForms export: validates against eForms schema v0.6+
- CSV exports: UTF-8, semicolon-delimited, conform PIANOo/RIVM spec
- API endpoints: RESTful + GraphQL (for reporting queries)

### NFR-006: Scalability

- Support 1000+ simultaneous users (during tender publication)
- Support 10,000+ contracts in portfolio (query <2s)
- Support 100,000+ SROI realization records (annual export <10min)

---

## Data Validation Rules

| Field | Rule | Error Message |
|-------|------|---------------|
| `mvi_criterium.code` | Matches `^(RVO\|ORG)-[A-Z0-9]{2,}-[A-Z0-9]{2,}$` | "Code must start with RVO or ORG, followed by org/source initials and serial (e.g., RVO-15-CO2-1)" |
| `sroi_verplichting.sroiPercentage` | 0 ≤ value ≤ 100 | "SROI % must be between 0 and 100" |
| `sroi_realisatie.gerealiseerdeWaardeEur` | ≥ 0 | "Realized value cannot be negative" |
| `mvi_portfolio_doelstelling.doeljaar` | ≥ 2024 | "Goal year must be current or future" |
| `mvi_monitoring_rapportage.periode` | Matches `^\d{4}-(Q[1-4]\|H[1-2]\|Jaar)$` | "Period must be format YYYY-QN, YYYY-HN, or YYYY-Jaar" |

