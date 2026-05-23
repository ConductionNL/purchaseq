---
status: tasks
created: 2026-05-23
---

# Tasks: Raamovereenkomst en Minicompetitie

## Phase 1: Data Layer & Persistence

- [ ] **Create database schema for `raamovereenkomst`**
  - Create table with all fields per design.md
  - Add indexes on `(nummer, aanbesteder_id)`, `(status, eindDatum)`, `(status, verbruiktVolumeEur)`
  - Add constraints: `maximumVolumeEur > 0`, `eindDatum > startDatum`, `looptijdJaren <= 4` (klassieke) or `<= 8` (speciale)
  - Create migration file with rollback
  - Test creation with sample data from design.md

- [ ] **Create database schema for `raamovereenkomst_leverancier`**
  - Create table with all fields
  - Add indexes on `(raamovereenkomst_id, leverancier_kvk)`, `(raamovereenkomst_id, actief)`
  - Add FK constraints to raamovereenkomst
  - Test creation and supplier association

- [ ] **Create database schema for `minicompetitie`**
  - Create table with all fields
  - Add indexes on `(raamovereenkomst_id, status)`, `(raamovereenkomst_id, sluitingsDatum)`, `(aanvragende_user_id, status)`
  - Add FK constraints to raamovereenkomst
  - Add CHECK constraint: `sluitingsDatum > publicatieDatum`
  - Test creation

- [ ] **Create database schema for `minicompetitie_inschrijving`**
  - Create table with all fields
  - Add FK constraints to minicompetitie and raamovereenkomst_leverancier
  - Add UNIQUE constraint: `(minicompetitie_id, leverancier_id)` (one bid per supplier per competition)
  - Add CHECK: `inschrijvings_bedrag_eur > 0`
  - Test creation

- [ ] **Create database schema for `minicompetitie_beoordeling`**
  - Create table with all fields
  - Add FK constraints to inschrijving
  - Add indexes on `(inschrijving_id, criterium_id)`, `(inschrijving_id, beoordelaar_user_id)`
  - Test creation

- [ ] **Create database schema for `raamovereenkomst_verlenging`**
  - Create table with all fields
  - Add FK constraint to raamovereenkomst
  - Test creation

- [ ] **Create database schema for `minicompetitie_proces_verbaal`**
  - Create table with all fields
  - Add FK constraint to minicompetitie
  - Add immutability trigger: prevent UPDATE or DELETE once `immutable = true`
  - Test creation

- [ ] **Create migration for all schemas**
  - Combine all schema creation into a single migration file
  - Version it as `2026_05_23_001_raamovereenkomst_minicompetitie_initial.sql`
  - Verify migration runs cleanly on clean database
  - Create rollback migration

---

## Phase 2: Service Layer Logic

- [ ] **Implement `CalculateFrameworkVolume(raamovereenkomst_id)` service**
  - Query summed awarded minicompetition amounts per supplier
  - Detect 70%, 85%, 95% thresholds
  - Send alerts at each threshold per REQ-RMC-007
  - Write unit tests for threshold detection
  - Test with seed data from design.md

- [ ] **Implement `ValidateAwardDoesNotExceedFrameworkMax()` service**
  - Check if award + current verbruikt would exceed maximum
  - Block award with appropriate error message if exceeded
  - Offer options: cancel, TenderNed wijziging, reduce scope, split
  - Unit test: 5 test cases covering each option

- [ ] **Implement `ValidateSupplierSelection()` service**
  - Check if selected suppliers are all active
  - Require 50+ character reason if subset selected
  - Block if inactive supplier in selection
  - Unit test: test subset selection, active/inactive mix, admin override flow

- [ ] **Implement `ValidateGunningsmodelWithinFrameworkMarges()` service**
  - Validate chosen model is in allowed set per raamovereenkomst
  - Validate weighting sums to 100%
  - Validate no component < 10% (unless 0)
  - Unit test: 6 test cases per REQ-RMC-003

- [ ] **Implement `DetectScoringDivergence()` service**
  - Calculate min/max of scores per criterion
  - Flag if divergence > 2 points
  - Set consensus_score = NULL until consensus reached
  - Unit test: test divergence detection, edge cases (all same, one outlier, etc.)

