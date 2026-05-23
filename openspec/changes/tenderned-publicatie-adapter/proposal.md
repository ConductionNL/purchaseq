# TenderNed Publicatie Adapter — Proposal

**Status:** Proposal  
**Created:** 2026-05-23  
**Change ID:** tenderned-publicatie-adapter

## Executive Summary

The TenderNed Publicatie Adapter gives purchaseq the bidirectional bridge to TenderNed, the Dutch national procurement platform, enabling Dutch contracting authorities to draft aanbestedingen in purchaseq, run the internal approval workflow, and publish to TenderNed and (for boven-drempel notices) to TED (Tenders Electronic Daily). The adapter handles the complete procurement lifecycle: vooraankondiging, contract notices, Q&A rounds, bid receipt, award notices, and post-award modifications. The adapter is Dutch-first, focusing on Dutch-specific regulatory requirements (AanbestedingsWet 2012, eForms 2.0 compliance, NUTS-NL codes, CPV hierarchy) while maintaining protocol fidelity and comprehensive audit trails for legal compliance.

## Demand

**Target User:** Inkoper / aanbestedingsadviseur at centrale overheid, gemeente, provincie, waterschap, zelfstandig bestuursorgaan, and speciale-sector-aanbesteders.

**Problem Statement:** Without this adapter, purchaseq is an internal procurement workbench only. Dutch inkopers must manually export data to external tools to publish European-threshold and most national-threshold tenders on TenderNed. This breaks the procurement workflow, introduces manual data-entry errors, prevents real-time status synchronization, and violates AW2012 audit trail requirements.

**Demand Drivers:**
- **Legal Compliance (High):** AW2012 art 2.61 requires all covered tenders to be published on TenderNed within 48 hours of decision to publish; TED publication is mandatory for boven-drempel. Every publication is a legal act that must be audited and retained for 7 years.
- **Workflow Integration (High):** Inkopers need a single place to draft, approve, and publish — not a multi-app manual export process.
- **eForms Migration (High):** EU Directive 2019/1780 and eForms 2.0 became mandatory on 2023-10-25 for boven-drempel; TenderNed phased the same forms into nationale aankondigingen during 2024-2025. Legacy TED Schema (XSD) is no longer supported.
- **Stakeholder Demand (Medium):** Inkoopcoördinatoren need visibility into publication status and historical audit trails; juridisch adviseurs need to validate legal language in notices before submission; compliance officers need export-ready audit data for jaarlijkse accountantscontrole.

**Value Proposition:**
- Eliminates manual export-to-external-tool step; procurement lifecycle stays inside purchaseq.
- Real-time status reconciliation: local status and TenderNed status are kept in sync; any drift is flagged and surfaced.
- Complete audit trail: every publication event is logged with timestamps, actors, eForms payloads, and TenderNed responses; 7-year retention is automatic.
- Compliance-by-default: eForms validation, drempelbedrag checks, and AW2012-specific rules are enforced before submission.
- Multi-language support: notices can be published in NL, EN, FR, DE per AW2012 art 2.78.

## Stakeholders

- **Inkoper / Aanbestedingsadviseur (Primary):** Drafts aanbestedingen, generates publication concepts, uploads documents, answers vragen, marks ready for approval. Daily user. Decision-maker for publication content.
- **Inkoopcoördinator / Hoofd Inkoop (Secondary):** Approves publications above mandaat-threshold, monitors fleet KPIs via mydash, handles escalations. Occasional user.
- **Juridisch Adviseur (Secondary):** Reviews legal language, validates notice content for AW2012 compliance, advises on wijzigingsaankondigingen and rectificaties. Occasional user.
- **Compliance Officer / Interne Controle (Tertiary):** Requests audit-trail exports for rechtmatigheidscontrole and accountantscontrole. Occasional.
- **CISO / FG (Tertiary):** Monitors CallLog and bewaartermijn policies for AVG and BIO 2.0 compliance. Rare user.
- **PIANOo / Ministerie van Binnenlandse Zaken (Indirect):** Not a direct user, but the adapter's Open Data output feeds compliance reporting to Brussel.

## User Journeys

### Journey 1: Publish an Open Procurement (Above-Threshold)

**Trigger:** Inkooper has completed aanbesteding object with titel, CPV-code, geraamde_waarde, procedure, gunningscriteria, en plaats_van_uitvoering, and it has transitioned to status `gereed_voor_publicatie`.

**Actors:** Inkoper, Inkoopcoördinator (for approval).

