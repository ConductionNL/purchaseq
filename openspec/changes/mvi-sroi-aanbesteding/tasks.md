---
status: ready
created: 2026-05-22
version: 1.0
---

# MVI- en SROI-eisen in Aanbestedingen — Implementation Tasks

## Phase 0: Foundation & Data Model (Week 1–2)

### Data Model & Registry Setup

- [ ] **Register schemas in openregister**
  - [ ] Create `mvi_criterium` schema with fields: code, bron, categorie, inkoopcategorie, titel, omschrijving, niveau, eisType, meetbaarheid, bewijsmiddelen[], geldigVanaf, geldigTot, versie
  - [ ] Create `aanbesteding_mvi_set` schema with fields: aanbestedingRef, criteriumRef, eisType, weging, minimumNiveau, verplichtBewijsmiddel, formuleringInBestek, motivering, geintegreerdIn
  - [ ] Create `sroi_verplichting` schema with fields: aanbestedingRef, contractRef, opdrachtwaardeEur, sroiPercentage, sroiBedragEur, peilDatum, regioPolicy, bouwblokkenModel, realisatieperiode{start, eind}, tussentijdseRapportagesFrequentie, eindrapportageDatum, status, vrijstellingMotivering
  - [ ] Create `sroi_bouwblok` schema with fields: code, categorie, subcategorie, weegfactorEur, eenheid, voorwaarden, geldigVanaf, geldigTot, bouwblokkenVersie
  - [ ] Create `sroi_realisatie` schema with fields: verplichtingRef, periode, bouwblokRef, aantal*, gerealiseerdeWaardeEur, bewijsmiddelen[], verantwoordingsbron, goedgekeurd, goedgekeurdDoor, goedkeuringsDatum, opmerkingen
  - [ ] Create `mvi_monitoring_rapportage` schema with fields: contractRef, periode, mviSetRef, gerapporteerdeWaarde, bewijsmiddelRef, voldoetAanEis, afwijking, verbetervoorstel, boeteOpgelegdEur, bonusToegekendEur
  - [ ] Create `mvi_portfolio_doelstelling` schema with fields: organisatieRef, thema, doeljaar, doelwaarde, baselineJaar, baselineWaarde, huidigeWaarde, voortgang, bestuurlijkAkkoord{}
  - [ ] Test all schema migrations; verify indices created (see Design NFR-005)

- [ ] **Seed openregister with reference data**
  - [ ] Import 3–5 RVO reference criteria per category (from RVO criteriatool) into `mvi_criterium` as seed
  - [ ] Import Bouwblokkenmethode 2023 bouwblokken (BB-WW-12, BB-Scholing, BB-LocaleInkoop, etc.) into `sroi_bouwblok`
  - [ ] Verify audit trail enabled on all tables (immutable_audit = true)

- [ ] **Set up audit trail & versioning**
  - [ ] Enable automatic timestamp + user tracking on all schemas (createdAt, createdBy, updatedAt, updatedBy)
  - [ ] Implement soft-delete pattern (deletedAt column, never hard-delete)
  - [ ] Audit log middleware: log all mutations with old/new JSON diff
  - [ ] Test rollback/recovery scenario (restore deleted record from audit trail)

---

## Phase 1: RVO Import & Criteria Management (Week 3–4)

### RVO Integration & Import Pipeline

- [ ] **Set up RVO API connector**
  - [ ] Implement HTTP client for mvicriteria.nl API
  - [ ] Parse RVO response format (criteria list with code, titel, omschrijving, niveau, kategorie)
  - [ ] Map RVO fields to `mvi_criterium` schema (code → RVO-{VERSION}-{CATEGORY}-{ID}, bron → rvo_criteriatool, etc.)
  - [ ] Implement versioning logic: on new import, diff against existing; mark old records with geldigTot if content changed

- [ ] **Implement import job in n8n / pipelinq**
  - [ ] Create workflow: RVO API call → parse → diff → upsert mvi_criterium
  - [ ] Schedule: weekly Monday 03:00 UTC
  - [ ] Implement retry logic: 3 attempts with exponential backoff; log errors
  - [ ] On success: email MVI-team with delta summary ("Added 5, updated 12, no change 2000")
  - [ ] On failure: ops alert (Slack) + log last-good timestamp + do NOT interrupt active tenders

- [ ] **Implement org-specific criteria management**
  - [ ] Create UI form: [code, bron (dropdown: organisatie_eigen), categorie, inkoopcategorie[], titel, omschrijving, niveau, eisType, meetbaarheid, bewijsmiddelen[]]
  - [ ] On save: auto-assign `bron = organisatie_eigen`, `code = ORG-{ORG_ID}-{SERIAL}`, versie = 1
  - [ ] Validate: no name/code duplication within org
  - [ ] Audit: log who created, timestamp
  - [ ] Display: org criteria appear alongside RVO criteria in selector dropdowns (badge "Org-specific")