- [ ] **Implement `ScheduledJob_ExtensionReminders()` service**
  - Query frameworks with endDate 180, 120, 60 days away
  - Build notification with available extensions
  - Send email per REQ-RMC-008 scenario 8.1
  - Schedule job to run daily at 08:00 UTC
  - Unit test: mock datetime, verify correct frameworks selected

- [ ] **Implement `ScheduledJob_ContractExpiryTriggers()` service**
  - Query contracts 90 days from expiry
  - Auto-create concept minicompetition
  - Send notifications to requester and RO owner
  - Schedule job to run daily at 08:00 UTC
  - Unit test: mock datetime, verify MC creation

- [ ] **Implement volume calculation on minicompetition save/award**
  - On minicompetition status → `gegund`, recalculate `verbruiktVolumeEur` on parent raamovereenkomst
  - Trigger volume threshold notifications
  - Test with seed data

- [ ] **Implement per-supplier volume capping**
  - On award validation, check per-supplier cap if set
  - Block or warn if exceeded per REQ-RMC-007 scenario 7.5
  - Test with and without caps

---

## Phase 3: API Layer

- [ ] **Implement POST /raamovereenkomsten**
  - Accept request per design.md API contract
  - Validate input (required fields, date ranges, volume > 0)
  - Check looptijd limits based on aanbesteder sector
  - Create raamovereenkomst record with status = `concept`
  - Return 201 with location header
  - Integration test: create and verify in database

- [ ] **Implement GET /raamovereenkomsten/{id}**
  - Fetch and return full raamovereenkomst with computed fields
  - Include supplier count, consumed volume, next threshold, etc.
  - Test authorization: only creator or admin can view draft frameworks

- [ ] **Implement POST /raamovereenkomsten/{id}/promote-from-aanbesteding**
  - Input: `aanbestedingId`
  - Fetch aanbesteding, validate it is gegund
  - Copy suppliers, percelen, looptijd, volume to new RO
  - Validate looptijd per AW2012
  - Create raamovereenkomst
  - Create raamovereenkomst_leverancier rows for each winner
  - Return 201
  - Integration test: promote sample aanbesteding from context-brief seed data

- [ ] **Implement POST /raamovereenkomsten/{ro-id}/minicompetities**
  - Accept request per design.md API contract
  - Call ValidateSupplierSelection, ValidateGunningsmodelWithinFrameworkMarges
  - Create minicompetitie with status = `concept`
  - Create minicompetitie_beoordeling rows (60 for 4 criteria, 5 subs, 3 reviewers)
  - Calculate sluitingsDatum from inschrijvingsTermijnDagen
  - Return 201
  - Integration test: create and verify in database, check beoordeling rows created

- [ ] **Implement POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/publish**
  - Validate status is `concept`
  - Set publicatieDatum = now, status = `uitgenodigd`
  - For each supplier in uitgenodige_leveranciers_ids:
    - Generate unique portal token
    - Send email via openconnector with deeplink
    - Store email_sent timestamp
  - Return 200 with invitations_sent count
  - Integration test: mock email sending, verify tokens generated

- [ ] **Implement POST /raamovereenkomst/{ro-id}/minicompetities/{mc-id}/submit-bid** (supplier portal)
  - Authenticate via supplier token
  - Validate sluitingsDatum not passed (to the second)
  - Validate inschrijvings_bedrag_eur > 0
  - Store bid with ingediend_datum = server timestamp
  - Set tijdig = true if before deadline
  - Mark as `vertrouwelijk_opslag = true` (encrypted before opening)
  - Return 201
  - Integration test: submit on time, reject late, verify encryption flag

- [ ] **Implement POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/open-bids** (dual signature)
  - Require minimum 2 authorizers in request
  - Validate both are committee members
  - Check both have authorized within 5-minute window
  - Decrypt all submissions
  - Set status = `in_beoordeling`
  - Auto-generate opening PV with both signatories
  - Call docudesk to digitally sign PV
  - Return 200 with pv_generated URL
  - Integration test: test dual auth flow, single auth rejection, PV generation

- [ ] **Implement POST /minicompetities/{mc-id}/scores** (committee scoring)
  - Authenticate committee member
  - Validate bids are open (status = `in_beoordeling`)
  - For each score in request:
    - Validate score is 0-max_punten for criterion
    - Store in minicompetitie_beoordeling
    - Call DetectScoringDivergence, set flagged_for_discussion if needed
  - Return 201 with divergences_detected list
  - Integration test: score multiple criteria, detect divergence