**Path:**
1. Inkoper navigates to Beheer > Connectors > TenderNed, clicks "Publiceer aanbesteding" for the aanbesteding.
2. System checks drempelbedrag and procedure compatibility; generates a concept `tenderned_publicatie` with `publicatieType = aankondiging_opdracht` and `eformsNoticeSubtype = 16` (open above-threshold works).
3. Inkoper uploads supporting documents (bestek, leidraad, besturingsdocumenten, beschrijvingen) as `tenderned_bijlage` objects with progress bar and SHA-256 checksum verification.
4. System validates the eForms XML locally (XSD, Schematron, TN-extension rules); validation results appear inline with deep links to source fields.
5. If above board-threshold, Inkoopcoördinator receives approval notification. Inkoper marks ready.
6. Inkoopcoördinator clicks "Keur goed" in the publicatie detail; system records goedgekeurdDoor and mandaat reference.
7. Inkoper clicks "Indienen bij TenderNed"; system POSTs the notice, receives 202 Accepted with correlation token, transitions status to `ingediend`.
8. System polls TenderNed every 30s until status returns `PUBLISHED` (typically 90s), then records `tendernedPublicatieId`, `tedPublicatieId`, `publicatieDatum`, and updates base aanbesteding's `actievePublicatieId`.
9. Inkoper sees "Gepubliceerd op TenderNed" with a deeplink to the live dossier. Notification fires. Journey ends.

**Sad Paths:**
- Validation fails: publication reverts to `concept`, validatieResultaten shows rule violations with deep links to source fields.
- TenderNed returns HTTP 503 during upload: openconnector retry envelope engages, resumes from last acknowledged chunk.
- TenderNed rejects with `INVALID_CPV_HIERARCHY`: publicatie reverts to `concept`, TN-side rule is captured in validatieResultaten, notification sent.
- Approval workflow times out: reminder notification escalates to manager.

### Journey 2: Answer Inlichtingen (Q&A Round)

**Trigger:** A published aanbesteding is in the inlichtingenperiode (between publicatie and sluitingsDatum).

**Actors:** Inkoper, optional Juridisch Adviseur.

**Path:**
1. Every 15 minutes, a scheduled n8n workflow polls TenderNed for new vragen.
2. New vragen are created as `tenderned_vraag_antwoord` records with status `open`. Notification fires to the Inkoper.
3. Inkoper views the vraag (anonymised text, optional document ref), drafts an antwoord.
4. Optional: Juridisch Adviseur reviews the antwoord for compliance and suggests changes.
5. Inkoper marks the antwoord ready and clicks "Publiceer nota van inlichtingen"; system generates a PDF NvI document, uploads it as a new `tenderned_bijlage` of type `nota_van_inlichtingen`.
6. If any antwoord materially changed the bestek (per AW2012 art 2.65), system submits a rectificatie-aankondiging automatically.
7. Vraag-antwoord statuses transition to `verwerkt_in_nota`. Journey ends.

**Sad Path:**
- Vraag arrives within 6 days of sluitingsDatum: system flags `laat_ontvangen = true` and warns that AW2012 art 2.65 lid 3 may require deadline extension.

### Journey 3: Publish Award Notice and Post-Award Modifications

**Trigger:** Contract has been signed with the winning leverancier.

**Actors:** Inkoper, Juridisch Adviseur (for major wijzigingen), Inkoopcoördinator (for approval).

**Path:**
1. Inkoper clicks "Genereer gunningsaankondiging" from the contract object.
2. System pre-fills with award date, winnaar name/KvK/adres, gunningsbedrag, aantal_inschrijvingen, and submission deadline reminder (30 calendar days per AW2012 art 2.130).
3. For multi-perceel aanbestedingen, system only includes gegunde percelen; marks niet-gegunde percelen with reason (geen_geschikte_inschrijving, ingetrokken, etc.).
4. Inkoper and optional Juridisch Adviseur review, then submit.
5. If an in-contract wijziging occurs (scope, value, or term per AW2012 art 2.163a-g), Inkoper drafts a wijzigingsaankondiging.
6. System checks AW2012 art 2.163b (10% supplies/services / 15% works cumulative thresholds); warns if exceeded, requires motivering.
7. Inkoopcoördinator approves; system submits to TenderNed and polls until PUBLISHED. Journey ends.

**Sad Path:**
- Wijziging exceeds cumulative threshold: system blocks submission until extra motivering is added.

### Journey 4: Reconciliation and Drift Handling