- [ ] **Implement criteria version history**
  - [ ] API endpoint: GET /api/mvi-criteria/{id}/history → list all versions with geldigVanaf/geldigTot, diff sidebar
  - [ ] UI: timeline view showing version changes + who changed what + why (import vs. manual edit)
  - [ ] Test: older tenders correctly reference their locked-in version

---

## Phase 2: Tender MVI Intake & Recommendation (Week 5–6)

### MVI Set Recommendation Engine

- [ ] **Implement recommendation logic**
  - [ ] Input: tender CPV code (or manual category selection), region, ambitieniveau (basis | significant | ambitieus)
  - [ ] Lookup: RVO recommended criteria for category × niveau
  - [ ] Filter: org portfolio goals (boost weighting for circulair if goal = circulair 50%)
  - [ ] Output: `List<Recommendation>` with code, titel, eisType, suggestedWeight, formulation (from template)
  - [ ] Test data: 5 tenders across 5 categories (IT, catering, cleaning, bouw, energy)

- [ ] **Implement MVI set UI in procurement flow**
  - [ ] Procurement workflow step: "Configure MVI" (after tender metadata, before bestek drafting)
  - [ ] Button: "Suggest MVI Set for {category} ({ambitieniveau})"
  - [ ] Display returned recommendation; allow officer to:
    - [ ] Accept as-is → save to aanbesteding_mvi_set, move to next step
    - [ ] Customize: add/remove criteria, adjust weights, change formulation text
    - [ ] Compare: side-by-side view of "Basis" vs. "Significant" vs. "Ambitieus" sets
  - [ ] On save: create aanbesteding_mvi_set record(s), versioning = 1, createdBy = officer

- [ ] **Portfolio goal linkage in UI**
  - [ ] When MVI set is recommended, display banner: "This tender contributes to: [Goal Name] (currently X%, target Y% by 20ZZ)"
  - [ ] Banner clickthrough → goal detail page (baseline, trajectory, decision record)
  - [ ] Test: goal banner appears for ≥1 recommended criterion that maps to goal

- [ ] **Validate MVI formulation**
  - [ ] Check: formuleringInBestek is non-empty, meets min length (50 chars), is Dutch language (spell-check)
  - [ ] Warn: if formulation deviates >20% from default (highlight changes)

---

## Phase 3: Scoring & EMVI Integration (Week 7–8)

### Gunning Score Matrix & Proof Validation

- [ ] **Implement score matrix builder**
  - [ ] For each MVI gunning criterion in tender:
    - [ ] Auto-suggest score matrix (e.g., CO₂-Ladder: 5→100, 4→80, …, 0→0)
    - [ ] Allow override per level (with logged motivation)
    - [ ] Show weighting: if criterion weight = 25pt, max score = 25 pt (if max possible = 100pt)
  - [ ] Save matrix to `aanbesteding_mvi_set` record

- [ ] **Implement proof submission & validation**
  - [ ] Tender publication workflow: bidders upload proof documents (PDFs, images) for each MVI criterion
  - [ ] For certificate-based criteria (CO₂-Ladder, Energy Star, etc.):
    - [ ] Extract candidate metadata via OCR (issuer, level, expiry date)
    - [ ] Validate format (SKAO CO₂-Ladder format v2+ expected)
    - [ ] Check expiry: if expired → mark invalid (red), if expiring <180d → mark at-risk (orange)
    - [ ] Log extraction + validation result to audit trail
  - [ ] For qualitative criteria (e.g., "repair-right promise"):
    - [ ] Store proof as attachment (manual verification by commission)

- [ ] **Implement scoring commission UI**
  - [ ] Per MVI gunning criterion, scoring form with:
    - [ ] Bidder name + uploaded proof display
    - [ ] Auto-filled score (based on proof level + matrix), with confidence badge (green/orange/red)
    - [ ] Manual override field (with mandatory motivation text)
    - [ ] Submit → system logs: bidder, score, proof validity, override (if any)
  - [ ] Aggregate scoring: compute total MVI score per bidder
  - [ ] Save scores to `aanbesteding_mvi_set` record (denormalized) for contract generation

- [ ] **Test scoring workflow**
  - [ ] Simulate 3 bidders with CO₂-Ladder proofs (L5, L3, none)
  - [ ] Verify auto-scoring, override logging, final scores locked

---

## Phase 4: SROI Obligation Calculation (Week 9–10)

### SROI Determination & Policy Integration

