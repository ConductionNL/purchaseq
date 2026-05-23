---
status: specs
created: 2026-05-23
---

# Specifications: Raamovereenkomst en Minicompetitie

## Functional Requirements

### REQ-RMC-001: Create Raamovereenkomst from Awarded Tender

**Description:** The system MUST allow promotion of a gegunde aanbesteding (status `gegund` in werkproces) to a `raamovereenkomst` object, copying perceelindeling, gegunde leveranciers, looptijd, maximumvolume, and the original gunning framework.

**Acceptance Criteria:**

**Scenario 1.1: Promote gegunde aanbesteding with multiple percelen to framework**
- GIVEN a gegunde aanbesteding with:
  - 3 percelen (A: Hardware, B: Software Licenses, C: Support Services)
  - 5 winnaars (Perceel A: 1, Perceel B: 2, Perceel C: 2)
  - looptijd of 3 years
  - estimated max volume EUR 5.000.000
  WHEN the user clicks "Promote to Raamovereenkomst"
  THEN:
  - The system creates 1 `raamovereenkomst` record with:
    - `status = 'concept'` (not yet published)
    - `modaliteit = 'meerdere_minicompetitie'` (system detects from aanbesteding structure)
    - `maximumVolumeEur = 5000000`
    - `looptijdJaren = 3`
  - 5 `raamovereenkomst_leverancier` rows are created, one per winner:
    - Each row references the winning supplier from the aanbesteding
    - `perceel` is set to the winner's perceel (A, B, or C)
    - `actief = true`
  - The user is redirected to the new raamovereenkomst detail page with status message: "Framework created from aanbesteding {number}. Next: verify suppliers, publish."

**Scenario 1.2: Reject promotion if looptijd exceeds klassieke sector limit**
- GIVEN a gegunde aanbesteding for a klassieke sector aanbesteder with:
  - looptijd of 6 years (exceeds AW2012 art 2.140 limit of 4 years)
  WHEN the user attempts promotion
  THEN:
  - The system rejects the action with error:
    "**Looptijd overschrijdt limiet** (6 jaar): AW2012 art 2.140 geeft maximaal 4 jaren voor klassieke sectoren. U kunt maximaal 4 jaar instellen, of u kunt een uitzonderingsgrond aangeven voor verlening tot maximaal 8 jaren."
  - A form appears allowing the user to either:
    - Option A: Reduce looptijd to 4 years
    - Option B: Provide written justification for exception (legal counsel reviews before approval)

**Scenario 1.3: Allow 8-year looptijd for speciale-sector aanbesteder**
- GIVEN a gegunde aanbesteding for a speciale-sector aanbesteder (e.g., water board, port authority) with:
  - looptijd of 8 years (permitted under AW2012 art 3.55)
  WHEN promotion is attempted
  THEN:
  - The system accepts it without objection
  - Raamovereenkomst is created with `looptijdJaren = 8`
  - The user sees an informational badge: "8-year framework (special sector exemption per AW2012 art 3.55)"

**Scenario 1.4: Copy gunning framework and default gunning models**
- GIVEN a gegunde aanbesteding that specified allowed gunning models for minicompetitions (e.g., "only EMVI and lowest-price")
  WHEN promotion occurs
  THEN:
  - The system stores the set of allowed gunningsmodels on the raamovereenkomst
  - Every minicompetition created within this framework will default to this constraint (REQ-RMC-003)

---

### REQ-RMC-002: Minicompetition Trigger and Supplier Selection

**Description:** The system MUST support triggering a minicompetition via (a) manual request by business unit, (b) automatic trigger 90 days before contract end, or (c) budget request from coupled planning. Supplier selection MUST default to all active suppliers in the relevant perceel and MUST require justification if a subset is chosen.

**Acceptance Criteria:**

**Scenario 2.1: Manual trigger with all active suppliers pre-selected**
- GIVEN a raamovereenkomst for "IT-inhuur Java developers" with 8 active suppliers in perceel "Java"
  WHEN a business unit manager clicks "New minicompetition" and fills in:
    - Title: "Java Team - Migration Project Q3 2024"
    - Description: "..."
    - Budget: EUR 450.000
  AND clicks "Next"
  THEN:
  - The system pre-selects all 8 active suppliers
  - The user sees: "8 of 8 suppliers selected from this framework"
  - De-selecting any supplier requires a text field with min 50 characters: "Reason for excluding {supplier name}:"
  - Once deselection is attempted without reason, an error appears: "Please provide reason (min 50 characters) for excluding this supplier."

**Scenario 2.2: No inactive suppliers can be pre-selected or manually added**
- GIVEN a raamovereenkomst where one supplier (e.g., "TechStaff BV") is marked `actief = false` with `wegvalDatum = 2024-04-30` (bankruptcy)
  WHEN minicompetition selection occurs
  THEN:
  - The inactive supplier does NOT appear in the supplier list
  - If a user manually tries to type the supplier name, autocomplete returns nothing
  - If the supplier ID is directly submitted in an API call, the validation rejects: "Supplier {kvk} is no longer active in this framework."
  - Admin override option exists but requires admin user + explicit justification logged to audit trail