- [ ] **Implement POST /minicompetities/{mc-id}/consensus** (record agreed score)
  - Authenticate procurement officer or chair
  - Update minicompetitie_beoordeling.consensus_score and consensus_motivering
  - Return 200
  - Integration test: update consensus after divergence

- [ ] **Implement POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/award**
  - Authenticate procurement officer
  - Call ValidateAwardDoesNotExceedFrameworkMax
  - Validate winnaar_leverancier_id is in uitgenodige_leveranciers
  - Calculate Alcatel deadline = today + 20 days
  - Set status = `gegund`, winnaar_leverancier_id, gunnings_bedrag_eur
  - Auto-generate award decision PDF via docudesk with full feedback per-bidder
  - Sign PDF
  - Send emails to all bidders with decision (personalized per-bidder feedback)
  - Recalculate verbruiktVolumeEur on RO; check thresholds
  - Return 201
  - Integration test: award, verify PDF generation, email sending, Alcatel deadline set

- [ ] **Implement blocking of contract execution during Alcatel period**
  - On POST /contracts/ or contract execution endpoint:
    - Check if contract is linked to a minicompetitie
    - If yes, check if `today < alcatel_deadline_date`
    - If so, reject with message per REQ-RMC-009
  - Integration test: try execute before deadline (blocked), after deadline (allowed)

- [ ] **Implement GET /raamovereenkomsten/{ro-id}/audit-export**
  - Query all minicompetities in date range
  - For each, collect:
    - Invitation PDF
    - All submissions (inschrijvingen)
    - Reviewer scores (per reviewer)
    - Three PVs (opening, scoring, award)
    - Award decision
  - Generate audit_log.jsonl from audit_events table
  - Create ZIP with structure per REQ-RMC-010
  - Encrypt ZIP with one-time password
  - Return 200 with download link (24h expiry)
  - Integration test: export sample data, verify ZIP structure and encryption

---

## Phase 4: UI/Frontend

### Raamovereenkomst Management

- [ ] **Create raamovereenkomst list page**
  - Table: Nummer, Titel, Status, Looptijd, MaxVolume, Verbruikt%, Einddatum, Actions
  - Filters: Status, Category, Eindatum range, Verbruikt% >= X
  - Sort: by Verbruikt% desc (highest first), by Einddatum asc
  - Search: by nummer or titel
  - Actions: View detail, Create minicompetition, Manage extensions, View audit trail
  - Link from Aanbestedingen > Raamovereenkomsten/Minicompetities per ADR-001

- [ ] **Create raamovereenkomst detail page**
  - Display all fields (RO info, suppliers, looptijd, max volume, verbruikt)
  - Show volume consumption graph: 70% / 85% / 95% / max thresholds
  - Supplier table: Leverancier, Active, Volume consumed, #Awarded, Actions (deactivate, view)
  - Minicompetitie list: Recent competitions
  - Extension management: Current status, available options, schedule
  - Audit trail link
  - Alert banner if approaching thresholds

- [ ] **Create promote-from-aanbesteding flow**
  - Step 1: Select gegunde aanbesteding
  - Step 2: Verify looptijd, show warning if klassieke sector > 4y
  - Step 3: Confirm suppliers and volume
  - Step 4: Create RO and redirect to detail page
  - Validation at each step, error messages

### Minicompetition Workflow

- [ ] **Create minicompetition creation page**
  - Form fields: Title, Description, Department, Trigger type, Budget, Gunning model, Weighting
  - Show selected suppliers (pre-selected all active per REQ-RMC-002)
  - Allow deselection with 50+ char reason input
  - Validate weighting per REQ-RMC-003
  - Submit → creates MC in `concept` status

- [ ] **Create minicompetition detail page**
  - Display all MC info: title, suppliers, gunning model, criteria, deadlines
  - Show submission count and status
  - Committee members list
  - Timeline: Invitation sent, Deadline approaching (countdown), Bids received, Opening date, Scoring in progress, Award decision, Alcatel period
  - Actions: Publish invitations, Open bids, Score submissions, Award decision, View audit trail