- [ ] **Implement SROI policy registry**
  - [ ] Admin UI: configure regional policies
    - [ ] Policy name (e.g., "Regio Rijnmond SROI 2024")
    - [ ] Rules: [{ procurementCategory, orderValueThreshold, sroiPercentage, exemptions[] }]
    - [ ] Example: "Diensten ≥ €250k → 5%, EXCEPT cloud-only"
    - [ ] Versioning: each policy revision creates new version with geldigVanaf/geldigTot
  - [ ] Test data: policies for 3 regions (Rotterdam, Amsterdam, Den Haag) with at least 5 rules each

- [ ] **Implement SROI obligation calculator**
  - [ ] Input: tender value (EUR), procurement category (enum), region
  - [ ] Logic:
    - [ ] Lookup applicable policy (current region, current date)
    - [ ] Match category + value to policy rules
    - [ ] Return: sroiPercentage, sroiBedragEur (= value × %), bouwblokkenModel (from policy), reportsFrequency
  - [ ] Output: sroi_verplichting record, status = concept
  - [ ] Test: 10 scenarios across categories (diensten, werken, other) + regions

- [ ] **Implement exemption workflow**
  - [ ] Officer can request exemption with motivering (text field)
  - [ ] System routes to SROI-Coördinator (workflow step, with notification)
  - [ ] Coördinator reviews: approve → status = vrijgesteld, sroiBedragEur = 0; reject → status = vastgelegd, obligation stands
  - [ ] Audit log: who requested, when, reason, decision, by whom, when
  - [ ] Test: approve 1, reject 1, verify state transitions

- [ ] **Implement below-threshold suggestion UI**
  - [ ] For orders below policy threshold: show suggestion "Voluntary SROI available. Apply 2%?"
  - [ ] Officer can accept or dismiss
  - [ ] If accept: recalculate obligation with 2% (or org-configured voluntary %)

- [ ] **Integration with tender workflow**
  - [ ] After MVI set is confirmed, next step: "Calculate SROI Obligation"
  - [ ] Auto-calculate on save; officer reviews + confirms → status = vastgelegd
  - [ ] Display: obligation amount, percentage, deadline, bouwblokkenmodel

---

## Phase 5: Vendor SROI Reporting & Approval (Week 11–13)

### Vendor Portal & Realization Workflow

- [ ] **Build vendor portal (separate auth realm)**
  - [ ] Authentication: SSO (OIDC) or dedicated username/password
  - [ ] Vendor dashboard: list of active contracts + SROI obligations
  - [ ] Per obligation: due date, status (pending/approved/rejected), cumulative realized amount, remaining due
  - [ ] Mobile-responsive design (vendor may use phone/tablet)

- [ ] **Implement realization report submission UI**
  - [ ] Form per obligation + period (e.g., "Q2 2026"):
    - [ ] Bouwblok selector (dropdown, filtered to applicable types per obligation)
    - [ ] Quantity inputs: [aantalUren OR aantalFtes OR aantalPersonen] (radio: choose 1 unit type)
    - [ ] System auto-calculates: gerealiseerdeWaardeEur = quantity × weegfactor
    - [ ] Proof upload: file input(s) for loonadministratie, plaatsingsverklaring, etc. (drag-drop, multi-file)
    - [ ] Optional notes field
    - [ ] Submit → creates sroi_realisatie record with goedgekeurd = false

- [ ] **Implement proof document parsing & validation**
  - [ ] For payroll extract (loonadministratie):
    - [ ] Extract: employee names, gross wages, period (date range), total hours
    - [ ] Validate: period matches reported periode (Q2 = Apr–Jun); total hours match reported hours ±5%
  - [ ] For placement letter (plaatsingsverklaring):
    - [ ] Extract: candidate name, placement start date, hours/week, duration (weeks)
    - [ ] Validate: dates match payroll; placement dates are contiguous; hours ≥ bouwblok minimum
    - [ ] Check: signature present (required for legal validity)
  - [ ] For bouwblok "lokale_inkoop": require onderaannemerscontract as proof
  - [ ] Log all validation results; if validation succeeds, show green checkmark; if partial, orange; if failed, red
  - [ ] Allow manual correction (vendor can re-upload if parse errors detected)

- [ ] **Implement anti-stacking validation**
  - [ ] For "lokale_inkoop" bouwblokken: check that vendor does not subcontract without org knowledge
  - [ ] Rule: if bouwblok = "lokale_inkoop", onderaannemerscontract must exist AND org must approve supplier before invoice
  - [ ] Test: submit lokale_inkoop without contract → validation fails with clear message

- [ ] **Test vendor portal**
  - [ ] Submit realistic reports for 3 obligations:
    - [ ] WW placement (payroll + letters)
    - [ ] Scholing (certificates)
    - [ ] Lokale inkoop (contract + order)
  - [ ] Verify auto-calculations, proof parsing, validation feedback

