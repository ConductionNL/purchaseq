---
status: proposal
created: 2026-05-23
---

# Proposal: Raamovereenkomst en Minicompetitie

## Executive Summary

The Raamovereenkomst & Minicompetitie capability delivers the complete lifecycle and workflow for framework agreements (raamovereenkomsten) and the minicompetitie mechanism for awarding specific contracts within those agreements.

Raamovereenkomsten are the dominant procurement mechanism in the Netherlands for repetitive purchasing with multiple suppliers, used by municipalities for temp staffing, central procurement organizations for IT hardware and licenses, and regional partnerships for services like youth care and green maintenance. This capability covers all three modalities (single supplier, multi-supplier with fixed conditions, multi-supplier with minicompetition required) but focuses heavily on the third due to its complexity.

## Business Value

- **Legal compliance**: Implements AW2012 art 2.140-2.144 and EU directive 2014/24/EU art 33 for framework agreements
- **Risk reduction**: Enforces proper documentation, dual-authorization, and audit trails per article 4.13 (7-year retention)
- **Volume control**: Real-time monitoring against maximum volumes with 70%, 85%, 95% thresholds and blocking at limit
- **Process integrity**: Minicompetition workflow with formal opening protocols, multi-reviewer scoring, and consensus mechanisms
- **Extended value**: Manages framework agreement extensions with automatic reminders and compliance checks

## Features

| Feature | Demand | Impact |
|---------|--------|--------|
| Create raamovereenkomst from awarded tender | High | Bridges aanbesteding-werkproces to framework agreement lifecycle |
| Minicompetition trigger & supplier selection | High | Supports manual requests, auto-trigger on contract end, budget-driven starts |
| Gunning model per minicompetition | High | Allows per-competition model choice (lowest price, EMVI, quality-within-cap) |
| Invitation publication & Q&A channel | High | Securely distributes invitations and anonymizes vendor questions |
| Receipt closure & dual-authorization opening | High | Prevents unauthorized score viewing before formal opening ceremony |
| Multi-reviewer scoring with consensus | High | Minimizes scoring bias, enforces discussion on divergences >2 points |
| Real-time volume monitoring | High | Blocks gunnings that would exceed framework maximum without TenderNed notice |
| Extension management with auto-reminders | High | Manages renewals at 180, 120, 60-day marks with compliance checks |
| Award decision with Alcatel termination | High | Generates decision document with 20-day complaint period and legal tracking |
| Complete audit trail & process records | High | Creates three PVs per competition (opening, scoring, award), digitally signed |

## User Stories

### Story 1: Create Raamovereenkomst from Awarded Tender
**As an** inkoper (procurement officer)
**I want to** promote a gegunde aanbesteding to a raamovereenkomst
**So that** I can manage the framework agreement lifecycle and track its supplier portfolio

**Acceptance Criteria:**
- GIVEN a gegunde aanbesteding with multiple percelen and winnaars
  WHEN I click "Promote naar raamovereenkomst"
  THEN the system creates a raamovereenkomst with all suppliers, percelen, looptijd, and maximum volume copied
- GIVEN a framework agreement exceeding the 4-year maximum (klassieke sector)
  WHEN promotion is attempted
  THEN the system rejects it with clear guidance on AW2012 art 2.140 limits
- GIVEN a special-sector aanbesteder
  WHEN promotion occurs
  THEN the system allows the 8-year maximum per AW2012 art 3.55

### Story 2: Trigger Minicompetition and Select Suppliers
**As an** aanvragende afdelingsmanager (business unit requesting competition)
**I want to** start a minicompetition with appropriate supplier selection
**So that** the right suppliers are invited and accountability is clear

**Acceptance Criteria:**
- GIVEN a framework agreement with 8 active suppliers in a perceel
  WHEN I start a new minicompetition
  THEN all 8 are pre-selected; deselecting any requires a 50+ character reason
- GIVEN a contract ending 2026-12-31
  WHEN the scheduled trigger runs 90 days prior
  THEN a concept minicompetition is auto-created with owner=original contractor
- GIVEN suppliers marked inactive (bankruptcy, terminated)
  WHEN minicompetition selection runs
  THEN inactive suppliers are hidden and cannot be re-added without admin override

### Story 3: Configure Gunning Model with Validation
**As an** inkoper
**I want to** set the pricing model and weighting for a specific minicompetition
**So that** the competition rules are clear and proportional

