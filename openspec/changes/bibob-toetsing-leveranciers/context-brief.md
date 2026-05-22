---
status: draft
app: purchaseq
spec: bibob-toetsing-leveranciers
depends_on:
  - purchaseq base
target_users:
  - Inkoper / aanbestedingsadviseur
  - Integriteitscoordinator (BIBOB-coordinator)
  - Juridisch adviseur
  - Bestuurder / gemandateerd bestuursorgaan
  - Landelijk Bureau Bibob (LBB) als externe adviseur
standards:
  - Wet bevordering integriteitsbeoordelingen door het openbaar bestuur (Wet Bibob)
  - Uitvoeringsregeling Wet Bibob
  - Beleidsregel Bibob van het bestuursorgaan
  - Algemene wet bestuursrecht (Awb) hoofdstuk 4 (besluitvorming)
  - AVG / Uitvoeringswet AVG (bijzondere persoonsgegevens)
  - Aanbestedingswet 2012 (uitsluitingsgronden artikel 2.86 / 2.87)
  - Wet politiegegevens (Wpg) voor LBB-rapportage
  - Wet ter voorkoming van witwassen en financieren van terrorisme (Wwft) — UBO-overlap
---

# BIBOB Toetsing Leveranciers en Opdrachtnemers

## Purpose

Gestructureerde uitvoering van de Wet Bibob in het aanbestedings- en vergunningsproces, met volledige dossiervorming, AVG-compliant verwerking van bijzondere persoonsgegevens, en een verdedigbaar bestuursrechtelijk besluit aan het einde van de keten. De Wet Bibob geeft bestuursorganen het instrument om bij vermoeden van crimineel misbruik van overheidsopdrachten of -vergunningen onderzoek te doen naar de integriteit van de aanvrager, en op basis daarvan een opdracht niet te gunnen of een vergunning te weigeren of in te trekken. De praktijk is dat bestuursorganen vaak de wet wel kennen maar het uitvoeringsproces zwak organiseren: formulieren raken zoek, termijnen verlopen, de motivering van de uiteindelijke beslissing is dun, en bij een gang naar de rechter sneuvelt het besluit op procedurele gronden.

Deze spec maakt het BIBOB-proces een gestandaardiseerd workflow-traject: per risico-leverancier wordt automatisch BIBOB-formulier 1 (eigen onderzoek) aangeboden, op basis van de uitkomsten daarvan kan worden geescaleerd naar BIBOB-formulier 2 (verdiepend onderzoek), en bij aanhoudende twijfel wordt een adviesaanvraag bij het Landelijk Bureau Bibob (LBB) voorbereid. Het uiteindelijke gunningsbesluit of weigeringsbesluit is volledig onderbouwd, voldoet aan de motiveringsplicht van de Awb, en het dossier is bewaard volgens de strikte bewaarregels van de Uitvoeringsregeling Wet Bibob.

Het doel is dat de organisatie aantoonbaar het Bibob-instrument effectief inzet zonder onnodige administratieve last voor bonafide leveranciers, met respect voor de privacy van betrokkenen, en met een ijzersterk dossier voor de bezwaar- en beroepsfase.

## Data Model

**BibobDossier** — kerneenheid per onderzoek. Velden: dossiernummer, aanleiding (aanbesteding/vergunning/subsidie), aanleiding-referentie (opdracht-id of vergunning-id), risico-indicatoren (sector hoog-risico bouw/horeca/vastgoed/afval ja/nee, opvallende prijs, atypische ondernemingsstructuur, recente overdracht aandelen), onderzoeksfase (geen/formulier-1/formulier-2/lbb-advies/besluit), startdatum, deadline-datum, status (open/onderzoek/advies/besloten/gesloten), conclusie (geen-bezwaar/lichte-mate/ernstige-mate-van-gevaar), besluit-link.

**BibobBetrokkene** — natuurlijke of rechtspersoon die wordt onderzocht. Velden: type (aanvrager/bestuurder/aandeelhouder/UBO/financier/leidinggevende/zakelijke-relatie), naam, KvK-nummer (rechtspersoon), BSN (natuurlijk persoon, sterk beveiligd), geboortedatum, woonadres, relatie-tot-aanvrager, mate-van-zeggenschap-percentage, dossier-id.