**Scenario 2.3: Automatic trigger on contract expiry (90 days prior)**
- GIVEN an active contract linked to minicompetition MC-2024-001 with `eindDatum = 2024-12-31`
  WHEN the scheduled job ScheduledJob_ContractExpiryTriggers() runs on 2024-10-02 (90 days prior)
  THEN:
  - A concept minicompetition is auto-created within the same raamovereenkomst with:
    - `titel = "Renewal of [Original Contract Title]"`
    - `triggerType = 'aflopend_contract'`
    - `aanvragende_user_id = [original contract requester]`
    - `status = 'concept'` (not yet published; awaiting review)
  - The original requester (business unit) receives a notification: "Contract {name} is expiring on 2024-12-31. A renewal minicompetition has been prepared for your review."
  - The raamovereenkomst owner receives a notification: "New renewal competition prepared: MC-[number]"

**Scenario 2.4: Automatic trigger on budget request (from coupled planning)**
- GIVEN purchaseq is integrated with a budget/planning app that sends a POST to /minicompetities/trigger-from-budget with:
  ```json
  {
    "raamovereenkomst_id": "ro-uuid",
    "estimated_amount": 600000,
    "description": "Q4 staffing allocation per annual plan",
    "requester_user_id": "budget-user-uuid"
  }
  ```
  WHEN the endpoint is called with valid data
  THEN:
  - A minicompetition is created with pre-populated budget and description
  - The requester is notified: "Minicompetition created from your budget request"
  - The minicompetition remains in `concept` status, awaiting completion of supplier selection and model setup

---

### REQ-RMC-003: Gunning Model Per Minicompetition Within Framework Margins

**Description:** The system MUST allow per-minicompetition choice of gunning model from the set permitted by the raamovereenkomst. Weighting schema (price/quality/sustainability percentages) MUST validate to 100% total and no component below 10% (proportionality principle).

**Acceptance Criteria:**

**Scenario 3.1: Reject disallowed gunning model**
- GIVEN a raamovereenkomst that permits only `emvi_prijs_kwaliteit` and `emvi_prijs_kwaliteit_duurzaamheid`
  WHEN a minicompetition attempts to set `gunningsmodel = 'laagste_prijs'`
  THEN:
  - The system rejects with error: "**Gunningsmodel not allowed within this framework.** Permitted models for this RO: EMVI price/quality, EMVI price/quality/sustainability. Please choose one of these."
  - The dropdown for gunningsmodel only shows the permitted options

**Scenario 3.2: Reject weighting that does not sum to 100%**
- GIVEN a minicompetition with EMVI model and the user enters:
  - Prijs: 60%
  - Kwaliteit: 35%
  - Duurzaamheid: 4%
  - (Total: 99%)
  WHEN the user attempts to save
  THEN:
  - The system validates and rejects: "Weighting must sum to 100%. Current: 99%. Please adjust."
  - The problematic field is highlighted

**Scenario 3.3: Enforce minimum 10% threshold per criterion if included**
- GIVEN a minicompetition with EMVI model and the user enters:
  - Prijs: 60%
  - Kwaliteit: 35%
  - Duurzaamheid: 5%
  - (Total: 100% but duurzaamheid < 10%)
  WHEN the user attempts to save
  THEN:
  - The system rejects: "**Duurzaamheid is 5% but minimum is 10% if included.** Option 1: Increase duurzaamheid to ≥10%. Option 2: Remove duurzaamheid and adjust prijs/kwaliteit to 100%."
  - The UI suggests Option 2: "Set Duurzaamheid to 0% and reweight others"

**Scenario 3.4: Accept valid weighting**
- GIVEN a minicompetition with EMVI model and the user enters:
  - Prijs: 50%
  - Kwaliteit: 30%
  - Duurzaamheid: 20%
  - (Total: 100%, all ≥10% if included)
  WHEN the user saves
  THEN:
  - The system accepts and stores the weighting schema
  - The minicompetition now shows the weighting summary: "Price 50% | Quality 30% | Sustainability 20%"

**Scenario 3.5: Allow EMVI without sustainability (prijs + kwaliteit only)**
- GIVEN a minicompetition with EMVI model but the user chooses to omit sustainability:
  - Prijs: 60%
  - Kwaliteit: 40%
  - Duurzaamheid: 0%
  WHEN saved
  THEN:
  - The system accepts (both 60% and 40% are ≥10%)
  - The scoring interface does not present duurzaamheidscriteria for this competition
  - The award decision document notes "Awarded under price/quality EMVI model (no sustainability criteria)"

---

### REQ-RMC-004: Invitation Publication and Bilateral Q&A Channel

**Description:** The system MUST publish minicompetition invitations via secure channel (encrypted email or secure portal link) to selected suppliers. A two-way Q&A channel MUST anonymize vendor questions and share all answers with all invited suppliers.

**Acceptance Criteria:**

