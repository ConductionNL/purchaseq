---
status: draft
---
# TenderNed Publicatie Adapter

## Placement & Information Architecture

**Placement type:** `SETTING` — Setting under the app's Beheer/Admin/Configuration surface. Lives in the existing settings UI; no top-level menu entry.

**Lives at:** Beheer > Connectors > TenderNed

**Rationale:** adapter, not user menu  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Purpose

The TenderNed Publicatie Adapter capability gives purchaseq the bidirectional bridge to TenderNed, the Dutch national procurement platform (operated by PIANOo / Ministerie van Economische Zaken) where every Dutch contracting authority is legally obliged to publish European-threshold and most national-threshold tenders. Without this adapter purchaseq is just an internal procurement workbench; with it the tool becomes the single place where a Dutch inkoper drafts an aanbesteding, runs the internal goedkeuringsflow, presses "publiceren", and watches the procurement go live on TenderNed and (for boven-drempel) on TED (Tenders Electronic Daily, the EU-wide publication channel). The adapter must round-trip the full lifecycle: vooraankondiging (prior information notice), aankondiging van opdracht (contract notice), publicatie van inlichtingen (Q&A round), ontvangst van inschrijvingen (bid receipt, where TenderNed acts as the secure depot), gunningsbeslissing en aankondiging van gegunde opdracht (award notice), and post-award wijzigingen (modification notices for scope, term or value changes during contract execution). The adapter must also handle the negative paths: trekken (withdraw) van een aanbesteding before deadline, gestaakte procedure (no award after evaluation), and rectificatie (correction notice). Beyond the workflow, the adapter is the place where two regulatory regime shifts land: the migration from TED Schema (XSD) to eForms 2.0 (eForms became mandatory on 25 October 2023 for boven-drempel notices, and TenderNed phased the same forms into nationale aankondigingen during 2024-2025) and the gradual rollout of EU Open Data dataspaces for procurement (ePO / Public Procurement Data Space). The capability is consciously a Dutch-first specialisation; a future capability `ted-publicatie-adapter` will cover non-NL EU member states using the same eForms backbone but different national platforms (Boamp/FR, Plataforma de Contratos del Sector Publico/ES, BOSA/BE-federaal, e-Notices2 fallback). The adapter is intentionally thin on business rules - drempelbedragen, procedure choice, evaluation method live in `aanbesteding-werkproces` (the base capability) - and intentionally thick on protocol fidelity, error recovery, and audit. Every publication is a legal act; every modification leaves an audit trail; every status change from TenderNed must be reconciled back into the local aanbesteding object so the procurement team and external bidders see identical state. The adapter belongs in purchaseq because procurement publication has Dutch-specific quirks (NUTS-NL codes, CPV-NL terminology, AanbestedingsWet 2012 + Gewijzigd 2016 references, perceelindeling rules, motiveringsplicht voor niet-opdelen) that do not belong in the generic openconnector adapter layer.

## Data Model

The capability introduces and extends several schemas in the `purchaseq` register.

**Schema `tenderned_publicatie`** (new) records every publication transaction. Fields: `aanbestedingId` (ref to base aanbesteding), `publicatieType` (enum: vooraankondiging, aankondiging_opdracht, aankondiging_gunning, aankondiging_wijziging, rectificatie, vrijwillige_transparantievooraf, concessie, sociale_specifieke_diensten), `eformsNoticeSubtype` (the 40+ eForms subtype codes like "16" for open above-threshold or "29" for concession award), `tendernedPublicatieId` (the TN-internal identifier returned on successful publish), `tedPublicatieId` (where applicable, the TED publication number in format YYYY/S NNN-NNNNNN), `status` (concept, gevalideerd, ingediend, gepubliceerd, ingetrokken, geweigerd), `publicatieDatum`, `sluitingsDatum`, `eformsXml` (the canonical submitted payload), `validatieResultaten` (TED Validator + TN validator output with rule IDs), `bijlagen` (array of refs to `tenderned_bijlage`), `correlatieId` (UUID for idempotency), `gepubliceerdDoor` (user ref), `goedgekeurdDoor` (board/mandaat ref when above board-threshold).