**BibobFormulier1** — eigen onderzoek door bestuursorgaan. Velden: dossier-id, ingevuld-door, ingevuld-op, antwoorden-jsonb (gestandaardiseerde vragenlijst over financieringsbronnen, eerdere veroordelingen, fiscale schulden, ondernemingsstructuur, zakelijke relaties), bijgevoegde-stukken (KvK-uittreksel, jaarrekening, financieringsovereenkomsten), eerste-indicatie (geen-twijfel/twijfel/sterke-twijfel).

**BibobFormulier2** — verdiepend onderzoek. Velden: dossier-id, aanvullende-vragen, openbare-bronnen-geraadpleegd (Faillissementsregister, BIG-register, openbare-veroordelingen, mediabronnen), zienswijze-betrokkene, voorlopige-conclusie.

**LbbAdviesaanvraag** — alleen bij ernstige twijfel. Velden: dossier-id, aanvraagdatum, aanvraag-tekst (gestructureerde onderbouwing waarom LBB-advies nodig is), verzending-bevestiging, LBB-zaaknummer, advies-datum, advies-tekst (beveiligd opgeslagen, beperkt toegankelijk), advies-conclusie, geldigheidsduur-advies (max 2 jaar).

**BibobBesluit** — formeel bestuursbesluit. Velden: dossier-id, besluit-type (gunning/niet-gunning/weigering/intrekking/voorwaardelijk), motivering (lange tekst, voldoet aan Awb 3:46), juridische-grondslag, bezwaartermijn-startdatum, ondertekenaar (gemandateerd bestuursorgaan), datum-uitreiking-aan-betrokkene, ontvangstbevestiging.

**BibobBewaarregel** — afgeleid, voor compliance. Per dossier de berekende vernietigingsdatum (LBB-advies max 5 jaar na onherroepelijk besluit; eigen onderzoek 5 jaar na sluiting dossier), met automatische workflow voor anonimisering/vernietiging.

## Requirements

### REQ-001: Automatische risico-detectie bij nieuwe opdrachten

GIVEN een nieuwe aanbestedingsopdracht in purchaseq
WHEN de opdracht een geraamde waarde boven de organisatie-drempel heeft EN/OF in een hoog-risico-sector valt (vooraf gedefinieerd: bouw, horeca, afvalverwerking, vastgoed, transport) EN/OF de leverancier minder dan 24 maanden bestaat of recent van eigenaar is gewisseld
THEN signaleert het systeem proactief aan de aanbestedingsadviseur dat een BIBOB-onderzoek overwogen moet worden, met voorstel-formulier-1 als startknop, zonder de aanbesteding zelf te blokkeren.

### REQ-002: BIBOB-formulier 1 als verplichte stap bij risico-indicatoren

GIVEN een aanbestedingsopdracht waar de risico-detectie heeft afgegaan
WHEN de inkoper de opdracht naar status "voorlopige-gunning" wil zetten
THEN dwingt het systeem af dat BIBOB-formulier 1 is ingevuld en de eerste-indicatie is vastgelegd, of dat een gemotiveerde afwijzing is geregistreerd door de integriteitscoordinator waarom het onderzoek niet nodig is.

### REQ-003: Privacy-gescheiden opslag bijzondere persoonsgegevens

GIVEN dat een BIBOB-dossier persoonsgegevens bevat over strafbare feiten of financiele situatie van natuurlijke personen
WHEN deze gegevens worden opgeslagen
THEN gebeurt dat in een gescheiden, encrypted-at-rest tabel met expliciete autorisatielijst van maximaal 3 personen (BIBOB-coordinator, jurist, bestuurder); reguliere inkopers en afdelingshoofden zien alleen het bestaan van een dossier en de uiteindelijke conclusie, nooit de inhoud van het onderzoek.

### REQ-004: Escalatie van formulier 1 naar formulier 2

GIVEN een afgerond BIBOB-formulier-1 met eerste-indicatie "twijfel" of "sterke-twijfel"
WHEN de BIBOB-coordinator het dossier opent
THEN biedt het systeem een gestructureerde keuze: (a) opwaarderen naar formulier 2 met verplichte onderbouwing en zienswijze-procedure voor de betrokkene, of (b) afsluiten met motivering waarom verder onderzoek niet proportioneel is; beide keuzes worden vastgelegd in de auditlog.