- [ ] **Create Q&A management interface** (procurement officer)
  - Display all questions from suppliers
  - Show supplier name and submission timestamp
  - Button: "Answer this question" → modal with:
    - Auto-anonymize supplier name
    - Text editor for answer
    - Option to extend Q&A closure/deadline if after official closure
    - Option to "Don't answer (too late)"
  - After answer, show Q&A to all suppliers

- [ ] **Create supplier portal page** (supplier login, accessed via unique token)
  - Welcome message with competition title and deadline
  - Download section: RFQ, PvE, contract template, price template, Q&A document
  - Bid submission form:
    - Price field (EUR)
    - Price breakdown (optional line items)
    - Quality answers (per criterion, text + file upload)
    - Sustainability answers (if applicable)
    - Document uploads (quotation, plan of approach, etc.)
    - Submit button
  - Confirmation page: "Bid received at [timestamp]"
  - View previously submitted bid (read-only)

- [ ] **Create bid opening interface** (committee, dual-signature gate)**
  - Permission check: user must be in beoordeling_commissie_ids
  - Show minicompetition info and bidders
  - Button: "Open Bids" → modal requiring:
    - User 1 clicks "I authorize"
    - System waits for User 2 to click "I authorize" (within 5 min)
    - On dual auth, unlock bids and show opening PV
  - Opening PV display: details, signatories, bid list
  - Redirect to scoring interface

- [ ] **Create scoring interface** (committee members)**
  - View one submission at a time or all in table format (configurable)
  - Display bid documents (price, quality answers)
  - Score input per criterion with 0-max_punten scale
  - Justification text field (required)
  - Show other committee members' scores (read-only) once all have scored
  - Flag criteria with divergence > 2 points
  - Mark as "Scored" when all criteria scored

- [ ] **Create consensus discussion interface**
  - List all criteria with divergences
  - Show individual scores and justifications
  - Consensus score input field
  - Consensus rationale text field
  - Submit consensus
  - Log to audit trail

- [ ] **Create award decision page**
  - Display final ranking (1st, 2nd, 3rd scores)
  - Show per-bidder feedback (anonymized preview)
  - Button: "Generate Award Decision"
    - Auto-generates PDF with all sections per REQ-RMC-009
    - Shows preview of PDF
    - Button: "Send to Bidders"
    - Confirmation: Decision sent, Alcatel deadline set to {date}
  - Countdown timer to Alcatel deadline
  - Blocking notice if trying to execute contract before deadline

### Extension Management

- [ ] **Create extension request form**
  - Available options dropdown (from verlengingsopties)
  - Motivation text field
  - Optional: Price indexation toggle + percentage input + CPI source
  - Optional: Updated KPI appendix file upload
  - Submit → creates raamovereenkomst_verlenging with status = `in_voorbereiding`
  - Show timeline: created, submitted, approved/rejected, new end date

- [ ] **Create extension approval workflow** (director/procurement manager)
  - List pending extensions
  - View request, motivation, proposed modifications
  - Actions: Approve, Reject (with reason), Return for revision
  - On approve: set status = `akkoord`, notify all suppliers, create TenderNed notice if required

### Audit & Compliance

- [ ] **Create audit export interface**
  - Framework dropdown
  - Date range pickers (from, to)
  - Button: "Generate Export"
  - Show progress and download link when ready
  - Link includes one-time password for decryption

- [ ] **Create audit log viewer**
  - Per-minicompetition: chronological log of all mutations
  - Show: Event type, Field changed, Old value, New value, Changed by, Timestamp, Reason
  - Filter: Event type, date range, user
  - Export to CSV

---

## Phase 5: Integration with Other Capabilities

- [ ] **Integrate with aanbesteding-werkproces**
  - Add "Promote to Raamovereenkomst" action to gegunde aanbesteding detail page
  - Link from RO back to source aanbesteding
  - Test promotion flow end-to-end

- [ ] **Integrate with tenderned-publicatie-adapter**
  - On framework creation: optionally publish via TenderNed (or just link to existing notice)
  - On wijzigingsaankondiging (volume exceeded): publish via adapter
  - On extension approval: publish via adapter if required
  - Test integration: mock TenderNed responses