**Acceptance Criteria:**
- GIVEN a framework allowing EMVI models only
  WHEN I attempt to set "laagste_prijs"
  THEN the system rejects with "Gunningsmodel not allowed within this framework"
- GIVEN EMVI with price 60%, quality 35%, sustainability 4%
  WHEN I attempt to save
  THEN validation fails (sustainability <10%) with suggestion to raise it or remove it
- GIVEN EMVI with price 50%, quality 30%, sustainability 20% (totaling 100%)
  WHEN I save
  THEN validation succeeds

### Story 4: Publish Invitations and Handle Q&A
**As an** inkoper
**I want to** send formal invitations to selected suppliers and answer their questions
**So that** all suppliers have equal information and the process is transparent

**Acceptance Criteria:**
- GIVEN 5 selected suppliers
  WHEN I click "Send invitations"
  THEN each receives a unique email with portal deeplink, all documents downloadable, and I see read-receipts
- GIVEN supplier 2 asks a PvE question
  WHEN I click "Answer"
  THEN the question is anonymized, all 5 suppliers receive the Q&A pair, and it's added to the public document set
- GIVEN a late question after Q&A closure
  WHEN submitted
  THEN I can choose "Don't answer (too late)" or "Answer anyway (must provide reason and extend deadline)"

### Story 5: Close Receipt and Open Bids with Dual Signature
**As a** commissielid (review committee member)
**I want to** formally close the submission window and open bids with a colleague
**So that** the integrity and fairness of the process is maintained

**Acceptance Criteria:**
- GIVEN a minicompetition with deadline 2026-06-01 15:00:00
  WHEN submission arrives at 15:00:01
  THEN it's rejected with server timestamp and "Deadline exceeded" message
- GIVEN unopened bids before the opening ceremony
  WHEN I attempt to view them
  THEN the system blocks access and logs the attempt; opening requires dual signature
- GIVEN opening ceremony commenced with 2 committee members authorized
  WHEN both approve simultaneously
  THEN all bids unlock, a PV of opening auto-generates with bids/amounts/times, both digitally sign it

### Story 6: Score Submissions with Multi-Reviewer Consensus
**As a** commissielid
**I want to** score submissions independently and reach consensus on divergent scores
**So that** scoring is fair and defensible

**Acceptance Criteria:**
- GIVEN 4 quality criteria, 5 submissions, 3 reviewers
  WHEN scoring begins
  THEN 60 scoring rows are created, each reviewer sees only their own rows
- GIVEN reviewer A scores criterion 1 as 8, B as 4, C as 7 (divergence >2)
  WHEN the system detects this
  THEN the row is flagged, a consensus action created, final score stays null until consensus via dedicated UI
- GIVEN all criteria scored and consensus reached
  WHEN "Calculate final scores" is clicked
  THEN weighted scores per submission are calculated, ranking established, and a scoring PV is auto-generated

### Story 7: Monitor Volume Against Framework Maximum
**As a** raamovereenkomst-eigenaar (framework owner)
**I want to** see real-time volume consumption and receive warnings
**So that** I don't accidentally exceed the framework maximum

**Acceptance Criteria:**
- GIVEN a framework with max EUR 2M and EUR 1.7M (85%) consumed
  WHEN a new minicompetition is started
  THEN I see a banner "85% of maximum consumed; average capacity for X more competitions"
- GIVEN EUR 1.95M consumed
  WHEN a EUR 100k award is prepared
  THEN the system blocks it with "Would exceed maximum; requires TenderNed notice or competition cancellation"
- GIVEN per-supplier caps (e.g., supplier X max EUR 500k)
  WHEN an award to X would exceed that cap
  THEN the system blocks and suggests re-awarding or splitting

### Story 8: Manage Extension with Automatic Reminders
**As a** raamovereenkomst-eigenaar
**I want to** receive timely reminders and manage extensions
**So that** the framework doesn't lapse unexpectedly

**Acceptance Criteria:**
- GIVEN framework ending 2026-12-31 with extension option (12 months, non-exclusive, max 2x)
  WHEN the scheduled job runs on 2026-07-04 (180 days prior)
  THEN a reminder notification is sent to the owner with extension creation option
- GIVEN an extension in preparation with 2.3% price indexation (CPI-linked)
  WHEN I add it with updated KPI appendix
  THEN the system approves (proportional adjustment under AW2012 art 2.163d) and logs the rationale
