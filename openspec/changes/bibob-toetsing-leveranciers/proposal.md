---
status: proposed
app: purchaseq
spec: bibob-toetsing-leveranciers
version: 1.0
depends_on:
  - purchaseq base
  - openconnector kvk-adapter
  - decidesk besluitvorming
  - docudesk
  - openzaak
  - shillinq
standards:
  - Wet bevordering integriteitsbeoordelingen door het openbaar bestuur (Wet Bibob)
  - Uitvoeringsregeling Wet Bibob
  - Algemene wet bestuursrecht (Awb) hoofdstuk 4 (besluitvorming)
  - AVG / Uitvoeringswet AVG (bijzondere persoonsgegevens)
  - Aanbestedingswet 2012 (uitsluitingsgronden artikel 2.86 / 2.87)
---

# BIBOB Toetsing Leveranciers en Opdrachtnemers

## Purpose

Gestructureerde uitvoering van de Wet Bibob in het aanbestedings- en vergunningsproces, met volledige dossiervorming, AVG-compliant verwerking van bijzondere persoonsgegevens, en een verdedigbaar bestuursrechtelijk besluit aan het einde van de keten. Het doel is dat de organisatie aantoonbaar het Bibob-instrument effectief inzet zonder onnodige administratieve last voor bonafide leveranciers, met respect voor de privacy van betrokkenen, en met een ijzersterk dossier voor de bezwaar- en beroepsfase.

## Target Users

- **Inkoper / aanbestedingsadviseur** — initieert aanbesteding, ontvangt risicowaarschuwing
- **Integriteitscoordinator (BIBOB-coordinator)** — voert formulier-1 en formulier-2 in, escalatiebeslissingen
- **Juridisch adviseur** — reviews motivering Awb-besluit, begeleidt zienswijze-procedure
- **Bestuurder / gemandateerd bestuursorgaan** — ondertekent formeel BIBOB-besluit
- **Landelijk Bureau Bibob (LBB)** — externe adviseur bij ernstige twijfel

## Features

### F1: Automatische risico-detectie (Demand: 9/10)

Bij nieuwe aanbestedingsopdrachten boven organisatie-drempel en/of in hoog-risico-sectoren (bouw, horeca, afval, vastgoed, transport) signaleert het systeem proactief dat BIBOB-onderzoek overwogen moet worden. De opdracht zelf wordt niet geblokkeerd.

**Subtasks:**
- Risico-indicatoren scannen: bedrag, sector, bedrijfsleeftijd, aandeeloverdracht
- Proactieve waarschuwing naar aanbestedingsadviseur
- Voorstel-formulier-1 als startknop

### F2: BIBOB-formulier 1 (eigen onderzoek) (Demand: 10/10)

Gestructureerde vragenlijst over financieringsbronnen, strafbare feiten, fiscale schulden, ondernemingsstructuur en zakelijke relaties. Resultaat: eerste-indicatie (geen-twijfel / twijfel / sterke-twijfel).

**Subtasks:**
- Gestandaardiseerde vragenset met bijlagen-upload (KvK-uittreksel, jaarrekening)
- Automatische KvK-data enrichment
- Eerste-indicatie vastleggen
- Blokkering "voorlopige-gunning" zonder afgeronde formulier-1

### F3: Escalatie naar formulier-2 (verdiepend onderzoek) (Demand: 8/10)

Bij indicatie "twijfel" of "sterke-twijfel" escalatie naar gestructureerd verdiepend onderzoek. Includes zienswijze-procedure conform Awb.

**Subtasks:**
- Gestructureerde keuze: opwaarderen naar formulier-2 of afsluiten met motivering
- Openbare-bronnen-raadpleging (Faillissementsregister, BIG, veroordelingen, media)
- Zienswijze-procedure: conceptbrief, 2-weken termijn, binnenkomende zienswijze registreren
- Auditlog vastleggen

### F4: LBB-adviesaanvraag (Demand: 7/10)

Bij onsluitend eigen oordeel: gestructureerde adviesaanvraag aan Landelijk Bureau Bibob met volledige onderbouwing. Advies sterk beveiligd opgeslagen, beperkt toegankelijk.