- [ ] **Integrate with mvi-sroi-aanbesteding**
  - On minicompetition creation: pre-populate duurzaamheidscriteria from RO's MVI defaults (if set)
  - Allow procurement officer to override
  - Test: create MC with and without MVI defaults

- [ ] **Integrate with openconnector**
  - Configure SMTP or API source for email delivery
  - Send invitations per REQ-RMC-004
  - Send decision notifications per REQ-RMC-009
  - Send Alcatel reminders
  - Test: verify emails sent with correct content, recipients, links

- [ ] **Integrate with docudesk**
  - Configure PV templates (opening, scoring, award)
  - Generate and digitally sign PVs
  - Store signed PDFs in openregister
  - Test: generate sample PVs, verify signatures

- [ ] **Integrate with mydash**
  - Create KPI widgets:
    - "Raamovereenkomsten in final year" (count by status)
    - "Framework volume consumption" (gauge per RO)
    - "Average minicompetition duration" (days from trigger to award)
    - "Average bids per competition" (count)
    - "MKB share in frameworks" (%)
  - Test: widget data queries with sample data

- [ ] **Integrate with scheduled job runner (n8n / pipelinq)**
  - Schedule ScheduledJob_ExtensionReminders to run daily at 08:00 UTC
  - Schedule ScheduledJob_ContractExpiryTriggers to run daily at 08:00 UTC
  - Set up error alerting
  - Test: mock datetime, verify jobs trigger at correct times

---

## Phase 6: Testing & QA

### Unit Tests

- [ ] **Service layer tests**
  - CalculateFrameworkVolume: 5 tests (no subs, 1 sub, multiple subs, threshold detection)
  - ValidateAwardDoesNotExceedFrameworkMax: 5 tests
  - ValidateSupplierSelection: 5 tests
  - ValidateGunningsmodelWithinFrameworkMarges: 6 tests
  - DetectScoringDivergence: 4 tests
  - Each test with setup, assertion, teardown
  - Target: 90% code coverage of service layer

### Integration Tests

- [ ] **API endpoint tests**
  - POST /raamovereenkomsten: 3 tests (valid, invalid, looptijd exceeded)
  - POST /raamovereenkomsten/{id}/promote-from-aanbesteding: 3 tests
  - POST /raamovereenkomsten/{ro-id}/minicompetities: 5 tests
  - POST /minicompetities/{mc-id}/submit-bid: 3 tests (on time, late, invalid bid)
  - POST /minicompetities/{mc-id}/open-bids: 3 tests (dual auth, single auth, timeout)
  - POST /minicompetities/{mc-id}/scores: 3 tests (valid score, divergence, consensus)
  - POST /raamovereenkomsten/{ro-id}/minicompetities/{mc-id}/award: 3 tests
  - Contract execution block: 2 tests (during Alcatel, after Alcatel)
  - GET /raamovereenkomsten/{ro-id}/audit-export: 2 tests (valid export, date range)
  - Each test uses seed data, verifies database state, HTTP status

### Scenario Tests

- [ ] **End-to-end flow: unplanned minicompetition**
  - Create RO from gegunde aanbesteding
  - Create minicompetition manually (trigger: new need, 3 suppliers, EMVI model, 4 criteria)
  - Publish invitations
  - 2 suppliers submit bids on time, 1 late (verify rejection)
  - Committee opens bids (dual sig)
  - Committee scores: detect divergence, reach consensus
  - Procurement officer awards to winner
  - Verify Alcatel deadline set
  - Try contract execution (blocked)
  - Verify audit export includes all documents
  - Test duration: ~2 hours for manual execution; automated test should complete in <30s

- [ ] **End-to-end flow: contract expiry trigger**
  - Create contract linked to minicompetition
  - Set contract end date to 90 days from today
  - Run scheduled job
  - Verify concept MC auto-created in RO
  - Verify requester notified
  - Complete competition through award
  - Test duration: automated test with mocked datetime

- [ ] **Threshold alert flow**
  - Create RO with EUR 2M max
  - Award minicompetitions totaling EUR 1.4M (70%)
  - Verify banner alert shown
  - Award to EUR 1.7M (85%)
  - Verify email alert sent
  - Award to EUR 1.95M (95%)
  - Verify critical email
  - Try award EUR 100k (blocked)
  - Test duration: <5 minutes