- GIVEN extension options already fully used
  WHEN extension is attempted
  THEN the system rejects with "Must start new procurement"

### Story 9: Generate Award Decision with Alcatel Protection
**As a** commissie-voorzitter (committee chair)
**I want to** generate a formal award decision
**So that** the process is legally defensible and losers have grounds to protest

**Acceptance Criteria:**
- GIVEN a minicompetition with ranking [winner, 2nd, 3rd]
  WHEN "Generate award decision" is clicked
  THEN a PDF is created with: (a) anonymous ranking for losers, (b) full score explanation for each loser's submission, (c) rationale for winner, (d) Alcatel clause with dispute deadline
- GIVEN the decision sent to all bidders
  WHEN the 20-day Alcatel period expires
  THEN the system unblocks final contract execution
- GIVEN a loser files a dispute during the period
  WHEN registered in purchaseq
  THEN the gunning moves to `bezwaar_ingesteld`, execution is halted, and legal workflow is triggered

### Story 10: Create Complete Audit Trail and Process Records
**As a** compliance / auditeur
**I want to** retrieve all competition documents and decision trails
**So that** I can verify the process was fair and compliant

**Acceptance Criteria:**
- GIVEN a minicompetition with opening, scoring, and award phases
  WHEN the competition completes
  THEN three digitally-signed PVs are auto-generated (opening, scoring, award) with attendees, timestamps, content, and PDF signature
- GIVEN an audit query for "all competitions in framework RO-2024-IT-001 in Q2 2025"
  WHEN the export runs
  THEN a ZIP is generated with per-competition: invitations, submissions, reviewer scores, three PVs, award decision, and mutation audit log
- GIVEN a competition record older than 7 years
  WHEN the purge job runs
  THEN the record is marked `bewaartermijn_verlopen` and requires explicit admin + 2FA to delete

## Stakeholders

| Stakeholder | Role | Responsibilities |
|---|---|---|
| Inkoper / contractmanager | Daily user | Executes minicompetitions, monitors volume, plans extensions |
| Categoriemanager / portfolio-inkoper | Portfolio oversight | Monitors framework portfolio for category compliance, strategic alignment |
| Aanvragende afdelingsmanager | Business unit | Triggers competitions, defines quality criteria, serves on scoring committee |
| Beoordelingscommissie-leden | Scoring | Score submissions independently, reach consensus, sign PVs |
| Contractmanager / leveranciersmanager | Supplier oversight | Monitors awarded competitions, supplier performance, renewal input |
| Juridisch adviseur | Legal counsel | Reviews edge cases, handles disputes within Alcatel period |
| Compliance / interne controle | Audit | Uses audit trail for compliance verification |
| Bestuur / college / directie | Executive | Receives portfolio and large-competition reports |
| Leveranciers in raamovereenkomst | Indirect / external | Receive invitations, submit bids, receive award decisions |

## Customer Journey

### Journey 1: Year-start Framework Planning
**Trigger:** Budget cycle begins (January)
**Actors:** Categoriemanager, Inkoper
**Steps:**
1. Categoriemanager reviews framework portfolio for expiring agreements (60+ days)
2. For each approaching end, Inkoper begins extension evaluation or replacement tender
3. If extension: start extension workflow with price/KPI update
4. If replacement: hand off to aanbesteding-werkproces for new tender
5. System sends reminders at 180, 120, 60 days to decision-owners

**Outcome:** Extensions approved or new tenders initiated; portfolio does not lapse

### Journey 2: Unplanned Minicompetition from Business Request
**Trigger:** Afdeling requests new service (e.g., additional Java developers on existing staffing framework)
**Actors:** Aanvragende afdelingsmanager, Inkoper, Commissie
**Steps:**
1. Afdeling creates new minicompetition with description and budget
2. Inkoper reviews, selects suppliers (defaults to all active, can deselect with reason)
3. Inkoper configures gunning model and weighting (e.g., EMVI 50% price / 30% quality / 20% sustainability)
4. Inkoper publishes invitations to selected suppliers
5. Suppliers submit bids by deadline; system rejects late arrivals automatically
6. Commissie (min 3 members) reviews and opens bids (dual-sign gate)
7. Commissie independently scores each submission per criterion
8. System flags divergences >2 points; Commissie holds consensus meeting
9. System calculates final scores and ranking
10. Inkoper generates award decision, publishes to all bidders
11. Alcatel 20-day dispute period passes
12. Inkoper executes contract with winner