**Scenario 4.1: Send invitations with unique portal links**
- GIVEN 5 suppliers selected for MC-2024-001
  WHEN the user clicks "Publish Invitations"
  THEN:
  - Each supplier receives an individual email with:
    - Salutation: "Dear {supplier contact name},"
    - Subject: "Invitation to compete: {minicompetition title}"
    - Body: Invitation text, deadline, submission instructions
    - Unique deeplink: `https://purchaseq/supplier-portal/mc/{mc-id}/token/{unique-token-per-supplier}`
    - Links to downloadable documents (RFQ, PvE, contract template, price template)
  - The user sees a status page: "Invitations sent to 5 suppliers"
  - For each supplier, a read receipt is tracked; the user can see "Email opened: 2024-05-10 10:30 (TechStaff)"

**Scenario 4.2: Handle late Q&A questions with deadline flexibility**
- GIVEN a supplier submits a question on the due date but after the official Q&A closure (e.g., 3 days before deadline)
  WHEN the question arrives
  THEN:
  - The system logs: "Late Q&A question from {supplier} on {date} after closure at {closure_date}"
  - The user (procurement officer) receives notification and can choose:
    - Option A: "Don't answer (too late)" → Auto-response to supplier: "Your question arrived after Q&A closure. It was not answered to ensure equal treatment."
    - Option B: "Answer anyway" → Requires:
      - Text field: "Justification for answering late question:"
      - Checkbox: "Extend Q&A closure and deadline by {N} days?"
      - If extended, all 5 suppliers receive notification: "Q&A period extended to {new date}; new deadline is {new date}."

**Scenario 4.3: Anonymize vendor questions**
- GIVEN supplier "TechStaff" asks: "Can you clarify the expected availability of the Java developers?"
  WHEN the user answers
  THEN:
  - The answer is published as:
    - Question: "Can you clarify the expected availability of the Java developers?" (supplier name hidden)
    - Answer: "{answer text}"
  - The Q&A is published to all 5 invited suppliers via the portal and email
  - It is added to the "Public Q&A Document" downloadable by all suppliers
  - The scoring committee will also see the Q&A in the public documentation set

**Scenario 4.4: Guarantee equal information to all suppliers**
- GIVEN the procurement officer answers Supplier 1's question at 2024-05-12 10:00
  WHEN that same day
  THEN:
  - All other suppliers are notified (via email and portal): "New Q&A published: {question & answer}"
  - A consolidated Q&A document is generated and re-posted to the portal
  - The audit log records: "Q&A published to {supplier list} by {user} on {date/time}"

---

### REQ-RMC-005: Inschrijvingen Ontvangen en Sluiten op Tijdstip

**Description:** The system MUST accept bids until exactly the sluitingsDatum (to the second), reject late submissions automatically, and prevent opening of bids before the formal opening ceremony by the committee (four-eyes principle with dual-signature).

**Acceptance Criteria:**

**Scenario 5.1: Reject late submissions with automatic timestamp-based blocking**
- GIVEN a minicompetition with `sluitingsDatum = 2024-06-01 15:00:00 UTC`
  WHEN a supplier attempts to submit at 2024-06-01 15:00:01 UTC (1 second late)
  THEN:
  - The system rejects the submission immediately
  - The supplier receives an error message: "**Submission deadline has passed.** Your submission was received at 15:00:01 UTC on 2024-06-01, but the deadline was 15:00:00 UTC on the same date. We regret that we cannot accept late submissions to ensure fair treatment of all bidders."
  - The system logs the rejection with server timestamp and supplier identity to the audit trail
  - The procurement officer is NOT notified (expected behavior, not an incident)

**Scenario 5.2: Accept submissions submitted before deadline with timestamp proof**
- GIVEN 3 suppliers submit bids:
  - Supplier A: 2024-05-31 09:30:00
  - Supplier B: 2024-06-01 14:59:59
  - Supplier C: 2024-06-01 15:00:00 (exactly on deadline)
  WHEN submissions are received
  THEN:
  - All three are accepted and marked `tijdig = true`
  - Each submission stores `ingediend_datum` with server timestamp for audit purposes
  - When the opening PV is generated, it will list all three as "timely received"

**Scenario 5.3: Block unauthorized viewing of unopened bids (dual-signature gate)**
- GIVEN three bids received and the opening ceremony has NOT yet occurred
  WHEN a committee member (even authorized) attempts to view the bid documents or scores
  THEN:
  - The system blocks access and displays: "**Bids are sealed.** Opening ceremony has not yet commenced. Contact the procurement officer to schedule the formal opening."
  - The system logs the access attempt (to detect unauthorized snooking):
    - `audit_log: { "action": "attempt_view_sealed_bid", "user": "user-id", "timestamp": "...", "allowed": false, "reason": "opening_ceremony_not_started" }`