**Subtasks:**
- Verplichte onderdelen invullen: aanleiding, onderzoeksresultaten, specifieke vragen, betrokkenen, antwoord-datum
- Beveiligde verzending via beveiligde verbinding
- Advies in beveiligde container met loggen raadplaging
- Advies zichtbaar voor alleen bestuurder, BIBOB-coordinator, jurist

### F5: BIBOB-besluit met Awb-motivering (Demand: 10/10)

Formeel bestuursbesluit (gunning / niet-gunning / weigering / intrekking / voorwaardelijk) met verplichte Awb-artikel-3:46-motivering: feitelijke grondslag, juridische grondslag, belangenafweging, proportionaliteitstoets.

**Subtasks:**
- Besluit-template met verplichte onderdelen
- Minimum-tekstlengte per blok
- Digitale ondertekening via decidesk
- Bezwaartermijn-berekening en herinneringen

### F6: Privacy-gescheiden opslag bijzondere persoonsgegevens (Demand: 9/10)

Persoonsgegevens (strafbare feiten, financiële situatie) encrypted-at-rest in gescheiden tabel. Autorisatielijst: max. 3 personen (BIBOB-coordinator, jurist, bestuurder). Inkoper ziet alleen bestaan dossier en uiteindelijke conclusie.

**Subtasks:**
- Encrypted-at-rest opslag
- Expliciet autorisatielijst met 3-personen-limiet
- Field-level RBAC per rol
- Audit-log van elke raadpleging

### F7: Automatische bewaarregels en vernietiging (Demand: 8/10)

Na wettelijke bewaartermijn (5 jaar na onherroepelijk besluit): anonimisering persoonsgegevens, verwijdering LBB-advies, behoud geanonimiseerde statistieken. Vernietiging vereist tweede-handtekening archivaris.

**Subtasks:**
- Bewaartermijn-berekening per dossier
- Vernietigingsworkflow met anonimisering
- Archivaris-betrokkenheid en tweede-handtekening
- Logging van vernietiging

### F8: Bezwaartermijn-bewaking (Demand: 6/10)

Registratie uitreikingsdatum, berekening zes-weken-bezwaartermijn, herinneringen jurist, logging bezwaarschriften gekoppeld aan oorspronkelijk dossier.

**Subtasks:**
- Uitreikingsdatum registreren
- Termijn-berekening en herinneringssysteem
- Bezwaarschrift-tracking

## Data Entities

- **BibobDossier** — kerneenheid per onderzoek
- **BibobBetrokkene** — natuurlijke of rechtspersoon onder onderzoek
- **BibobFormulier1** — eigen onderzoek door bestuursorgaan
- **BibobFormulier2** — verdiepend onderzoek
- **LbbAdviesaanvraag** — adviesaanvraag Landelijk Bureau Bibob
- **BibobBesluit** — formeel bestuursbesluit
- **BibobBewaarregel** — afgeleid voor compliance

## User Stories

### US-001: Risicowaarschuwing bij nieuwe opdracht
**As** inkoper  
**I want** het systeem automatisch waarschuwen als nieuwe opdracht BIBOB-relevantie signaleert  
**So that** ik geen risico-leveranciers over het hoofd zie

**Acceptance criteria:**
- GIVEN nieuwe opdracht boven drempel EN in hoog-risico-sector
- WHEN opdracht in aanbestedingssysteem wordt ingevoerd
- THEN systeem stuurt melding naar BIBOB-coordinator met risicofactoren en voorstel-formulier-1

### US-002: Formulier-1 invullen
**As** BIBOB-coordinator  
**I want** gestructureerde vragenlijst met bijlagen-upload en KvK-enrichment  
**So that** ik volledige informatie verzamel voor eerste-indicatie

**Acceptance criteria:**
- GIVEN risicowaarschuwing ontvangen
- WHEN formulier-1 aangemaakt
- THEN vragen-per-thema (financiering, strafbare feiten, schulden, structuur, relaties) met upload-slots; KvK-data auto-verrijkt