---

## Phase 6: SROI Approval & Escalation (Week 14–15)

### SROI-Coördinator Review & Risk Management

- [ ] **Implement approval queue UI**
  - [ ] SROI-Coördinator dashboard: list of pending realization reports (goedgekeurd = false)
  - [ ] Per report: vendor name, period, claimed amount, proof validation status (green/orange/red)
  - [ ] Detail view: show submitted data + parsed proof contents + vendor notes
  - [ ] Action buttons: [Approve] [Reject + Request Revision] [Escalate]
  - [ ] Mandatory opmerkingen field (min 10 chars) before approval/rejection

- [ ] **Implement approval logic**
  - [ ] On "Approve":
    - [ ] Set goedgekeurd = true, goedgekeurdDoor = coordinator_id, goedkeuringsDatum = now()
    - [ ] Update sroi_verplichting cumulative realisatie (sum all approved realizations for period)
    - [ ] Compare to plan: if realization ≥ 95% of expected pro-rata, mark green; 80–95% = orange; <80% = red
    - [ ] Send confirmation email to vendor + CC contractmanager
    - [ ] Log approval to audit trail
  - [ ] On "Reject":
    - [ ] Set goedgekeurd = false
    - [ ] Send email to vendor with rejection reason + deadline to resubmit (default 14 days)
    - [ ] Keep report in queue (not finalized)
  - [ ] On "Escalate":
    - [ ] Create escalation ticket (linked to obligation + realization)
    - [ ] Notify category manager + MVI-adviseur
    - [ ] Set internal flag for management review

- [ ] **Implement deadline reminder automation**
  - [ ] n8n job: daily 09:00 local
    - [ ] Find all sroi_verplichting with upcoming deadline (≤7 days)
    - [ ] Filter by reportsFrequency (maandelijks, per_kwartaal, halfjaarlijks)
    - [ ] Send email to vendor: "Your {PERIODE} SROI report due {DEADLINE} (X days)"
    - [ ] Include portal link + last reported amount (if any)
    - [ ] CC contractmanager
    - [ ] Log reminder sent
  - [ ] 2nd reminder: if vendor has not reported 3 days before deadline
    - [ ] Same email with escalation warning + phone number of contractmanager

- [ ] **Implement lag detection & escalation**
  - [ ] n8n job: daily 18:00 local
    - [ ] For each sroi_verplichting in_uitvoering:
      - [ ] Calculate expected cumulative amount (pro-rata of elapsed time from start to today)
      - [ ] Sum approved realizations to date
      - [ ] Lag = (expected - actual) / expected
    - [ ] If lag ≥ 15%:
      - [ ] Mark obligation risicovol = true
      - [ ] Create escalation ticket with lag %, assigned to SROI-Coördinator
      - [ ] Notify contractmanager + MVI-adviseur
      - [ ] Log escalation
    - [ ] Test: simulate contracts with 20%, 35%, 50% lag; verify escalation triggers

- [ ] **Implement end-of-period penalty calculation**
  - [ ] When realization period ends and all reports are finalized:
    - [ ] Calculate total realized amount vs. sroi_verplichting.sroiBedragEur
    - [ ] If realized < obligation:
      - [ ] Shortfall = obligation - realized
      - [ ] Look up contract penalty clause (stored in contract record)
      - [ ] Penalty = shortfall × penalty_multiplier (e.g., 25% → shortfall × 1.25)
      - [ ] Create mvi_monitoring_rapportage with boeteOpgelegdEur = penalty
      - [ ] Generate invoice for vendor + contractmanager
      - [ ] Log penalty to audit trail
    - [ ] Test: €20k obligation, €16k realized → €4k shortfall → €5k penalty (25% multiplier)

---

## Phase 7: Contract MVI Monitoring (Week 16–17)

### Post-Award MVI Obligation Tracking

- [ ] **Implement contract MVI obligation linking**
  - [ ] After award: create `mvi_monitoring_rapportage` records for each contract MVI obligation (eisType = contracteis)
  - [ ] Pre-populate:
    - [ ] contractRef (from award decision)
    - [ ] mviSetRef (from aanbesteding_mvi_set)
    - [ ] periode = each contract year (e.g., "2026-Jaar", "2027-Jaar")
    - [ ] voldoetAanEis = null (pending vendor report)
  - [ ] Per obligation, calculate renewal deadline:
    - [ ] For certificate-based obligations (CO₂-Ladder): renewal 30 days before expiry or anniversary, whichever earlier
    - [ ] For annual reporting: anniversary date of contract start