**Scenario 5.4: Dual-signature opening protocol**
- GIVEN the opening ceremony is scheduled for 2024-06-02 10:00
  WHEN two committee members (minimum for dual signature) navigate to "Open Bids" page
  THEN:
  - The system displays: "Two signatories required to formally open sealed bids. Both must authorize simultaneously."
  - User 1 clicks "I authorize the opening of these bids"
  - The system waits for User 2's authorization
  - User 2 clicks "I authorize the opening of these bids" within the same session or within 5 minutes
  - The system unseals the bids (decrypts if encrypted) and:
    - Sets status to `in_beoordeling`
    - Auto-generates an opening PV with:
      - Date/time of opening
      - Names of both signatories
      - List of submitted bidders
      - Bid amounts (anonymized per supplier, or full if detail required)
      - Any special notes (late questions, missing documents, etc.)
    - Both signatories digitally sign the PV via docudesk
    - The PV is stored as immutable PDF
    - All committee members are notified: "Bids are now open for review"

**Scenario 5.5: Prevent opening if fewer than two signatories available**
- GIVEN opening ceremony scheduled and only one committee member is present/available
  WHEN that member clicks "Open bids"
  THEN:
  - The system rejects: "Two authorized signatories required. Current available: 1 of 3. Reschedule opening ceremony."

---

### REQ-RMC-006: Beoordelingsproces met Meerdere Beoordelaars en Consensus

**Description:** Each quality criterion MUST be scored by minimally 3 committee members independently. Individual scores and justifications are recorded. Divergences >2 points (on a 10-point scale) trigger discussion flag. Final score is consensus-based, not automatically averaged.

**Acceptance Criteria:**

**Scenario 6.1: Initialize scoring matrix for independent review**
- GIVEN a minicompetition with:
  - 4 quality criteria (Experience, Methodology, Communication, KT)
  - 5 bid submissions
  - 3 committee members
  WHEN the bids are opened (REQ-RMC-005)
  THEN:
  - The system creates 4 × 5 × 3 = 60 `minicompetitie_beoordeling` rows
  - Each row starts with `score = NULL`
  - The scoring interface shows each member only their own rows
  - Member 1 sees 20 rows (4 criteria × 5 submissions); cannot see Members 2 or 3's scores
  - The interface note: "Your scores for these criteria. Other committee members' scores will be revealed after all members have finished scoring."

**Scenario 6.2: Flag divergence and trigger consensus discussion**
- GIVEN 3 committee members score inschrijving #1 on criterion "Plan van Aanpak" as:
  - Member A: 8
  - Member B: 4
  - Member C: 7
  WHEN Member C clicks "Submit" on their final score
  THEN:
  - The system detects divergence: max(8) - min(4) = 4 points > 2-point threshold
  - The criterion is automatically flagged: "Divergence detected: scores range from 4 to 8"
  - A consensus discussion action is created
  - The consensus_score remains NULL
  - Committee members are notified: "Divergence on criterion {X} in submission {Y}. Please review and reach consensus."
  - The scoring interface displays: "[⚠️ REQUIRES DISCUSSION] Plan van Aanpak: scores are 8, 4, 7. Please discuss and agree on a final score."

**Scenario 6.3: Reach consensus and record agreed score**
- GIVEN the divergence described above
  WHEN the committee meets (video call recorded to PV, or chat consensus in the system)
  THEN:
  - The procurement officer or committee chair opens a "Consensus Resolution" interface
  - It displays:
    - Individual scores: Member A: 8, Member B: 4, Member C: 7
    - Justifications from each member
    - A text field: "Final agreed score (0-max_punten for this criterion):"
    - A text field: "Consensus rationale (why we agreed on this score):"
  - After agreement, the chair clicks "Record Consensus"
  - The system sets:
    - `consensus_score = 7` (example)
    - `consensus_motivering = "After discussion, Member B's interpretation was incorrect. Plan of Approach clearly meets the 7-point threshold based on..."`
  - The consensus record is locked and added to the scoring PV

**Scenario 6.4: Auto-generate scoring PV with consensus results**
- GIVEN all submissions have been scored and consensus reached on divergences
  WHEN the committee chair clicks "Calculate Final Scores"
  THEN:
  - The system computes final weighted score per submission:
    - Submission 1: (crit1_consensus × weight1 + crit2_consensus × weight2 + ...) / 100
  - A ranking is established
  - An "Assessment Process Verbal" (PV) is auto-generated with:
    - Date/time of scoring period
    - Committee members present
    - Per-submission scoring table (consensus scores)
    - Per-criterion weighting applied
    - Final ranking (1st, 2nd, 3rd, etc.)
    - Note of any divergences that required discussion
  - Both the chair and secretary digitally sign the PV
  - Status changes to `in_beoordeling` (if not already) and then awaits award decision

**Scenario 6.5: Minimum of 3 reviewers enforced**
- GIVEN a minicompetition assigned to only 2 committee members
  WHEN the system attempts to finalize scoring
  THEN:
  - The system blocks: "Minimum 3 committee members required per AW2012 audit standards. Current: 2. Add at least 1 more member before scoring can be finalized."

---

### REQ-RMC-007: Volume Monitoring Real-Time Against Framework Maximum

**Description:** The system MUST track consumed volume per minicompetition award, per supplier, and in total. Warnings are triggered at 70%, 85%, and 95% of framework maximum. Awards are blocked if they would exceed the maximum without prior TenderNed notice.

