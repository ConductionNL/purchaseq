## Status

Draft — purchaseq spec brief, 2026-05-21.

# Supplier Onboarding — Vragenlijst & Workflow

## Purpose

Provide a structured, risk-tiered onboarding workflow for new leveranciers (suppliers) entering a gemeentelijke or provinciale inkoop-administratie. Today, supplier-intake is typically a PDF questionnaire mailed to crediteurenadministratie, manually re-typed into the ERP, with KvK and BTW re-typed (and re-mistyped) into the grootboek. purchaseq replaces that with a self-service supplier-portal flow plus an internal approval-pipeline: the leverancier completes a category-aware questionnaire (a logistiek-leverancier doesn't get the BIBOB-vragen; a zorg-aanbieder does), uploads bewijslast (KvK-uittreksel, IBAN-bewijs, VOG, certificaten), and the data is validated against external bronnen (KvK-API, BTW-Vies, Peppol-directory) before the leverancier ever lands in the crediteurenstam. The result: cleaner master data, faster Peppol-onboarding, MVI/SDG-conformiteits-bewijs op orde, and BIBOB-doorlichting in-process voor risico-categorieën zonder dat een leverancier ooit weer een Word-bijlage hoeft te mailen. Jaarlijkse her-validatie houdt het dossier vers.

## Data Model

- **Leverancier**: handelsnaam, statutaire naam, KvK-nr, BTW-nr (RSIN), Peppol-id, rechtsvorm, vestigingsadres, correspondentieadres, IBAN, IBAN-tnv, contactpersoon, e-mail, telefoon.
- **OnboardingDossier**: leverancier, categorie (logistiek/dienstverlening/bouw/ict/zorg/...), risico-tier (laag/midden/hoog), status (uitnodiging/ingevuld/intern_review/goedgekeurd/afgewezen/verlopen), uitnodigingsdatum, gereeddatum, vervaldatum (jaarlijks).
- **VragenlijstAntwoord**: dossier, vraag-code, antwoord (string/bool/select/file-ref), validatie-status (ok/te_controleren/fout), validatie-bron (KvK-API/BTW-Vies/handmatig).
- **Bewijsstuk**: dossier, type (KvK-uittreksel/IBAN-bewijs/VOG/VAR/MVI-cert/ISO-cert/SBR-uittreksel), bestand, geldig_tot, geverifieerd_door.
- **BIBOBToets**: dossier, aanleiding, status, behandelaar, conclusie, vastgelegd_op.
- **ApprovalStep**: dossier, rol (categorie-inkoper/contractmanager/financien/integriteit), beslisser, beslissing, motivatie, datum.

## Requirements

**REQ-001: Category-driven questionnaire branching.** GIVEN a leverancier selecteert tijdens self-service-intake een categorie (logistiek/dienstverlening/bouw/ict/zorg), WHEN de vragenlijst geladen wordt, THEN purchaseq toont alleen vragen relevant voor die categorie + risico-tier; een logistiek-leverancier krijgt geen BIG-registratie-vraag, een zorg-aanbieder geen UAV-GC-vraag.

**REQ-002: KvK validatie real-time.** GIVEN een leverancier vult een KvK-nummer in, WHEN het veld blur-event vuurt, THEN purchaseq roept de KvK-API aan; bij hit pre-vult statutaire naam, rechtsvorm, vestigingsadres en SBI-code; bij miss markeert het veld validatie-status=fout met een actie "handmatig controleren" voor de inkoper.

**REQ-003: IBAN-validatie incl. tnv-check.** GIVEN een ingevulde IBAN, WHEN opgeslagen, THEN purchaseq valideert de IBAN-checksum, en — indien IBAN-Naam-Check-service beschikbaar is (Surepay) — verifieert de tenaamstelling tegen de statutaire naam; mismatch blokkeert dossier-goedkeuring tot opheldering.

**REQ-004: BIBOB-tiering op risico-categorie.** GIVEN een dossier in een categorie die volgens het gemeentelijk BIBOB-beleid risico-tier=hoog kent (bv. vastgoed, horeca, bouw boven drempel), WHEN dossier wordt ingediend, THEN purchaseq creëert automatisch een BIBOBToets-record en routeert naar de integriteitscoördinator; categorieën met tier=laag/midden slaan deze stap over.

**REQ-005: MVI-conformiteits-bewijs verplicht boven drempelbedrag.** GIVEN een verwachte jaaromzet met deze leverancier > €50k (configurabel), WHEN dossier wordt ingediend, THEN purchaseq verplicht upload van MVI-bewijslast (PSO-certificaat, CO2-prestatieladder, of self-declaration tegen het lokale MVI-beleid) voordat status naar intern_review kan.

**REQ-006: Peppol-onboarding-check.** GIVEN een leverancier vult een Peppol-id in, WHEN opgeslagen, THEN purchaseq query't de Peppol-SML/SMP-directory om te bevestigen dat het Peppol-endpoint actief is en welke document-types (UBL-invoice, OrderResponse) ondersteund worden; resultaat wordt vastgelegd op het dossier.

**REQ-007: Multi-step approval met rolgebaseerde routing.** GIVEN een dossier transitioneert naar status=intern_review, WHEN routing wordt bepaald, THEN purchaseq creëert sequentiële ApprovalSteps voor categorie-inkoper → contractmanager → financien (IBAN/crediteurenstam), plus parallel een integriteit-step als BIBOBToets bestaat; pas na alle goedkeuringen wordt de leverancier in de crediteurenstam (ERP) gepushed.

**REQ-008: Jaarlijkse her-validatie.** GIVEN een goedgekeurd dossier passeert zijn vervaldatum (default +12 maanden), WHEN de daily job draait, THEN purchaseq mailt de leverancier een uitnodiging om wijzigingen door te geven; ongewijzigde dossiers krijgen een 1-klik bevestiging, gewijzigde dossiers doorlopen verkorte re-review; verlopen dossiers zonder respons na 60 dagen worden gemarkeerd inactief in de crediteurenstam.

## Standards

- **NEN-EN-IEC 16931** — Europese factuur-norm (Peppol-BIS Billing 3.0).
- **Peppol BIS 3.0** — e-invoicing en e-ordering.
- **Wet BIBOB** — bevordering integriteitsbeoordelingen openbaar bestuur.
- **MVI-criteria PIANOo** — Maatschappelijk Verantwoord Inkopen.
- **PSO 30+** — Prestatieladder Socialer Ondernemen.
- **CO2-Prestatieladder (SKAO)**.
- **KvK API & VIES BTW-validatie**.
- **ISO 20022** — financiële berichten incl. SEPA.

## Cross-app

- **openregister** — Leverancier / OnboardingDossier / VragenlijstAntwoord schemas.
- **openconnector** — adapters naar KvK-API, VIES, Surepay, Peppol-SMP, en ERP (AFAS, Unit4, Exact, SAP).
- **docudesk** — bewijsstuk-archivering met retentie (7 jaar fiscaal, 10 jaar contract).
- **decidesk** — formele goedkeurings-besluiten voor grote contracten boven mandaatdrempel.
- **opencatalogi** — gepubliceerde leveranciers-lijst onder Woo (transparantie inkoop > drempelbedrag).

## Target users

- **Leverancier** — self-service portal, vragenlijst invullen, bewijslast uploaden, jaarlijks bevestigen.
- **Categorie-inkoper** — eerste review op inhoudelijke geschiktheid.
- **Contractmanager** — review op voorwaarden en MVI-conformiteit.
- **Crediteurenadministratie / Financien** — IBAN-validatie, ERP-koppeling.
- **Integriteitscoördinator** — BIBOB-toets bij risico-categorieën.
- **Inkoopcoördinator** — overzicht actieve dossiers, verlopen dossiers, KPI's op doorlooptijd.