- [ ] **Extension flow**
  - Create RO with end date 180 days away, 1 extension option (12m, non-exclusive)
  - Run 180-day reminder job
  - Verify owner notified
  - Create extension with CPI indexation 2.3%
  - Approve extension
  - Verify new end date set
  - Try create second extension (RO should reject due to max_aantal_keer)
  - Test duration: ~1 hour

### Browser/UI Tests (Functional)

- [ ] **Raamovereenkomst list & detail**
  - Load list page, apply filters, search
  - Click on item, view detail
  - Verify all fields display correctly
  - Verify graphs and alerts render

- [ ] **Minicompetition creation**
  - Create MC with deselection of 1 supplier
  - Verify 50-char reason enforced
  - Attempt invalid weighting (fails)
  - Correct and save (succeeds)

- [ ] **Supplier portal**
  - Access portal via unique token
  - Download documents
  - Submit bid with documents
  - Verify confirmation shown
  - View submitted bid (read-only)
  - Access with expired token (verify failure)

- [ ] **Committee scoring**
  - Open bids (dual auth flow)
  - Score multiple criteria
  - Verify divergence flag
  - Reach consensus
  - View final ranking

- [ ] **Award & Alcatel**
  - Generate award decision
  - View PDF preview
  - Send to bidders
  - Verify decision emails received
  - Wait out Alcatel period (or mock time)
  - Execute contract (verify success after period)

### Performance Tests

- [ ] **Load test: simultaneous submissions**
  - Minicompetition with 5-minute deadline
  - 30 concurrent supplier submissions in final minute
  - Verify all accepted on time, no timeouts
  - Measure submission endpoint p95 latency

- [ ] **Load test: audit export**
  - Framework with 100 minicompetitions
  - Generate audit export (ZIP)
  - Target: complete in < 30 seconds

- [ ] **Load test: scoring calculation**
  - 100 bids, 10 criteria, weighted score calculation
  - Target: complete in < 5 seconds

### Security Tests

- [ ] **Authorization: non-committee member tries to score**
  - User not in beoordeling_commissie_ids calls POST /scores
  - Verify 403 Forbidden response
  - Verify audit log of attempt

- [ ] **Authorization: supplier tries to see other supplier's bid**
  - Supplier 1 accesses supplier portal for Supplier 2's bid submission
  - Verify access denied

- [ ] **Authorization: procurement officer tries to open bids alone**
  - POST /open-bids with only 1 authorizer
  - Verify rejection, require 2nd signer

- [ ] **Authorization: contractor tries to view bids before opening**
  - Committee member views bids before dual-signature ceremony
  - Verify access blocked, attempt logged

- [ ] **Late submission rejection**
  - Submit bid 1 second after deadline
  - Verify immediate rejection, no appeals

---

## Phase 7: Documentation & Deployment

- [ ] **Write API documentation**
  - OpenAPI / Swagger spec for all endpoints
  - Request/response examples
  - Error codes and messages
  - Published to /docs endpoint

- [ ] **Write user guide (Dutch)**
  - Procurement officer perspective: create RO, minicompetition, award
  - Committee member perspective: score, consensus, PV
  - Supplier perspective: portal access, submission, feedback
  - Administrator perspective: dossier management, audit trail
  - Screenshots for each workflow

- [ ] **Write deployment guide**
  - Database migrations
  - Environment variables (email config, docudesk, TenderNed endpoints)
  - Cron job setup (scheduled tasks)
  - Integration setup (openconnector, docudesk, other adapters)
  - Rollback procedures

- [ ] **Create release notes**
  - Feature summary
  - Known issues
  - Breaking changes (if any)
  - Upgrade path for existing tenants (if data migration needed)

- [ ] **Deploy to staging**
  - Run all tests on staging
  - User acceptance testing (UAT) with real procurement officers
  - Fix issues found in UAT
  - Performance testing on production-like data volume

- [ ] **Deploy to production**
  - Coordinate rollout with procurement team
  - Monitor for errors, slow performance, bugs
  - Have rollback plan ready
  - Post-launch: weekly check-ins for first month

---

## Phase 8: Post-Launch Monitoring & Improvement

- [ ] **Monitor key metrics (first 30 days)**
  - API response times (p50, p95, p99)
  - Error rates (5xx, validation failures)
  - Minicompetition creation rate (how many active)
  - User adoption (login frequency)
  - Support tickets (issues reported)
  - Alert if any metric degrades