**Acceptance Criteria:**

**Scenario 7.1: 70% threshold warning**
- GIVEN a framework with `maximumVolumeEur = 2.000.000` and currently `verbruiktVolumeEur = 1.400.000` (70%)
  WHEN a new minicompetition is started
  THEN:
  - A banner appears on the minicompetition page: "⚠️ **Framework at 70% of maximum budget.** EUR 1.4M of EUR 2M consumed. At current competition avg, approximately X more competitions can fit."
  - The raamovereenkomst owner is notified via email: "Framework RO-2024-IT-001 has reached 70% of budget consumption."

**Scenario 7.2: 85% threshold warning and email alert**
- GIVEN a framework at `verbruiktVolumeEur = 1.700.000` (85% of EUR 2M)
  WHEN viewed
  THEN:
  - A prominent banner appears: "🔴 **Framework at 85% of maximum.** Only EUR 300,000 remaining. Ensure upcoming competitions fit within this limit or plan a TenderNed wijziging."
  - The raamovereenkomst owner and category manager are notified: "RO-2024-IT-001 at 85% budget consumption. Time to evaluate extension or new procurement."

**Scenario 7.3: 95% threshold critical warning**
- GIVEN a framework at `verbruiktVolumeEur = 1.900.000` (95% of EUR 2M)
  WHEN viewed
  THEN:
  - A critical banner: "🔴🔴 **Framework at critical 95%.** Only EUR 100,000 remains. New competitions will likely be blocked unless framework is expanded via TenderNed."
  - Escalation email to director/portfolio manager

**Scenario 7.4: Block award exceeding maximum**
- GIVEN framework at EUR 1.950.000 consumed
  WHEN a new award of EUR 100.000 is attempted to be finalized
  THEN:
  - The system blocks with error: "**Award would exceed framework maximum.** EUR 1.95M + EUR 100K = EUR 2.05M exceeds limit of EUR 2M. Options:\n(1) **Cancel this competition** – do not proceed with this award\n(2) **File TenderNed wijziging** – publish notice to increase framework max to EUR 2.5M (or other amount), then retry award\n(3) **Reduce scope** – restart competition with lower budget\n(4) **Split competition** – award to multiple suppliers with lower amounts"

**Scenario 7.5: Per-supplier volume caps**
- GIVEN a supplier in the framework has a negotiated cap: `maximumVolumePerLeverancierEur = 500.000`
  AND the supplier has already consumed EUR 450.000
  WHEN an award of EUR 100.000 to this supplier is prepared
  THEN:
  - The system blocks: "**Award to {supplier} would exceed their cap.** EUR 450K + EUR 100K = EUR 550K exceeds limit of EUR 500K. Options:\n(1) Award to a different supplier\n(2) Reduce this award amount to max EUR 50K\n(3) Negotiate higher cap with supplier (admin action)"

**Scenario 7.6: Audit trail of volume thresholds crossed**
- GIVEN any of the above thresholds is crossed
  WHEN logged
  THEN:
  - The audit log records:
    ```json
    {
      "event": "framework_volume_threshold_crossed",
      "raamovereenkomst_id": "...",
      "threshold_percent": 70,
      "threshold_eur": 1400000,
      "timestamp": "2024-05-15T10:30:00Z",
      "notified_users": ["user-1", "user-2"]
    }
    ```

---

### REQ-RMC-008: Extension Management with Automatic Reminders

**Description:** The system MUST send notifications 180, 120, and 60 days before framework end date regarding extension decisions. An extension workflow supports motivation, mandate check, and light renegotiation (price indexation, scope confirmation) without material change.

**Acceptance Criteria:**

**Scenario 8.1: 180-day reminder for extension planning**
- GIVEN a framework with `eindDatum = 2026-12-31`
  WHEN the scheduled job runs on 2026-07-04 (180 days before)
  THEN:
  - A notification is sent to the raamovereenkomst owner (user who created it):
    - Subject: "Framework RO-2024-IT-001 expires in 180 days. Extension planning needed."
    - Body: "Your framework ends on 2026-12-31. If extension is desired, available options are:\n{list of verlengingsopties}\n\nCreate extension now" (CTA)
  - The owner can click the CTA to pre-fill an extension request form with the available options

**Scenario 8.2: 120-day and 60-day reminders**
- GIVEN the same framework
  WHEN the scheduled job runs on 2026-08-03 (120 days before) and 2026-10-31 (60 days before)
  THEN:
  - Similar reminders are sent
  - If an extension was already initiated, the reminder says: "Extension is in progress (decision expected by {date}). Current status: {status}."

**Scenario 8.3: Create extension with price indexation (CPI-linked)**
- GIVEN an extension in preparation
  WHEN the user selects the option to include "Price adjustment (CPI 2.3%)"
  THEN:
  - The system marks this as a proportional adjustment under AW2012 art 2.163d
  - The adjusted prices are calculated (original price × 1.023)
  - The user provides justification: "Indexation per annual CPI measurement, {reference date}."
  - The system logs: "Extension modification: Price indexation 2.3% justified under AW2012 art 2.163d – proportional adjustment."
  - The modification is approved (does not require full re-tendering)