**Schema `tenderned_bijlage`** (new) tracks document uploads: `publicatieId`, `documentRef` (OpenRegister file attachment), `bestandsnaam`, `mimeType`, `groottebytes`, `documentType` (selectieleidraad, gunningsleidraad, bestek, beschrijvend_document, programma_van_eisen, conceptovereenkomst, nota_van_inlichtingen, proces_verbaal_opening, gunningsbeslissing), `taal` (NL, EN, FR, DE - per AW2012 art 2.78), `tendernedBijlageId` (set on successful upload), `checksumSha256`, `vertrouwelijk` (boolean - vertrouwelijke documenten zijn alleen na inschrijving zichtbaar), `uploadStatus`.

**Schema `tenderned_vraag_antwoord`** (new) for the inlichtingenronde: `publicatieId`, `vraagnummer` (auto-incremented per aanbesteding), `vraag` (text, anonymised), `vraagstelDatum`, `antwoord` (text), `antwoordDatum`, `bijlagen` (refs), `betreftDocument` (ref to bestek/leidraad), `betreftHoofdstuk`, `status` (open, beantwoord, verwerkt_in_nota), `notaVanInlichtingenRef`.

**Schema `tenderned_inschrijving`** (new) for bid receipts coming back from TenderNed's secure depot: `publicatieId`, `inschrijverNaam` (post-opening only), `inschrijverKvK`, `tendernedInschrijvingId`, `ingediendDatum` (must be before sluitingsDatum), `openingsDatum`, `documentRefs` (encrypted until opening), `inschrijvingsBedrag` (post-opening), `combinatie` (boolean, with deelnemers array), `onderaanneming` (boolean, with onderaannemers), `geldigVerklaard` (boolean, with motivering).

**Schema `cpv_code`** (new, reference data) and `nuts_code` (new, reference data) are imported reference registries. CPV (Common Procurement Vocabulary) has ~9400 codes in a 9-digit hierarchy with verification digit; NUTS (Nomenclature of Territorial Units for Statistics) for NL has 4 NUTS-1, 12 NUTS-2, 40 NUTS-3 codes plus LAU municipality codes. Both are loaded as seed data and refreshed quarterly from EU Open Data Portal.

**Schema `drempelbedrag_periode`** (new) holds the two-year EU thresholds (revised every two years; 2024-2025 values: works 5.538.000, supplies/services central 143.000, supplies/services sub-central 221.000, social services 750.000, concessions 5.538.000) plus the indicative Dutch national drempelbedragen as published by PIANOo.

The base `aanbesteding` schema (owned by `aanbesteding-werkproces`) gains three reverse-reference fields managed by this capability: `actievePublicatieId`, `publicatieHistorie[]`, and `tendernedDossierUrl`.

## Requirements

**REQ-TPA-001 - Concept publicatie samenstellen uit aanbestedingsdossier**
The system MUST generate a concept `tenderned_publicatie` from an existing `aanbesteding` object, mapping internal fields (titel, CPV-hoofdcode, geraamde_waarde, procedure, gunningscriteria, perceelindeling, looptijd, plaats_van_uitvoering) to the corresponding eForms BT-fields (BT-21 title, BT-262 main CPV, BT-27 estimated value, BT-105 procedure, BT-539 award criteria, BT-137 lot identifier, BT-36 duration, BT-727 place of performance).
- GIVEN an aanbesteding with hoofd-CPV `45000000-7` (construction works), drie percelen, looptijd 24 maanden, gunning op EMVI, WHEN the user clicks "Genereer publicatie" voor type `aankondiging_opdracht`, THEN a concept publicatie is created with eformsNoticeSubtype "16" (open above-threshold works), three BT-137 lot entries, BT-105 set to "open", and BT-539 populated from the gunningscriteria fields.
- GIVEN an aanbesteding with geraamde_waarde 80.000 EUR and CPV in supplies, WHEN the concept is generated, THEN the system selects nationale aankondiging (not EU subtype) because the value is under the EU drempel for sub-central supplies.
- GIVEN an aanbesteding marked `concessie = true`, WHEN concept generation runs, THEN the publicatieType is forced to `concessie`, eformsNoticeSubtype to "25" (concession notice), and the drempelcheck uses the concessie drempel (5.538.000 EUR for 2024-2025).