**Outcome:** Service awarded fairly and within framework terms; audit trail complete

### Journey 3: Contract Auto-End Triggers Renewal Minicompetition
**Trigger:** Underlying contract is 90 days from expiry
**Actors:** System (scheduled), Inkoper, Business unit
**Steps:**
1. Scheduled job detects expired contract on framework
2. System creates concept minicompetition titled "Renewal of contract X"
3. Inkoper is notified; business unit owner is pre-assigned
4. Inkoper reviews and either proceeds (most suppliers) or deselects inactives/underperformers
5. Remainder of journey follows Story 2 (Trigger Minicompetition)

**Outcome:** Framework maintained without contract gap; no manual reminder needed

### Journey 4: Volume Limit Approaches and Blocks New Award
**Trigger:** Existing framework at 85% of EUR 2M maximum
**Actors:** Inkoper, RO-owner, TenderNed adapter (optional)
**Steps:**
1. New minicompetition is started for urgent need
2. Inkoper receives banner: "Framework at 85%; estimated room for 2 more competitions at historical average"
3. Minicompetition completes and reaches award decision (EUR 500k)
4. Inkoper attempts to execute award
5. System blocks: "EUR 1.95M + EUR 500k = EUR 2.45M exceeds maximum of EUR 2M"
6. Inkoper chooses option A: Cancel this minicompetition
   - OR Option B: File TenderNed wijzigingsaankondiging to increase framework maximum
   - OR Option C: Deselect lower-performing bidders and re-bid with lower budget
7. If Option B chosen: TenderNed adapter publishes notice; framework maximum updated after publication period
8. Award then executes

**Outcome:** Framework maximum is respected; legal transparency maintained via TenderNed

### Journey 5: Dispute During Alcatel Period
**Trigger:** Losing bidder files protest (e.g., via brief van gegriefdheid to municipality)
**Actors:** Juridisch adviseur, Commissie, Inkoper
**Steps:**
1. Juridisch adviseur logs dispute in purchaseq under the minicompetition
2. System marks minicompetition as `bezwaar_ingesteld`
3. Contract execution is auto-blocked
4. System notifies Commissie and Inkoper; legal workflow begins
5. Juridisch adviseur reviews scoring, PVs, and documentation
6. If disputed party is right: system allows re-scoring or re-award
7. If dispute is rejected: legal advice concludes; contract execution resumes

**Outcome:** Process integrity is maintained; disputes are formally tracked and resolved

## Pain Points Addressed

1. **Compliance drift**: Automated enforcement of AW2012 and EU directive limits (4/8-year looptijd, minicompetition rules)
2. **Scoring bias**: Consensus mechanism and divergence flagging reduce unfair scoring; dual-signature on opening ensures integrity
3. **Volume overrun**: Real-time monitoring prevents accidental exceed of framework maximum
4. **Supplier exclusion**: Clear rationale required for deselecting active suppliers; inactive suppliers auto-filtered
5. **Lost documentation**: Auto-generated, digitally-signed PVs and complete audit trail per AW2012 art 4.13
6. **Extension forgetting**: Scheduled reminders 180/120/60 days prevent lapsed frameworks
7. **Alcatel breaches**: Automatic hold on contract execution during 20-day dispute period; legal workflow tracking

## Design Constraints

- **Information Architecture**: Per ADR-001, this capability is a **sub-page** under *Aanbestedingen > Raamovereenkomsten/Minicompetities*, not a top-level menu
- **Entity reuse**: All entities (raamovereenkomst, minicompetitie, etc.) defined in this spec; no entities to invent
- **Integration**: Bridges **aanbesteding-werkproces** (source of RO creation), **tenderned-publicatie-adapter** (for wijzigingsaankondigingen and publication), **mvi-sroi-aanbesteding** (for sustainability criteria defaults), **openconnector** (for invitation email), **docudesk** (for PV generation), **mydash** (KPI tiles)
- **Legal**: Every minicompetition MUST generate three digitally-signed PVs; audit trail MUST cover all mutations for 7-year retention

## Success Metrics

- **Volume control**: Zero undetected framework-maximum overruns
- **Compliance**: 100% of minicompetitions generate three signed PVs
- **Timelines**: 80% of framework extensions triggered via reminder (not manual request)
- **Audit readiness**: 100% of competition data retrievable in 7-year export
- **User adoption**: >70% of eligible framework agreements use minicompetition feature within first 12 months