**Scenario 8.4: Update KPI appendix without material change**
- GIVEN an extension in preparation where KPI (key performance indicators) or SLA (service level agreements) need update
  WHEN the user uploads an "Updated KPI Appendix v2" with minor revisions (e.g., response time SLA changed from 8h to 6h for high-priority tickets)
  THEN:
  - The system records this as a scope confirmation (not a material change)
  - The updated appendix is attached to the extension record
  - The justification "Performance expectations clarified per operational feedback" is logged
  - The extension proceeds without re-tendering

**Scenario 8.5: Reject extension if verlengingsopties are exhausted**
- GIVEN a framework with `verlengingsopties = [{duur_maanden: 12, eenmalig: true, max_aantal_keer: 1}]`
  AND the framework has already been extended once (the single allowed extension used)
  WHEN the owner attempts to create another extension
  THEN:
  - The system rejects: "**All available extension options have been used.** The original verlengingsopties allowed 1 extension of 12 months, which has been used. To continue this category, you must start a new procurement process per AW2012 art 2.140-2.144."
  - A link to "Start new tender" is provided

**Scenario 8.6: Formally record extension decision**
- GIVEN an extension is prepared with motivation and modifications reviewed
  WHEN the approver (e.g., procurement manager or director) clicks "Approve Extension"
  THEN:
  - A `raamovereenkomst_verlenging` record is created with:
    - `besluit = 'akkoord'`
    - `besluit_door_user_id = [approving user]`
    - `besluit_datum = [today]`
    - `nieuwe_eind_datum = [calculated end date]`
  - A notification is sent to all suppliers: "Framework RO-2024-IT-001 is extended until {nieuwe_eind_datum}. Pricing adjusted per {modifications}."
  - If TenderNed notice is required, it is published automatically

---

### REQ-RMC-009: Award Decision with Automatic Motivation Document

**Description:** The system MUST auto-generate an award decision document including score summary, choice motivation, loser feedback (Alcatel clause: 20-day dispute period), and activate compliance hold until the period expires.

**Acceptance Criteria:**

**Scenario 9.1: Generate award decision with full transparency**
- GIVEN a minicompetition with ranking:
  - 1st: Submission by TechStaff (score 87.5)
  - 2nd: Submission by CodeForce (score 84.2)
  - 3rd: Submission by StaffingOost (score 79.8)
  WHEN the procurement officer clicks "Generate Award Decision"
  THEN:
  - A PDF is auto-generated with:
    - **Section 1: Award Decision**
      - "...hereby award the contract to TechStaff... for a consideration of EUR 650,000, based on the following evaluation:"
    - **Section 2: Anonymous Ranking (visible to all bidders)**
      - Bidder 1 (winner): 87.5 points → **Awarded**
      - Bidder 2: 84.2 points
      - Bidder 3: 79.8 points
    - **Section 3: Per-Bidder Score Explanation (each bidder sees only their own)**
      - For CodeForce (2nd place):
        "Your submission scored as follows:
        - Experience: 25/30
        - Methodology: 20/25
        - Communication: 15/20
        - Knowledge Transfer: 14/15
        - **Total: 74/90 (raw); weighted: 84.2/100**
        
        [Detailed feedback per criterion]"
    - **Section 4: Rationale for Award**
      - "The winning bid (TechStaff) was selected because it achieved the highest weighted score under the evaluation model (EMVI 50% price, 40% quality, 10% sustainability). While slightly higher-priced than Bidder 2, the superior experience in microservices (30/30 vs 25/30) and knowledge-transfer planning justified the selection."
    - **Section 5: Alcatel Clause (Mandatory)**
      - "Per EU directive 2014/24/EU and Dutch law (AW2012 art 2.130), unsuccessful bidders have the right to challenge this decision within 20 calendar days from publication of this notice. Challenges must be filed by [date 20 days from today] to [legal address/email]. The procurement entity will not execute a contract with the awarded bidder until after expiry of this period or until all challenges are formally resolved."
    - **Section 6: Signature Block**
      - Procurement officer signature (digital)
      - Date and time of signing

**Scenario 9.2: Send decision to all bidders and set Alcatel deadline**
- GIVEN the award decision is generated and signed
  WHEN the procurement officer clicks "Send to Bidders"
  THEN:
  - Each bidder (winner and losers) receives an email with:
    - Personalized greeting
    - Full PDF decision document (with per-bidder feedback visible only to themselves)
    - Alcatel clause notice: "You have 20 calendar days from 2024-06-20 to challenge this decision."
    - Link to submission feedback portal
  - The system sets: `alcatel_deadline_date = 2024-06-20` (20 days from decision date)
  - A calendar event is created in the procurement officer's calendar: "Alcatel deadline for MC-2024-001 expires"
  - The minicompetition status changes to `gegund` (awarded)
  - Contract execution is **blocked** until after `alcatel_deadline_date`