### US-003: Escalatie naar formulier-2
**As** BIBOB-coordinator  
**I want** gestructureerde keuze om te escaleren of af te sluiten  
**So that** ik verhoogde twijfel correct inscaleer

**Acceptance criteria:**
- GIVEN formulier-1 afgerond met eerste-indicatie "twijfel" of "sterke-twijfel"
- WHEN dossier geopend voor escalatie-beslissing
- THEN systeem biedt keuze (a) formulier-2 met verplichting onderbouwing, of (b) afsluiten met motivering; beide gelogd

### US-004: Zienswijze-procedure
**As** juridisch adviseur  
**I want** geautomatiseerde zienswijze-brief en termijn-bewaking  
**So that** ik Awb-procedure correct volg

**Acceptance criteria:**
- GIVEN formulier-2 afgerond
- WHEN verdiepend onderzoek compleet
- THEN conceptbrief gegenereerd, 2-weken termijn ingesteld, binnenkomende zienswijze geregistreerd, definitief besluit geblokkeerd tot termijn verstreken

### US-005: LBB-adviesaanvraag
**As** BIBOB-coordinator  
**I want** gestructureerde aanvraag-builder naar Landelijk Bureau Bibob  
**So that** ik onsluitende situatie kan escaleren

**Acceptance criteria:**
- GIVEN dossier in formulier-2 zonder sluitend oordeel
- WHEN LBB-adviesaanvraag start
- THEN verplichte onderdelen ingevuld (aanleiding, onderzoeksresultaten, vragen, betrokkenen, datum); pas na volledigheid verzending

### US-006: BIBOB-besluit
**As** bestuurder  
**I want** besluit-template met verplichte Awb-motivering-onderdelen  
**So that** mijn besluit procedureel waterdicht is

**Acceptance criteria:**
- GIVEN onderzoek afgerond met conclusie
- WHEN bestuurder besluit voorbereidt
- THEN template met verplichte blokken (feitelijke grondslag, juridische basis, belangenafweging, proportionaliteit); minimum-tekstlengte; digitale ondertekening

### US-007: Privacy-beheer
**As** BIBOB-coordinator  
**I want** RBAC-gescheiden zicht op gevoelige dossier-onderdelen  
**So that** ik inkopers geen bijzondere persoonsgegevens toon

**Acceptance criteria:**
- GIVEN dossier bevat persoonsgegevens strafbare feiten/schulden
- WHEN inkoper dossier opent
- THEN zien alleen "dossier open" + uiteindelijke conclusie, NIET onderzoeksdetails; BIBOB-coordinator/jurist/bestuurder zien alles

### US-008: Bewaarregels
**As** archivaris  
**I want** automatische vernietigingsworkflow na bewaartermijn  
**So that** ik compliance waarborg

**Acceptance criteria:**
- GIVEN gesloten dossier met 5-jaar-termijn verstrijken
- WHEN vernietigingsworkflow start
- THEN persoonsgegevens anonimiseerd, LBB-advies verwijderd, statistieken bewaard, archivaris tekent tweede-handtekening, logging onomkeerbaar

## Cross-app Dependencies

- **purchaseq base**: opdracht/leverancier-context als aanleiding
- **openconnector kvk-adapter**: automatische KvK-verrijking en UBO-register
- **decidesk besluitvorming**: BIBOB-besluit digitale ondertekening, mandaatregister
- **docudesk**: dossiervorming en vernietigingsworkflow
- **openzaak**: BIBOB-onderzoek registreren als zaak
- **shillinq**: betaling-blokkering leverancier bij lopend onderzoek

## Acceptance Criteria (Proposal Level)

- ✓ Alle acht features kunnen end-to-end getest worden (risicowaarschuwing → BIBOB-besluit)
- ✓ Privacy-gescheidenis verifieerbaar via auditlog
- ✓ Wettelijke motiveringsverplichting (Awb 3:46) gedwongen in besluit-template
- ✓ Bewaartermijn en vernietiging automatisch berekend en gelogd
- ✓ Cross-app-dependencies voor openconnector/decidesk/docudesk geverifieerd