**Trigger:** Scheduled daily reconciliation job (runs overnight).

**Actors:** System, Ops alert (if drift detected).

**Path:**
1. System fetches all local publicaties with status `gepubliceerd` or `ingetrokken`.
2. For each, system queries TenderNed's /notices/{publicatieId}/status to fetch TN-side status.
3. If local status is `gepubliceerd` but TN status is `withdrawn`, system updates local status to `ingetrokken`, logs reason `external_withdrawal`, and notifies inkoper.
4. If local status is `ingetrokken` but TN status is `published`, system raises critical alert (legal position divergence) requiring manual investigation.
5. If TenderNed is unreachable for 24h, system stops polling, keeps last-known statuses, and raises ops alert. Journey waits for TenderNed recovery.

## Features

### Feature 1: Concept Publication Generation and eForms Serialization
**ID:** FEA-TPA-001  
**Demand Score:** 9/10  
**Description:** Generate a concept `tenderned_publicatie` from an existing aanbesteding, mapping internal fields to eForms 2.0 BT-fields. Serialize to eForms XML, validate locally (XSD, Schematron, TN-extension rules), and surface validation results inline with deep links to source aanbesteding fields.

### Feature 2: Document Upload with Resumable Sessions and Checksum Verification
**ID:** FEA-TPA-002  
**Demand Score:** 8/10  
**Description:** Upload `tenderned_bijlage` objects to TenderNed via multipart endpoints, support files up to 200 MB, report progress per chunk, compute SHA-256 checksums end-to-end, verify against TN-returned checksum, and retry idempotently on 5xx errors with resumable session tokens.

### Feature 3: Publication Submission and Status Polling
**ID:** FEA-TPA-003  
**Demand Score:** 9/10  
**Description:** Submit validated publicaties to TenderNed, capture `tendernedPublicatieId` and `tedPublicatieId`, transition status from `ingediend` to `gepubliceerd` via polling, and record publicatieDatum from TN response (not local clock). Handle rejections by reverting to concept and capturing TN-side rule violations.

### Feature 4: Vragen-en-Antwoorden Round-Trip and Nota van Inlichtingen
**ID:** FEA-TPA-004  
**Demand Score:** 8/10  
**Description:** Poll TenderNed every 15 minutes for incoming vragen during inlichtingenperiode, create `tenderned_vraag_antwoord` records, expose them for response composition, generate and publish a nota van inlichtingen PDF, and auto-submit rectificatie-aankondiging if material changes detected.

### Feature 5: Award Notice and Post-Award Modifications
**ID:** FEA-TPA-005  
**Demand Score:** 8/10  
**Description:** Generate gunningsaankondigingen pre-filled with award data, support wijzigingsaankondigingen for in-contract changes (scope, value, term), enforce AW2012 art 2.163b cumulative thresholds, and link notices back to original opdrachtaankondiging via the publicatie chain.

### Feature 6: Status Synchronization and Reconciliation
**ID:** FEA-TPA-006  
**Demand Score:** 7/10  
**Description:** Run daily reconciliation comparing local publicatie statuses against TenderNed-side statuses, flag and surface drift (local `gepubliceerd` but TN `withdrawn`, or vice versa), and auto-correct where TenderNed side is authoritative. Raise critical alerts for legal-position divergence.

### Feature 7: Drempelbedrag Validation and Procedure Check
**ID:** FEA-TPA-007  
**Demand Score:** 9/10  
**Description:** Validate that the gekozen procedure matches geraamde_waarde against active drempelbedrag_periode, distinguish centrale/decentrale/speciale-sector, apply speciale-sector thresholds, and enforce AW2012 procedure constraints (e.g., enkelvoudige onderhandse not allowed for boven-drempel).

### Feature 8: CPV and NUTS Code Lookup and Validation
**ID:** FEA-TPA-008  
**Demand Score:** 8/10  
**Description:** Provide typeahead-search CPV lookup with hierarchical context, validate CPV check-digits, support multi-CPV (one hoofd + multiple bij-CPV), validate NUTS-codes against NUTS-NL register, and reject invalid combinations.

### Feature 9: Audit Trail and Legal Retention
**ID:** FEA-TPA-009  
**Demand Score:** 9/10  
**Description:** Keep immutable audit trail of every publicatie-mutatie with timestamps, actors, eForms payloads, and TenderNed responses. Store records for 7 years (AW2012 art 4.13 bewaartermijn). Expose audit log to auditors via read-only API. Prevent purge of bewaarplicht-records before wettelijke termijn expires.