- [ ] **Collect user feedback**
  - Send feedback form to procurement officers and committee members
  - Identify friction points (where users abandon, where they struggle)
  - Plan improvements for v1.1

- [ ] **Performance optimization (if needed)**
  - Profile slow queries (audit export, volume calculation)
  - Add caching if applicable (framework list, supplier list)
  - Optimize date range queries

- [ ] **Iterate based on feedback**
  - v1.1: UI/UX improvements, missing convenience features
  - v1.2: Advanced features (e.g., automated scoring templates, AI price-reasonableness checks)

---

## Acceptance Criteria by Phase

**Phase 1 Acceptance**: All 8 schemas created, migrations tested, sample data inserted successfully

**Phase 2 Acceptance**: All services tested with 8+ unit tests each, passing

**Phase 3 Acceptance**: All 13 API endpoints tested with integration tests, verified in database

**Phase 4 Acceptance**: All UI pages render, forms validate, links navigate correctly; browser testing passed

**Phase 5 Acceptance**: Integration tests with mock other-capabilities; end-to-end flows work

**Phase 6 Acceptance**: All scenario, unit, integration tests pass; >85% code coverage; security tests pass

**Phase 7 Acceptance**: Documentation complete; staging deployment successful; UAT sign-off; production release ready

**Phase 8 Acceptance**: Live metrics monitored, feedback collected, no critical bugs; post-launch review completed

---

## Dependencies & Prerequisites

- **Database**: PostgreSQL 13+ with JSON support
- **Backend framework**: (purchaseq stack, e.g., PHP Symfony or Python Flask)
- **Frontend**: (purchaseq stack, e.g., Vue.js, React)
- **docudesk integration**: API access for PV generation and digital signing
- **openconnector integration**: SMTP or secure email API configured
- **tenderned-publicatie-adapter**: Published and available as dependency
- **mvi-sroi-aanbesteding**: Published and available as dependency
- **Scheduled job runner**: n8n or pipelinq deployed and ready for new jobs

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Looptijd validation not enforced → non-compliant RO created | Database CHECK constraint on looptijdJaren; unit tests for both klassieke (<=4) and speciale (<=8); manual code review |
| Late bids accepted → fairness compromised | Immutable timestamp on submission, server-side deadline check (not client-side), timezone-aware deadline handling, test with concurrent submissions at deadline second |
| Scoring divergence not detected → biased award | Automated divergence flag (>2 points) in service layer; consensus requirement before final score; PV documents divergence discussion |
| Framework max volume exceeded → legal non-compliance | Database constraint on award creation; ValidateAwardDoesNotExceedFrameworkMax service blocks; test with thresholds |
| Alcatel period violated → bidder can sue | Auto-block contract execution until after deadline; countdown timer in UI; audit log of execution attempts during period; test before/after period boundaries |
| PV signature forged or modified → audit invalid | Immutable database flag on PV; digital signature via docudesk (third-party trust); checksums in audit export; test PV modification rejection |
| Supplier portal token leaked → competitor sees other bids | Single-use token or 24h expiry; HTTPS only; no token in email body (link only); log all portal access; test token reuse rejection |
| Scheduled jobs don't run → reminders missed, contracts not triggered | Monitor job execution logs; alert if job fails; manual fallback procedures documented; test jobs with mocked datetime |
| Volume calculation slow on large frameworks | Database indexes on (raamovereenkomst_id, status); aggregate cache updated on award; test with 1000+ minicompetitions |

---

## Success Criteria (Go-Live)

- [ ] Zero critical security bugs (no unauthorized access, signature forgery, data exposure)
- [ ] 100% of minicompetitions generate three signed PVs
- [ ] 0 framework-maximum overruns detected (volume constraint enforced)
- [ ] >95% API test pass rate
- [ ] >80% user acceptance testing pass rate (procurement officer review)
- [ ] Documentation complete (API, user guide, deployment guide)
- [ ] Performance: minicompetition creation <2s, audit export <30s, scoring calc <5s
- [ ] Adoption: >50% of procurement officers active within 2 weeks of launch
- [ ] Support: <5 critical bugs reported in first 30 days