- [ ] **Implement annual renewal reminders**
  - [ ] n8n job: daily 09:00 local
    - [ ] Find all mvi_monitoring_rapportage with renewal_deadline ≤ 60 days
    - [ ] Filter by mviSetRef → look up criterion type (CO₂-Ladder, etc.)
    - [ ] Send email to vendor + contractmanager: "CO₂-Ladder renewal due {DEADLINE} (X days). Please upload updated certificate."
    - [ ] Include portal link + old certificate details
    - [ ] Set status = pending_renewal
  - [ ] Test: create contract with CO₂ obligation expiring in 90 days; verify reminder sent at 60-day mark

- [ ] **Implement compliance validation**
  - [ ] Contractmanager logs compliance report (via portal or attached document):
    - [ ] Criterion, reported value (e.g., "CO₂-Ladder L4"), proof (file upload), assessment (compliant yes/no)
  - [ ] System validates proof:
    - [ ] For certificates: parse expiry, validate signature, check level against obligation
    - [ ] If expired or expired_recently (>30d ago): mark voldoetAanEis = false, afwijking = "Certificate expired"
    - [ ] If compliant: voldoetAanEis = true, no penalty
    - [ ] If non-compliant: trigger boete-berekening workflow
  - [ ] Test: upload expired cert → system marks non-compliant → penalty calculated

- [ ] **Implement over-performance bonus tracking**
  - [ ] If vendor reports higher performance (e.g., L5 instead of required L4):
    - [ ] Check contract for bonus clause (if present)
    - [ ] System auto-calculates bonus (e.g., "L5 = €5k bonus")
    - [ ] Create mvi_monitoring_rapportage with bonusToegekendEur = amount
    - [ ] Log over-performance for portfolio reporting
  - [ ] Test: report L5 when L4 required + contract has bonus clause → bonus calculated

- [ ] **Implement multi-year tracking**
  - [ ] For multi-year contracts, create separate mvi_monitoring_rapportage per year
  - [ ] Portfolio reporting aggregates YoY compliance trends
  - [ ] Test: 3-year contract with 3 annual compliance records; verify trend line in dashboard

---

## Phase 8: Portfolio Dashboard (Week 18–19)

### Real-Time MVI/SROI Reporting

- [ ] **Implement portfolio aggregation queries**
  - [ ] GraphQL query: `portfolioKPIs(orgId, date)` →
    - [ ] Total spend (EUR) across active contracts
    - [ ] MVI-compliant contracts (%) — defined as ≥1 MVI gunning criterion passed
    - [ ] Total SROI realized (EUR) + (%) of obligations
    - [ ] Per portfolio goal: huidigeWaarde (computed from mvi_monitoring_rapportage + sroi_realisatie)
  - [ ] Ensure query performance: <2s for 10,000 contracts (use caching + indices)

- [ ] **Implement mydash portfolio widget**
  - [ ] GraphQL data source: portfolioKPIs query
  - [ ] Tile layout:
    - [ ] MVI Compliance (large %, green/orange/red traffic light)
    - [ ] SROI Realized (EUR + %, gauge)
    - [ ] Per Goal (CO₂, Circular, SROI %): progress bar + % + target year
  - [ ] Auto-refresh every 4 hours (silent)
  - [ ] Timestamp: "Last refresh: {DATE} {TIME} UTC"

- [ ] **Implement goal tracking visualization**
  - [ ] Line chart per goal:
    - [ ] X-axis: years (baseline year → 2030)
    - [ ] Y-axis: value (% reduction for CO₂, % for circular, etc.)
    - [ ] Baseline point (baseline year, baseline value)
    - [ ] Current point (today, huidigeWaarde)
    - [ ] Target point (target year, doelwaarde)
    - [ ] Linear path line (connecting baseline → target)
    - [ ] Actual realization line (baseline → all intermediate actuals → current)
  - [ ] Traffic light: green if on track, orange if at risk, red if off-track
  - [ ] Tooltip: "At current pace, target achievable by [year]. Need X% annual reduction to stay on track."

- [ ] **Implement drill-down to categories**
  - [ ] Click MVI Compliance tile → category view:
    - [ ] Table: category, # contracts, MVI %, spend, SROI realized
    - [ ] Filter: show only MVI-compliant, show by status (active/ended), show by risk (red/orange)
    - [ ] Each row clickable to contract list

- [ ] **Implement drill-down to contracts**
  - [ ] Click category row → contract list:
    - [ ] Table: contract ID, vendor, start/end dates, spend, MVI scores, SROI obligation, SROI realized, penalty/bonus
    - [ ] Status badge (active/completed/at-risk)
    - [ ] Click contract → full detail page (all MVI scores, monitoring reports, proof docs if viewer has access)

- [ ] **Implement dashboard refresh automation**
  - [ ] n8n job: every 4 hours
    - [ ] Recompute huidigeWaarde for all mvi_portfolio_doelstelling
    - [ ] Call mydash widget mutation to refresh data source
    - [ ] Log refresh timestamp