**Scenario 9.3: Block contract execution during Alcatel period**
- GIVEN an award decision sent on 2024-05-31 with Alcatel deadline 2024-06-20
  WHEN the procurement officer attempts to click "Execute Contract" on 2024-06-10 (before deadline)
  THEN:
  - The system blocks with message: "**Cannot execute contract during Alcatel dispute period.** Deadline is 2024-06-20. After that date, contract execution will be permitted (unless a valid challenge is filed)."
  - A countdown timer shows: "11 days remaining before contract execution is permitted."

**Scenario 9.4: Unblock contract after Alcatel period expires**
- GIVEN Alcatel deadline is 2024-06-20
  WHEN the date passes and no challenge was filed
  THEN:
  - The system automatically clears the block
  - The procurement officer is notified: "Alcatel period has expired for MC-2024-001. Contract may now be executed."
  - The "Execute Contract" button becomes active again

---

### REQ-RMC-010: Complete Process Verbal and Audit Trail

**Description:** The system MUST auto-generate three digitally-signed process verbals (opening, scoring, award) per minicompetition. A complete audit trail of all mutations is maintained. Records are retained for 7 years per AW2012 art 4.13.

**Acceptance Criteria:**

**Scenario 10.1: Auto-generate opening PV**
- GIVEN a minicompetition bid opening (REQ-RMC-005 opening ceremony)
  WHEN the two signatories authorize the opening
  THEN:
  - An "Opening Process Verbal" (PV Openen Inschrijvingen) is auto-generated with:
    - **Header**: "PROCESS VERBAL – OPENING OF BIDS – {Minicompetition Title}"
    - **Details**:
      - Date and time of opening: 2024-06-02 10:30 UTC
      - Location / format: "Digital opening via purchaseq system"
      - Persons present / signatories: {Names and titles of 2 signatories}
    - **Received Submissions**:
      ```
      Bidder 1: TechStaff Uitzendbureau – Received 2024-05-25 09:15 – €720,000
      Bidder 2: CodeForce Consulting – Received 2024-05-28 14:45 – €650,000
      Bidder 3: StaffingOost – Received 2024-05-29 16:20 – €695,000
      ```
    - **Notes**: "All three bids received in time. No bids rejected. All documents present and complete per checklist."
    - **Signature Block**: Both signatories digitally sign via docudesk
    - **Storage**: PV is stored as immutable PDF in openregister with reference `minicompetitie_proces_verbaal.bijlage_ref`

**Scenario 10.2: Auto-generate scoring PV**
- GIVEN all bids have been scored and consensus reached (REQ-RMC-006)
  WHEN the committee chair clicks "Finalize Scoring"
  THEN:
  - A "Scoring Process Verbal" (PV Beoordeling) is auto-generated with:
    - **Header**: "PROCESS VERBAL – EVALUATION & SCORING – {Minicompetition Title}"
    - **Committee Members Present**: {Names, titles, roles (chair, secretary, member)}
    - **Scoring Period**: From [date] to [date]
    - **Evaluation Criteria & Scoring Table**:
      ```
      Criterion | Weight | Bidder1_Score | Bidder2_Score | Bidder3_Score
      Experience | 20% | 25 | 28 | 23
      Methodology | 15% | 20 | 20 | 19
      Communication | 5% | 15 | 13 | 12
      Knowledge Transfer | 10% | 14 | 15 | 12
      Price | 50% | 24 | 26 | 22
      ===================================================
      Weighted Total | 100% | 87.5 | 89.2 | 85.0
      ```
    - **Divergences Discussed**: "On criterion 'Experience', initial scoring diverged from 25-28 (Member A: 25, Member B: 28, Member C: 26). After discussion on 2024-06-10, consensus score of 25 was agreed, noting TechStaff's focus on legacy systems vs. microservices."
    - **Ranking**:
      ```
      1st: CodeForce – 89.2 points
      2nd: TechStaff – 87.5 points
      3rd: StaffingOost – 85.0 points
      ```
    - **Signature Block**: Committee chair and secretary digitally sign
    - **Storage**: Immutable PDF in openregister

**Scenario 10.3: Auto-generate award PV**
- GIVEN an award decision is finalized and the Alcatel period expires
  WHEN the contract is executed
  THEN:
  - An "Award Process Verbal" (PV Gunningsbeslissing) is auto-generated with:
    - **Header**: "PROCESS VERBAL – AWARD DECISION – {Minicompetition Title}"
    - **Award Details**:
      - Awarded to: CodeForce Consulting
      - Amount: EUR 650,000
      - Score: 89.2 / 100
      - Contract reference: {contract ID}
    - **Alcatel Clause Confirmation**: "Dispute period from 2024-05-31 to 2024-06-20 – no valid challenges filed."
    - **Signature Block**: Procurement officer and legal counsel (if applicable) digitally sign
    - **Storage**: Immutable PDF in openregister