**REQ-TPA-002 - eForms 2.0 XML serialisatie en validatie**
The system MUST serialise concept publicaties to eForms 2.0 SDK-compliant XML and validate them locally against (a) the EU TED-XSD schema for the chosen subtype, (b) the EU Schematron rules pack (eForms business rules), and (c) the TenderNed-specific extension rules before allowing submission.
- GIVEN a concept publicatie with eformsNoticeSubtype "16", WHEN the user requests validation, THEN the system produces an eForms XML conforming to SDK 1.13 (or the SDK version active on the configured `eformsSdkVersion` setting), runs xerces XSD validation, runs the Schematron rules through saxon-he, and returns a validatieResultaten array of {ruleId, severity, message, xpath}.
- GIVEN a publicatie missing a mandatory BT-field per Schematron rule "BR-OPT-13" (procurement project must have a title in the official language), WHEN validation runs, THEN the publicatie status remains `concept`, a validatieResultaten entry with severity `error` is recorded, and the UI surfaces the missing field with a deep link to the source aanbesteding field.
- GIVEN a publicatie that passes XSD and Schematron, WHEN TenderNed-extension validation runs (e.g. mandatory NL-specific BT-fields like motivering-niet-opdelen for opdrachten >EUR 1.000.000), THEN any warnings appear with severity `warning` and submission is still allowed but logged.