### Feature 10: Approval Workflow and Mandaat Integration
**ID:** FEA-TPA-010  
**Demand Score:** 8/10  
**Description:** Integrate with purchaseq's mandaat system for conditional approval gates based on publicatie-value, drempel, and procedure-type. Record goedgekeurdDoor and mandaat reference on submission. Support approval notifications and escalations for time-sensitive deadlines.

## Acceptance Criteria

All features MUST:
- Comply with AW2012 and EU Directive 2014/24/EU, 2014/25/EU, 2014/23/EU.
- Implement eForms 2.0 SDK-compliant serialization and validation (current SDK version pinned in settings).
- Maintain immutable audit trail with 7-year retention.
- Round-trip all status changes from TenderNed back to local state (daily reconciliation).
- Support resumable uploads and idempotent operations (openconnector retry envelope).
- Surface validation errors inline with deep links to source fields.
- Integrate with Nextcloud mandaat system for approval gates.
- Use OpenRegister for all schema persistence (no custom Entity/Mapper).

## Out of Scope

- Procedure-choice logic and drempelbedrag definitions live in `aanbesteding-werkproces` (not this adapter).
- Business rules for evaluation methods, perceelindeling rules, motiveringsplicht live in `aanbesteding-werkproces`.
- Minicompetities within raamovereenkomsten have a vereenvoudigde flow (separate from this adapter; see `raamovereenkomst-minicompetitie`).
- MVI and SROI criteria are supplied by the `mvi-sroi-aanbesteding` capability and injected into eForms by this adapter.
- Document generation (leidraden, programma's van eisen, gunningsbeslissingen) is supplied by docudesk; the adapter only receives and uploads them.
- EU Open Data dataspace integration (ePO / Public Procurement Data Space) is a future capability not in this scope.
- Non-NL EU member states (Boamp/FR, BOSA/BE) will be covered by `ted-publicatie-adapter` in a future release.

## Success Metrics

- **Time-to-Publish:** Inkopers can publish from concept to TenderNed live in <5 minutes for standard notices (no validation rework).
- **Validation Coverage:** 100% of mandatory eForms BT-fields and AW2012-specific rules are validated before submission.
- **Audit Completeness:** Every publication event is logged with timestamp, actor, and eForms payload; no orphan publications.
- **Status Accuracy:** Daily reconciliation catches and resolves all drift within 24 hours; zero legal-position divergences remain uncorrected.
- **User Adoption:** Within 6 months, >80% of Dutch inkopers using this feature for publication workflow.

## Implementation Constraints

- All HTTP traffic to TenderNed routes via openconnector `tenderned` source with OAuth2 client-credentials auth.
- eForms XML serialization uses XSLT mappings via openconnector (not custom PHP serializers).
- Scheduled jobs (vragen polling, daily reconciliation) run as n8n workflows consuming the adapter API (not embedded cron in PHP).
- All schemas and seed data live in OpenRegister; adapter has zero custom CRUD logic.

## Open Questions

- What is the active eForms SDK version to pin in settings? (Currently assumed SDK 1.13; EU releases new versions periodically.)
- Do we support published rectificatie-aankondigingen after ingetrokken status? (AW2012 art 2.67 permits this; should we allow it?)
- For speciale-sector-aanbesteders, should we auto-apply the sector-specific drempel or require manual selection? (Currently assumed auto-apply with UI warning.)
- What is the compliance-approval workflow for wijzigingsaankondigingen >AW2012 art 2.163b thresholds? (Currently assumes mandatory Juridisch Adviseur review; confirm with legal team.)

## Related Capabilities

- **aanbesteding-werkproces:** Base capability for aanbesteding object lifecycle; this adapter consumes and extends it.
- **mvi-sroi-aanbesteding:** Supplies extra eForms BT-fields for sustainability criteria; this adapter injects them.
- **raamovereenkomst-minicompetitie:** Minicompetities have vereenvoudigde eForms; not covered by this adapter.
- **docudesk:** Generates supporting documents; this adapter receives and uploads them.
- **openregister:** All schemas live here; zero custom Entity/Mapper.
- **openconnector:** All HTTP to TenderNed routes via here.
- **financeq:** Receives verplichting-refs from contracts; not in scope for this adapter.
- **planix:** Receives contract-deadline refs; not in scope for this adapter.

---

**Next Steps:** Design phase to detail eForms mapping tables, TenderNed API contracts, and n8n workflow definitions.