**Scenario 10.4: Complete audit trail of all mutations**
- GIVEN any change to the minicompetition (score update, status change, supplier deselection, etc.)
  WHEN the mutation occurs
  THEN:
  - An audit log entry is created:
    ```json
    {
      "minicompetitie_id": "mc-uuid",
      "event": "score_recorded",
      "field_changed": "minicompetitie_beoordeling.score",
      "old_value": null,
      "new_value": 25,
      "changed_by": "user-id",
      "changed_at": "2024-06-02T10:30:00Z",
      "reason": "Opening ceremony completion"
    }
    ```
  - The audit log is immutable (append-only; no edits or deletions)

**Scenario 10.5: 7-year retention and purge protocol**
- GIVEN a minicompetition created on 2024-06-01
  WHEN that minicompetition's record reaches 7 years (2031-06-01) and beyond
  THEN:
  - The record is marked: `retention_status = 'within_period'` while < 7 years
  - At 7-year mark: `retention_status = 'bewaartermijn_verlopen'`
  - The record is NOT automatically deleted (remains in system for archive purposes)
  - If an authorized admin (with 2FA) executes a purge command: `DELETE minicompetitie WHERE id = {id}`, the system:
    - Requires explicit confirmation: "This will permanently delete {minicompetition}. This action cannot be undone."
    - Requires 2FA token
    - Logs the deletion to an immutable log: `{ "event": "record_purged", "record_id": "...", "purged_by": "admin-user", "reason": "7-year retention expired", "timestamp": "...", "twofa_verified": true }`

**Scenario 10.6: Export complete dossier for audit**
- GIVEN a request from an auditor for "all minicompetitions within RO-2024-IT-001 in Q2 2025"
  WHEN the user navigates to Raamovereenkomst > Audit Export and selects:
    - Framework: RO-2024-IT-001
    - Date range: 2025-04-01 to 2025-06-30
    - Click "Export"
  THEN:
  - A ZIP file is generated with structure:
    ```
    RO-2024-IT-001_audit_export_Q2_2025.zip
    ├── MC-2024-IT-001-001/
    │   ├── invitation.pdf
    │   ├── submissions/
    │   │   ├── TechStaff_inschrijving.pdf
    │   │   ├── CodeForce_inschrijving.pdf
    │   │   └── StaffingOost_inschrijving.pdf
    │   ├── scores/
    │   │   ├── scores_reviewer1.xlsx
    │   │   ├── scores_reviewer2.xlsx
    │   │   └── scores_reviewer3.xlsx
    │   ├── pvs/
    │   │   ├── pv_opening_SIGNED.pdf
    │   │   ├── pv_scoring_SIGNED.pdf
    │   │   └── pv_award_SIGNED.pdf
    │   ├── award_decision_SIGNED.pdf
    │   └── contract.pdf
    ├── MC-2024-IT-001-002/
    │   └── [same structure]
    ├── audit_log.jsonl (one event per line)
    └── README.txt (export metadata: date range, framework, user, timestamp)
    ```
  - The ZIP is encrypted with a one-time password sent to the auditor's email
  - Download link expires in 24 hours

---

## Non-Functional Requirements

### NFR-RMC-001: Performance

- Minicompetition creation and publication MUST complete in < 2 seconds (p95)
- Score calculation (weighted final scores for 50 submissions) MUST complete in < 5 seconds
- Audit export ZIP generation for 1-year period (52 weeks, ~200 minicompetitions) MUST complete in < 30 seconds

### NFR-RMC-002: Security & Authorization

- All PV documents MUST be digitally signed using docudesk integration with tamper-evident verification
- Bid viewing and scoring MUST enforce role-based access (committee member, procurement officer, auditor)
- Supplier portal links MUST be single-use or time-limited tokens (24-hour expiry)
- Audit log MUST be immutable (append-only) and protected from unauthorized modification

### NFR-RMC-003: Data Integrity

- Framework maximum volume constraint MUST prevent any award that would violate it (database-level check)
- Scoring divergence detection MUST trigger automatically without manual review
- Alcatel deadline block MUST prevent contract execution even if a user attempts to bypass via API

### NFR-RMC-004: Localization

- All text MUST support Dutch language (primary) and optionally English per interface setting
- Date/time formatting MUST follow Dutch locale (dd-MM-yyyy HH:mm)
- All auto-generated documents (PVs, award decisions) MUST be in Dutch unless supplier requests English

### NFR-RMC-005: Audit & Compliance

- Every action (score, deselection, threshold crossing) MUST be logged with user, timestamp, and reason
- PVs MUST be immutable once signed (database constraint)
- Export for compliance MUST include cryptographic checksums (SHA-256) so auditor can verify data integrity

---

## Traceability to ADRs

- **ADR-001 (Information Architecture)**: Placement as sub-page under Aanbestedingen > Raamovereenkomsten/Minicompetities
- **ADR-005 (Security)**: Digital signatures via docudesk; role-based access control; immutable audit logs
- **ADR-008 (Testing)**: Scenario-based tests for all REQ acceptance criteria; integration tests for volume constraints
- **ADR-011 (Schema Standards)**: All entities follow NEN schema standards; field names in Dutch snake_case