### REQ-005: Zienswijze-procedure conform Awb

GIVEN een dossier in fase "formulier-2"
WHEN het verdiepend onderzoek is afgerond
THEN genereert het systeem een conceptbrief aan de betrokkene met de voorgenomen conclusie, biedt een termijn van twee weken voor zienswijze, registreert de binnenkomende zienswijze (geupload document of schriftelijke notulen van mondelinge zienswijze), en blokkeert het definitieve besluit tot deze stap is voltooid of de termijn verstreken.

### REQ-006: LBB-adviesaanvraag met gestructureerde onderbouwing

GIVEN een dossier waar de organisatie zelf geen sluitend oordeel kan vormen
WHEN de BIBOB-coordinator een LBB-adviesaanvraag start
THEN dwingt het systeem af dat de volgende onderdelen volledig zijn ingevuld: aanleiding, eigen onderzoeksresultaten samengevat, specifieke vragen aan LBB, alle bekende betrokkenen met hun rol, en de gewenste antwoord-datum; pas na volledige invulling kan de aanvraag worden verzonden via beveiligde verbinding.

### REQ-007: Beveiligde opslag en beperkte toegang LBB-advies

GIVEN een ontvangen LBB-advies
WHEN dit advies in het dossier wordt geupload
THEN slaat het systeem het op in een sterk beveiligde container met toegang voor uitsluitend de gemandateerde bestuurder, de BIBOB-coordinator en de jurist; elke raadpleging wordt gelogd met datum, tijd, gebruiker en doel; het advies kan nooit als bijlage worden meegestuurd in een uitgaand bericht.

### REQ-008: Bestuursbesluit met Awb-motiveringsplicht

GIVEN een afgerond onderzoek met conclusie "lichte-mate" of "ernstige-mate-van-gevaar"
WHEN de bestuurder een besluit neemt
THEN biedt het systeem een besluit-template met verplichte onderdelen conform Awb artikel 3:46: feitelijke grondslag, juridische grondslag, belangenafweging, proportionaliteitstoets, en de daaruit voortvloeiende conclusie; het besluit kan niet definitief worden zonder dat alle vier onderdelen substantieel zijn ingevuld (minimum tekst-lengte per blok).

### REQ-009: Bezwaartermijn-bewaking en uitreiking

GIVEN een definitief BIBOB-besluit
WHEN het besluit wordt vastgesteld
THEN registreert het systeem de uitreikingsdatum, berekent de zes-weken-bezwaartermijn, plant herinneringen voor de jurist, en logt eventuele binnenkomende bezwaarschriften gekoppeld aan het oorspronkelijke dossier zodat het volledige verloop later reproduceerbaar is.

### REQ-010: Automatische bewaarregels en vernietiging

GIVEN een gesloten BIBOB-dossier
WHEN de wettelijke bewaartermijn (5 jaar na onherroepelijk besluit) verstrijkt
THEN start het systeem een vernietigingsworkflow: anonimiseren van persoonsgegevens, verwijderen van LBB-advies, behouden van geanonimiseerde besluit-statistieken voor verantwoordingsdoeleinden; de vernietiging vereist tweede-handtekening van de archivaris en wordt onomkeerbaar gelogd.

## Cross-app

- **purchaseq base** levert de opdracht/leverancier-context die de aanleiding vormt.
- **openconnector kvk-adapter** voor automatische verrijking met KvK-data en UBO-register-koppeling.
- **decidesk besluitvorming** voor de bestuursbesluit-workflow met digitale ondertekening en mandaatregister.
- **docudesk** voor dossiervorming en de uiteindelijke vernietigingsworkflow met archivaris-betrokkenheid.
- **openzaak** voor het registreren van het BIBOB-onderzoek als zaak met bijbehorende zaaktype en bewaartermijn.
- **shillinq** voor het blokkeren van eventuele betalingen aan een leverancier waar een lopend BIBOB-onderzoek met "lichte-mate" of "ernstige-mate" conclusie loopt.