- [ ] **Test dashboard**
  - [ ] Load with 100 contracts + 5 portfolio goals
  - [ ] Verify KPI aggregation accuracy (spot-check 5 contracts)
  - [ ] Verify refresh timing (every 4 hours)
  - [ ] Verify drill-down navigation (3 levels: org → category → contracts)

---

## Phase 9: Annual MVOI Export & Reporting (Week 20–21)

### Governance Reporting & External Accountability

- [ ] **Implement MVOI-Monitor CSV export**
  - [ ] Query: all contracts for year X, grouped by procurement category
  - [ ] Per category, compute:
    - [ ] Total spend (sum of all contract values)
    - [ ] MVI-conformance % (# MVI-compliant contracts / total)
    - [ ] SROI realized (EUR + %)
    - [ ] CO₂ reduction (baseline + realization + %)
    - [ ] Circular index (%)
  - [ ] Output CSV: columns per PIANOo/RIVM spec (validate schema)
  - [ ] Test: generate for 2025 calendar year with 100+ contracts; spot-check 5 categories manually

- [ ] **Implement college governance report generator**
  - [ ] PDF template with sections:
    - [ ] Executive summary (1 page)
    - [ ] KPI tiles (dashboard KPIs + sparklines)
    - [ ] Goal progress charts (line chart per goal)
    - [ ] Top 10 performers (best MVI scores, highest SROI %)
    - [ ] Risk items (low-SROI contracts, pending renewals)
    - [ ] Policy recommendations (text, 1-2 pages)
    - [ ] Audit trail summary (# decisions, # proofs, exemptions, appeals)
  - [ ] Dynamic generation from GraphQL queries (not static template)
  - [ ] Signature field (org digital signature, optional)
  - [ ] Test: generate Q4 2026 report; review content accuracy + formatting

- [ ] **Implement auditor export bundle**
  - [ ] Admin-only function
  - [ ] Deliverables:
    - [ ] CSV: all contracts + MVI scores + SROI obligations/realizations (year X)
    - [ ] Portfolio goals list + decision records
    - [ ] Sample set (5 scored tenders): award sheet + uploaded proofs
    - [ ] Sample set (5 vendors): realization reports + proof docs + coordinator signatures
    - [ ] Audit log excerpt (all MVI/SROI mutations for year X)
    - [ ] Signed attestation (PDF): "Certified complete & accurate per [date], matches source data"
  - [ ] ZIP file delivery
  - [ ] Test: generate for 2025; verify all files present + sizes reasonable

- [ ] **Implement RIVM API integration (optional, if configured)**
  - [ ] Admin config: RIVM endpoint URL + API key
  - [ ] On finalized MVOI export:
    - [ ] Button: "Push to RIVM"
    - [ ] System calls RIVM API with CSV payload
    - [ ] Log response (success/error code)
    - [ ] On success: update status "Exported to RIVM {DATE}", send confirmation email
    - [ ] On failure: retry 3x, exponential backoff; ops alert if all fail
  - [ ] Test: with RIVM test endpoint (if available)

---

## Phase 10: Integration & Cross-System Linking (Week 22–23)

### openregister, purchaseq, TenderNed, docudesk Integration

- [ ] **Register API endpoints in openregister catalog**
  - [ ] Document all REST + GraphQL endpoints (see Design section)
  - [ ] Versioning: v1.0 initial release
  - [ ] Auth: OAuth 2.0 bearer tokens + role-based scopes (see NFR-003)

- [ ] **Integrate with purchaseq tender UI**
  - [ ] Implement "Configure MVI" workflow step (AC-MVI-002)
  - [ ] Call: POST /api/tenders/{tenderId}/mvi-set/recommend
  - [ ] On recommend: display set + allow customization + save
  - [ ] Call: POST /api/tenders/{tenderId}/sroi-obligation
  - [ ] Display obligation + allow exemption request

- [ ] **Implement TenderNed eForms export**
  - [ ] Map MVI criteria to eForms BT-755 (Accessibility), BT-805/806 (Green Procurement)
  - [ ] Map SROI obligation to BT-756 (Social Conditions)
  - [ ] Test: export sample tender to eForms v0.6 schema; validate XML

- [ ] **Integrate with docudesk**
  - [ ] Template library: pre-written bestek fragments per MVI criterion
  - [ ] On MVI set confirmed: retrieve matching fragments from docudesk
  - [ ] Auto-insert into bestek document (placeholder replacement)
  - [ ] Test: insert 3 criteria fragments; verify grammar + layout

- [ ] **Integrate with decidesk (portfolio goals)**
  - [ ] On portfolio goal creation: link to decision record in decidesk
  - [ ] On goal update: require new decision record + validation
  - [ ] Test: create goal + link to council decision; verify link persists

- [ ] **Integration testing**
  - [ ] End-to-end: create tender → recommend MVI → calculate SROI → publish to TenderNed → receive SROI report → approve → contract → monitor
  - [ ] Test data: 3 complete scenarios across 3 procurement categories
  - [ ] Verify data consistency across systems (openregister → purchaseq → mydash)

---

## Phase 11: Training & Rollout (Week 24)

### User Enablement & Deployment

- [ ] **Develop training materials**
  - [ ] Inkoper guide (4h): tender setup, MVI recommendation, scoring, SROI obligation
  - [ ] SROI-Coördinator guide (8h): approval queue, escalation, penalty calculation
  - [ ] MVI-Adviseur guide (6h): criteria curation, portfolio goals, reporting
  - [ ] Contractmanager guide (4h): compliance monitoring, renewals, bonuses
  - [ ] Videos: 5–10 short demo clips (5–10 min each, Dutch)

- [ ] **Pilot deployment**
  - [ ] Select 5 real tenders across 5 categories
  - [ ] Execute full workflow with actual procurement staff
  - [ ] Collect feedback on UI, performance, usability
  - [ ] Fix high-priority issues (UI bugs, performance <2s dashboard load)
  - [ ] Document any process deviations from spec

- [ ] **Production readiness checklist**
  - [ ] All unit tests passing (>80% code coverage)
  - [ ] All integration tests passing (end-to-end scenarios)
  - [ ] Performance benchmarks met (see NFR-001)
  - [ ] Security audit completed (OWASP top 10 review)
  - [ ] Data backup + restore procedure tested
  - [ ] Audit trail working (immutable logging confirmed)
  - [ ] Documentation complete (API docs, user guides, architecture)

- [ ] **Soft launch (1 org, 10 tenders)**
  - [ ] Deploy to single organization for 2-week soft launch
  - [ ] Monitor: error rates, performance, user feedback
  - [ ] Iterate: fix bugs, clarify UI, tune performance
  - [ ] Collect success metrics (see Proposal success criteria)

- [ ] **Full rollout (all orgs)**
  - [ ] Gradual rollout: 10 orgs → 50 orgs → 500+ orgs (staggered over 4 weeks)
  - [ ] Monitor: error rates, support ticket volume, user adoption
  - [ ] Executive comms: announce via PIANOo / RIVM channels

---

## Phase 12: Ongoing Operations (Beyond Release)

### Maintenance & Optimization

- [ ] **Monitor system health**
  - [ ] Daily: uptime check (SLA 99.5% monthly)
  - [ ] Daily: error log review (spike detection)
  - [ ] Weekly: RVO import job status (verify weekly run successful)
  - [ ] Weekly: n8n job status (reminders, escalations, dashboard refresh)
  - [ ] Monthly: performance review (query times, lock contention, cache hit rates)

- [ ] **Support triage & bug fixes**
  - [ ] Triage user-reported issues (priority: high/medium/low)
  - [ ] Critical bugs (data loss, scoring errors): fix within 24h
  - [ ] High bugs (workflow blocking): fix within 1 week
  - [ ] Medium/low: backlog for next sprint

- [ ] **RVO criteria updates**
  - [ ] Quarterly: RVO publishes new criteria → import job auto-updates
  - [ ] Manual review: spot-check 5 imported criteria (content accuracy)
  - [ ] Org notification: alert users if any active tenders affected by version change

- [ ] **Policy updates**
  - [ ] Annually (before budget year): review regional SROI policies
  - [ ] Update sroiPercentage, bouwblokkenModel, exemption rules
  - [ ] Version policy records; audit trail of all changes

- [ ] **Feature roadmap (future phases)**
  - [ ] Phase 13 (later): Predictive analytics (forecast portfolio goal achievement)
  - [ ] Phase 14 (later): AI-assisted scoring (auto-score based on proof documents)
  - [ ] Phase 15 (later): Peer benchmarking (compare org's MVI/SROI performance vs. other orgs)

---

## Acceptance Test Scenarios

### Scenario 1: Full Tender Workflow (Cleaning Services)

**Setup**:
- Organization: Municipality Rotterdam
- Tender: Annual cleaning services, estimated €400,000
- Portfolio goal: Circular 50% by 2030 (currently 18%)

**Steps**:
1. Procurement officer creates tender, selects "Cleaning" category
2. System recommends MVI set (CO₂, Social inclusion, Circular)
3. Officer customizes: boost circular weight to 30pt (goal alignment)
4. System calculates SROI: 5% × €400k = €20k obligation
5. Officer publishes tender to TenderNed (eForms with MVI/SROI criteria)
6. 3 bidders submit; 1 has ISO 14001 (circular), 2 do not
7. Scoring commission scores circular criterion (auto-fill from proof, no override needed)
8. Vendor A awarded (best score: €20k contract value)
9. SROI obligation created: status = vastgelegd, deadline Q1-Q4 2027
10. Vendor A reports Q1 2027: 5 WW-placements, 13 weeks, 32 hrs/wk = €15.6k realized
11. SROI-Coördinator approves (proof valid, anti-stacking check OK)
12. Q2–Q4: vendor continues reporting, cumulatively reaches €20.2k (104% realized)
13. End-of-period: no penalty (>95% realized)
14. Contract obligation (circular): annual renewal in 2028, vendor renews ISO 14001 cert
15. Portfolio reporting: circular spend updated (€400k added to compliant base, overall % = 21%)

**Acceptance**: All records created, approvals logged, portfolio KPI updated

---

### Scenario 2: Exemption Request (Cloud Software)

**Setup**:
- Tender: Cloud SaaS license, €800,000
- Initial SROI obligation: 5% = €40,000

**Steps**:
1. Procurement officer recognizes: "cloud-only, no NL headcount"
2. Officer clicks "Request Exemption", submits motivering
3. SROI-Coördinator reviews, approves exemption
4. Status changes: vastgelegd → vrijgesteld, sroiBedragEur = 0
5. Tender published without SROI obligation
6. Contract awarded; no SROI realization reports required
7. Audit trail: exemption request + approval visible to auditors

**Acceptance**: Exemption approved, obligation zeroed, audit trail complete

---

### Scenario 3: Escalation & Penalty (Lag Detection)

**Setup**:
- Tender: Schoonmaak, €300k
- SROI obligation: €15k, annual (Q1-Q4 2027)
- Expected pro-rata by mid-year (Q1-Q2): €7.5k

**Steps**:
1. Q1 report: vendor submits €3k (40% of plan)
2. Q2 arrives; vendor has reported nothing new
3. Escalation job runs: actual €3k vs. expected €7.5k = 60% lag
4. System flags as risicovol, creates escalation ticket
5. SROI-Coördinator calls vendor (contractmanager CC'd)
6. Vendor explains: team turnover, hiring delays
7. Q2 report submitted (late, but accepted): €5k
8. Cumulative: €8k (53% of plan by mid-year)
9. Still at risk (need €14k by end-year if linear), but improving
10. Q3-Q4: vendor reports €4k + €2k, cumulative = €14k (93% realized)
11. End-of-period: realized 93% < 95% threshold → penalty = (€15k − €14k) × 1.25 = €1.25k
12. Invoice issued, contractmanager collects

**Acceptance**: Escalation triggered at 60% lag, penalty calculated at end, audit trail complete

---

### Scenario 4: Portfolio Reporting (Quarterly to College)

**Setup**:
- Organization: Municipality Amsterdam
- Date: 2027-03-31 (end of Q1 2027)
- Portfolio: 150 active contracts, €50M annual spend, 3 portfolio goals (CO₂, Circular, SROI %)

**Steps**:
1. MVI-Adviseur logs in, clicks "Generate Q1 2027 Report"
2. System queries openregister for:
   - All contracts active in Q1, grouped by category + goal
   - MVI compliance %, SROI realized %, goal progress
3. Dashboard loads: KPIs aggregated, charts rendered
4. Adviseur customizes report (select goals to include, date range)
5. Clicks "Generate PDF"
6. System produces 10-page PDF: exec summary, KPIs, goal charts, top performers, risk items, recommendations
7. Adviseur reviews, adds executive text (2 paragraphs), clicks "Send to College"
8. PDF + covering memo sent to college secretary + relevant wethouders
9. College reviews in meeting, discusses recommendations
10. Within 2 weeks, auditor requests audit bundle
11. System generates: CSV (contracts, scores, SROI), goals list, 5-sample tenders, 5-sample SROI reports, audit log, signed attestation
12. Auditor downloads ZIP, reconciles to financials, signs off

**Acceptance**: Report generated, sent to college, audit bundle created

---

## Success Metrics (Post-Launch)

Track these KPIs for 6 months post-launch:

- **Adoption**: ≥80% of newly published tenders include ≥2 MVI criteria (vs. 30% pre-launch)
- **Compliance**: ≥95% of SROI deadline reports submitted on time (no escalations due to missed deadlines)
- **Quality**: ≥90% of SROI realizations approved on first submission (no rejections due to invalid proof)
- **Speed**: Portfolio dashboard loaded within 3 seconds (p95); SROI obligation calculated in <500ms
- **Satisfaction**: ≥4/5 user satisfaction on training (survey, post-training)
- **Audit**: 100% of MVI decisions + SROI approvals auditable with full chain of proof documents