**REQ-TPA-003 - Documentupload met progress en checksum-verificatie**
The system MUST upload bijlagen to TenderNed via the multipart-document endpoint, support files up to 200 MB per document (TenderNed's hard limit), report upload progress to the UI, compute and verify a SHA-256 checksum end-to-end, and retry idempotently on 5xx errors using the openconnector retry envelope.
- GIVEN a `tenderned_bijlage` of 150 MB with documentType `bestek`, WHEN the user starts upload, THEN the system streams the file in 4 MB chunks, emits progress events every chunk (0%, 3%, 6%, ...), computes SHA-256 on the fly, and on completion sends a finalisation call including the checksum; if TenderNed's returned checksum mismatches, the upload is marked failed and retried up to 3 times.
- GIVEN a TenderNed response HTTP 503 during the third chunk, WHEN the retry logic engages, THEN the upload resumes from the last acknowledged chunk using the resumable-upload session token, not from byte zero.
- GIVEN a bijlage flagged `vertrouwelijk = true`, WHEN uploaded, THEN the API call sets the `confidentialityLevel` parameter to `RESTRICTED`, and the document is not exposed via the public TenderNed dossier URL.

**REQ-TPA-004 - Publicatie indienen en bevestiging registreren**
The system MUST submit validated publicaties to TenderNed, capture the returned `tendernedPublicatieId` plus (for boven-drempel) the `tedPublicatieId`, transition the publicatie status to `ingediend` immediately and to `gepubliceerd` once TenderNed confirms the publication is live, and record the publicatieDatum from the TenderNed response (not the local clock).
- GIVEN a publicatie that passed all validation and has goedkeuring from the bevoegde mandaathouder, WHEN the user clicks "Indienen bij TenderNed", THEN the system POSTs to /notices, receives a 202 Accepted with a correlation token, sets status to `ingediend`, and starts polling /notices/{token}/status every 30s until status `PUBLISHED` or `REJECTED`.
- GIVEN TenderNed returns status `PUBLISHED` after 90 seconds, WHEN the polling loop detects this, THEN the publicatie record gets `tendernedPublicatieId`, `tedPublicatieId` (if boven-drempel), `publicatieDatum` from the response, `tendernedDossierUrl`, and the base aanbesteding's `actievePublicatieId` is updated.
- GIVEN TenderNed returns status `REJECTED` with reason `INVALID_CPV_HIERARCHY`, WHEN the polling detects this, THEN the publicatie reverts to `concept`, a validatieResultaten entry captures the TN-side rule violation, and a notification is sent to the publicerende user with a deep link back to the publicatie editor.

**REQ-TPA-005 - Vragen-en-antwoorden round-trip**
The system MUST poll TenderNed for incoming vragen during the inlichtingenperiode, expose them in the purchaseq UI for response composition, allow batched publishing of antwoorden as a nota van inlichtingen, and republish the consolidated NvI as a document update to the original publicatie.
- GIVEN a published aanbesteding in inlichtingenperiode, WHEN the scheduled poll job runs (every 15 min), THEN any new vragen since `laatstePollMoment` are fetched, created as `tenderned_vraag_antwoord` records with status `open`, and a notification fires to the inkoper.
- GIVEN three antwoorden in status `open`, WHEN the inkoper marks them ready and clicks "Publiceer nota van inlichtingen", THEN the system generates a PDF NvI document, uploads it as a new `tenderned_bijlage` of type `nota_van_inlichtingen`, submits a rectificatie-aankondiging if any antwoord materially changed the bestek (per AW2012 art 2.65), and updates the vraag-antwoord statuses to `verwerkt_in_nota`.
- GIVEN a vraag arrives within 6 calendar days of the sluitingsDatum, WHEN it is registered, THEN the system flags it as `laat_ontvangen` and warns the inkoper that AW2012 art 2.65 lid 3 may require deadline extension.

**REQ-TPA-006 - Gunningsaankondiging en wijzigingsaankondigingen post-award**
The system MUST publish aankondigingen van gegunde opdracht within 30 calendar days of contract signature, support modification notices for in-contract changes (scope, value, term per AW2012 art 2.163a-g), and link them back to the original opdrachtaankondiging via the `tendernedPublicatieId` chain.
- GIVEN a contract signed on date X with winning leverancier, WHEN the gunningsaankondiging is generated, THEN it is pre-filled with award date, winnaar (KvK + naam + adres), gunningsbedrag, aantal_inschrijvingen, lowest/highest tender values, and submission deadline reminder is shown if the 30-day window is nearing.
- GIVEN an in-contract wijziging that increases the contract value by 8%, WHEN a wijzigingsaankondiging is drafted, THEN the system checks AW2012 art 2.163b (10% supplies/services / 15% works thresholds), warns if the cumulative wijzigingen exceed the threshold, and requires extra motivering before allowing submission.
- GIVEN a multi-perceel aanbesteding where only 2 of 3 percelen are awarded, WHEN the gunningsaankondiging is generated, THEN it covers only the gegunde percelen, marks perceel 3 as `niet_gegund` with reason from a configurable enum (geen_geschikte_inschrijving, ingetrokken, etc.), and a second `aankondiging_opdracht` may be initiated for perceel 3.

**REQ-TPA-007 - Status synchronisatie en reconciliation**
The system MUST run a daily reconciliation job comparing local publicatie statuses against TenderNed-side statuses, flag and surface any drift (lokaal `gepubliceerd` maar TenderNed `withdrawn`, of vice versa), and auto-correct where the TenderNed side is authoritative.
- GIVEN a local publicatie in status `gepubliceerd` whose TenderNed counterpart was withdrawn by an admin via the TN web UI, WHEN reconciliation runs, THEN the local status is updated to `ingetrokken`, a drift event is logged with reason `external_withdrawal`, and the inkoper is notified.
- GIVEN a local publicatie marked `ingetrokken` whose TenderNed counterpart is somehow still live, WHEN reconciliation runs, THEN a critical alert is raised because the legal positions diverge (the authority believes it withdrew, but bidders see an active call), with manual-action required.
- GIVEN reconciliation cannot reach TenderNed for 24h, WHEN this persists, THEN no auto-corrections are applied, all publicaties keep their last-known statuses, and an ops alert is raised.

**REQ-TPA-008 - Drempelbedragcheck en procedurevalidatie**
The system MUST validate that the gekozen procedure matches the geraamde_waarde against the active `drempelbedrag_periode`, distinguishing centrale overheid vs decentrale overheid vs speciale-sector-aanbesteders, and refuse submission for combinations that violate AW2012 (e.g. enkelvoudige onderhandse procedure for an opdracht boven EU-drempel).
- GIVEN a centrale overheidsentiteit with opdrachtwaarde EUR 250.000 voor diensten, WHEN procedure `meervoudig_onderhands` is selected, THEN validation fails because boven-drempel diensten (143k centraal) eisen Europese procedure, with a corrigerend voorstel "open" of "niet-openbaar".
- GIVEN a speciale-sector-aanbesteder (water/energie/vervoer/post) met opdracht EUR 400.000 voor leveringen, WHEN the drempelcheck runs, THEN de speciale-sector-drempel (443.000 EUR voor 2024-2025) wordt toegepast in plaats van de klassieke sectordrempel.
- GIVEN an opdracht for sociale en andere specifieke diensten (CPV-codes in Bijlage XIV) with value EUR 600.000, WHEN procedure check runs, THEN het verlichte regime van AW2012 deel 3 wordt toegestaan (drempel 750.000), met aankondigingstype `sociale_specifieke_diensten`.

**REQ-TPA-009 - CPV- en NUTS-code lookup en validatie**
The system MUST provide a typeahead-search CPV lookup with hierarchical context, validate the CPV check-digit, support multi-CPV (one hoofd + meerdere bij-CPV), validate NUTS-codes against the active NUTS-NL register, and reject invalid combinations.
- GIVEN a user typing "kant" in the CPV search, WHEN typeahead fires, THEN matches include "30000000-9 Kantoor- en computermachines", "30100000-0 Kantoormachines en -benodigdheden", with breadcrumb showing parent levels.
- GIVEN a user pastes CPV "30000000-2" (invalid check digit), WHEN validation runs, THEN the code is rejected with message "Ongeldig controlegetal voor CPV-code".
- GIVEN an aanbesteding with plaats van uitvoering "Gemeente Utrecht", WHEN the system resolves to NUTS, THEN it returns NUTS-3 `NL310` (Utrecht) and the LAU `0344` (Utrecht municipality).

**REQ-TPA-010 - Volledige audit-trail en bewaarplicht**
The system MUST keep an immutable audit trail of every publicatie-mutatie, store eForms XML payloads and TenderNed responses for at least 7 years (AW2012 art 4.13 bewaartermijn), expose the audit log to auditors via a read-only API, and ensure no purge job can delete bewaarplicht-records before the wettelijke termijn expires.
- GIVEN any state transition on a publicatie (concept→ingediend→gepubliceerd), WHEN the transition occurs, THEN an audit log entry is created with timestamp, actor, from-status, to-status, and a hash of the eForms XML; entries are append-only.
- GIVEN a publicatie record older than 7 years, WHEN an admin attempts to delete it, THEN the operation is allowed only via an explicit `bewaartermijn_verlopen` workflow with a second-factor approval.
- GIVEN an auditor request for "all publicaties for procedure X between dates Y-Z", WHEN the audit export endpoint is called, THEN a signed JSON+XML bundle is produced including all `tenderned_publicatie`, `tenderned_bijlage` checksums, `audit_log_entry` records, and `validatieResultaten`.

## Standards & Sources

- **Aanbestedingswet 2012** (geconsolideerd via Stb. 2016, 154 en latere wijzigingen) - in het bijzonder Hoofdstuk 2.3 (procedures), art 2.61 (publicatie), art 2.65 (inlichtingen), art 2.78 (taal), art 2.130 (gunningsbeslissing), art 2.163a-g (wijzigingen), art 4.13 (bewaartermijn). Het Aanbestedingsbesluit en de Gids Proportionaliteit (versie 3 januari 2022) voor procedurekeuze.
- **EU Directive 2014/24/EU** (klassieke sectoren), 2014/25/EU (speciale sectoren), 2014/23/EU (concessies) - geimplementeerd in AW2012; de drempelwaardes uit Verordeningen (EU) 2023/2497 (klassieke), 2023/2495 (speciale), 2023/2496 (concessies) gelden 2024-2025.
- **eForms 2.0** - EU Implementing Regulation 2019/1780 plus SDK-releases op het Publications Office GitHub (op.europa.eu/en/web/eforms). De adapter moet de actieve SDK-versie kunnen pinnen en upgrades binnen 6 maanden van EU-release verwerken.
- **TED eSenders technical documentation** voor publicatie naar TED, plus de TenderNed API-documentatie (PIANOo developer portal) voor de NL-specifieke endpoints en extension fields.
- **CPV-Verordening (EC) 2195/2002** geamendeerd via 213/2008 en latere updates voor de Common Procurement Vocabulary.
- **NUTS-Verordening (EC) 1059/2003** met de NUTS 2024-revisie als actieve basis.
- **PIANOo handreiking eForms** (najaar 2023) en **PIANOo handreiking elektronisch aanbesteden** als praktische gidsen voor inkopers; de adapter implementeert de aanbevelingen daaruit als default-instellingen.
- **Gids Proportionaliteit** voor de proportionaliteitstoets bij procedurekeuze, perceelindeling en geschiktheidseisen.
- **NEN 2767** en **STABU** voor bouw-aanbestedingen (referentiekader voor leidraden), **GIBIT 2020** voor IT-aanbestedingen.

## Cross-app integration

- **openconnector**: alle HTTP-traffic naar TenderNed loopt via een `tenderned` source met OAuth2 client-credentials auth, configureerbare base-URLs voor productie en pre-productie (TN biedt een test-omgeving), en de openconnector retry/timeout/circuit-breaker envelope. CallLog houdt alle requests/responses vast voor audit. Mappings vertalen interne JSON naar eForms-XML via XSLT.
- **aanbesteding-werkproces** (purchaseq base capability): leest de aanbesteding-objecten, schrijft `actievePublicatieId` en `publicatieHistorie`, triggert publicatie vanuit de werkproces-statemachine (status `gereed_voor_publicatie` → adapter neemt over).
- **raamovereenkomst-minicompetitie** (zustercapability): minicompetities binnen raamovereenkomsten kennen vereenvoudigde publicatie-eisen; de adapter expose't een aparte flow zonder volledige eForms voor zuivere mini-comp uitnodigingen onder de drempel.
- **mvi-sroi-aanbesteding** (zustercapability): MVI- en SROI-eisen leveren extra eForms BT-fields aan (BT-805 GreenProcurement, BT-755 Accessibility, eForms social criteria); de adapter haalt deze op uit de MVI-koppeling en injecteert ze in de XML.
- **docudesk**: gegenereerde leidraden, programma's van eisen en gunningsbeslissingen komen uit docudesk en worden als `tenderned_bijlage` doorgegeven; documentversies blijven gelinkt zodat een wijzigingsaankondiging het juiste documenthistoriespoor heeft.
- **openregister**: alle schema's leven in een `purchaseq`-register; auditlog gebruikt OR's ingebouwde audit-trail capability.
- **mydash**: KPI-tegels voor "publicaties laatste 30 dagen", "gemiddelde doorlooptijd vooraankondiging→gunning", "rejecties door TenderNed".
- **nldesign**: de publicatie-UI volgt NL Design System patronen voor government-grade formulieren; eForms validatie-errors gebruiken NLDS error-summary component.
- **n8n / pipelinq**: scheduled polling (vragen, status reconciliation) draait als n8n workflows die de purchaseq adapter-API consumeren; geeft observability en pauzeerbaarheid los van de PHP-app.

## Target users

- **Inkoper / aanbestedingsadviseur** (centrale overheid, gemeente, provincie, waterschap, zelfstandig bestuursorgaan, speciale-sector-aanbesteder): de primaire dagelijkse gebruiker, verantwoordelijk voor het opstellen van de aanbestedingsstukken, beantwoorden van vragen, en het indienen bij TenderNed.
- **Inkoopcoordinator / hoofd inkoop**: keurt publicaties goed boven bepaalde drempelwaardes (mandaat-afhankelijk) en monitort de fleet via mydash.
- **Juridisch adviseur**: kijkt mee bij wijzigingsaankondigingen, vrijwillige transparantieaankondigingen vooraf, en bij rectificaties met juridische impact.
- **Compliance officer / interne controle**: gebruikt de audit-trail-export voor rechtmatigheidscontrole en jaarlijkse accountantscontrole (verbijzonderde interne controle inkopen).
- **CISO / FG**: heeft inzage in CallLog en bewaartermijn-beleid voor AVG en BIO 2.0 compliance.
- **PIANOo / Ministerie van Binnenlandse Zaken**: geen directe gebruiker, maar de Open Data uitvoer is de geautomatiseerde compliance-leverancier voor hun statistische verplichting (jaarlijkse rapportage aan Brussel).
- **Leveranciers**: indirecte gebruiker; zij zien het resultaat op TenderNed.nl en op TED. De adapter zorgt ervoor dat hun ervaring (correcte teksten, werkende deeplinks naar bijlagen, kloppende deadlines) niet wordt gefrustreerd door publicatiebugs.
