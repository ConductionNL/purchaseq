---
status: draft
---
# Raamovereenkomst en Minicompetitie

## Placement & Information Architecture

**Placement type:** `SUB_PAGE` — Sub-page beneath a top-level menu entry. Renders as a page inside the parent surface (usually reachable via a router child route or a tab on the parent index page).

**Lives at:** Aanbestedingen > Raamovereenkomsten/Minicompetities

**Rationale:** framework lots  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Purpose

The Raamovereenkomst & Minicompetitie capability gives purchaseq the lifecycle and workflow for raamovereenkomsten (framework agreements, FA in EU jargon) and the minicompetitie-mechaniek waarmee specifieke opdrachten binnen die raamovereenkomsten worden geplaatst. Raamovereenkomsten zijn in Nederland het dominante mechanisme voor herhaaldelijke inkoop met meerdere leveranciers: gemeenten gebruiken ze voor inhuur (Dynamisch Aankoopsysteem of klassieke raamovereenkomst met top-3), centrale inkooporganisaties (zoals de Belastingdienst, Rijkswaterstaat, of CMI-DICTU) gebruiken ze voor IT-hardware, software-licenties en advies, en regionale samenwerkingsverbanden gebruiken ze voor jeugdzorg, WMO-voorzieningen, leerlingenvervoer en groenonderhoud. De juridische basis ligt in AW2012 art 2.140-2.144 (raamovereenkomsten in klassieke sectoren), art 3.50-3.55 (speciale sectoren) en de EU-richtlijn 2014/24/EU art 33; deze bepalen de maximale looptijd van vier jaar (acht jaar in speciale sectoren), het verbod op wezenlijke wijziging, en de keuze tussen drie modaliteiten: raamovereenkomst met een enkele leverancier (alle opdrachten gaan direct daarheen), raamovereenkomst met meerdere leveranciers en volledig vastgelegde voorwaarden (waterval / cascade / verdeelsleutel), en raamovereenkomst met meerdere leveranciers waar niet alle voorwaarden vastliggen (en dus minicompetitie verplicht is). De capability dekt alle drie de modaliteiten maar focust zwaar op de derde, omdat die de meest complexe workflow en de meeste foutkans heeft. De capability levert (a) een raamovereenkomst-object met deelnemende leveranciers, percelen, looptijd, maximumvolume in EUR en in aantal opdrachten, verlengingsopties, en de minicompetitie-spelregels; (b) een minicompetitie-workflow van trigger → samenstelling van uit te nodigen leveranciers → publicatie van de uitnodiging → ontvangst van inschrijvingen → beoordeling tegen het gunningskader → gunningsbeslissing → contract; (c) volume-bewaking met realtime-aggregatie van bestede vs maximaal volume per leverancier en in totaal; (d) verlengingsmanagement met automatische herinneringen, herbeoordeling tegen de oorspronkelijke selectiecriteria, en optionele heronderhandeling; (e) een gestandaardiseerd score-systeem met meerdere gunningsmodellen (laagste prijs, EMVI op prijs+kwaliteit, EMVI met duurzaamheid, gunnen op beste kwaliteit binnen prijsplafond) en de mogelijkheid om die per minicompetitie aan te passen binnen de marges van de raamovereenkomst; (f) volledige documentatie van iedere minicompetitie inclusief proces-verbalen, beoordelingsmatrices en gunningsbeslissingen, met juridische bewaartermijn. De capability is een purchaseq-zustercapability van `tenderned-publicatie-adapter`: het opzetten van de raamovereenkomst zelf is een aanbesteding die via TenderNed wordt gepubliceerd (en die flow leeft daar), maar de minicompetities daarbinnen worden meestal niet op TenderNed gepubliceerd (alleen de gunningen, en alleen als ze boven de drempel uitkomen). De business logic, de auditing en de UI van minicompetities zijn substantieel genoeg om een eigen capability te rechtvaardigen.

## Data Model

**Schema `raamovereenkomst`** (new) is the root object. Fields: `nummer` (intern raamovereenkomst-kenmerk, e.g. RO-2024-IT-001), `titel`, `omschrijving`, `modaliteit` (enum: enkele_leverancier, meerdere_voorwaarden_vast, meerdere_minicompetitie), `categorie` (enum: leveringen, diensten, werken, sociale_diensten), `cpvHoofdcode`, `cpvBijcodes[]`, `aanbesteder` (ref to organisation), `startDatum`, `eindDatum`, `verlengingsopties` (array of {duur_maanden, eenmalig_boolean, max_aantal_keer}), `looptijdJaren` (computed, max 4 of 8), `maximumVolumeEur`, `maximumAantalOpdrachten` (optional), `verbruiktVolumeEur` (computed), `gebruiktePercelen[]`, `geraamdJaarlijksVolume`, `oorspronkelijkeAanbestedingId` (ref to base aanbesteding), `tendernedPublicatieId` (ref to publicatie record from adapter), `status` (concept, gepubliceerd, actief, in_verlenging, geeindigd, opgezegd), `eindigingsReden`, `documentRefs` (raamovereenkomst-document, algemene voorwaarden, programma van eisen, gunningsbeslissing).

**Schema `raamovereenkomst_leverancier`** (new) - one row per deelnemende leverancier per raamovereenkomst. Fields: `raamovereenkomstId`, `leverancierKvK`, `leverancierNaam`, `leverancierVestigingsadres`, `contactpersoon` (naam, email, telefoon), `perceel` (optional, if per perceel multiple winnaars), `rangorde` (optional, 1..n voor cascade-modaliteit), `aandeelPercentage` (optional, voor verdeelsleutel-modaliteit), `geschiktheidsbewijzenRefs[]` (UEA, VOG, beroepsbevoegdheid, financiele draagkracht, technische bekwaamheid), `verzekeringspolisRefs[]`, `actief` (boolean - tijdens looptijd kan een leverancier wegvallen), `wegvalDatum`, `wegvalReden`, `verbruiktVolumePerLeverancierEur` (computed), `gegundeMinicompetities` (count, computed).

**Schema `minicompetitie`** (new) is the per-opdracht workflow object. Fields: `raamovereenkomstId`, `nummer` (auto, e.g. MC-2024-IT-001-042), `titel`, `omschrijving`, `aanvragendeAfdeling`, `aanvragendeUserRef`, `triggerType` (enum: nieuwe_behoefte, aflopend_contract, vervangingsinvestering, projectopdracht), `perceel` (optional), `geraamdeWaardeEur`, `gunningsmodel` (enum: laagste_prijs, emvi_prijs_kwaliteit, emvi_prijs_kwaliteit_duurzaamheid, beste_kwaliteit_binnen_plafond), `prijsplafondEur` (alleen bij beste_kwaliteit_binnen_plafond), `wegingsschema` (object: {prijs_percentage, kwaliteit_percentage, duurzaamheid_percentage}), `kwaliteitscriteria[]` (array of {naam, omschrijving, max_punten, weging_percentage}), `duurzaamheidscriteria[]` (uit MVI-capability), `inschrijvingsTermijnDagen`, `publicatieDatum`, `sluitingsDatum`, `uitgenodigdeLeveranciers[]` (refs to raamovereenkomst_leverancier), `selectieReden` (waarom deze subset, indien niet alle deelnemers worden uitgenodigd), `status` (concept, uitgenodigd, ontvangst_inschrijvingen, in_beoordeling, gegund, gestaakt), `gunningsbeslissingRef`, `winnaarRef`, `gunningsbedragEur`, `contractRef`, `beoordelingsCommissie[]` (user refs, min 3 personen vereist voor objectiviteit).

**Schema `minicompetitie_inschrijving`** (new). Fields: `minicompetitieId`, `leverancierRef`, `ingediendDatum`, `inschrijvingsBedragEur`, `prijsOpbouw` (line items optional), `kwaliteitscores[]` (per criterium {criteriumId, score, motivering}), `duurzaamheidscores[]`, `totaalScore` (computed), `documentRefs[]` (offerte, prijsspecificatie, kwaliteitsantwoorden, plan van aanpak), `tijdig` (boolean), `geldigVerklaard` (boolean), `geldigheidsReden`.

**Schema `minicompetitie_beoordeling`** (new) - per beoordelaar per criterium. Fields: `inschrijvingId`, `beoordelaarRef`, `criteriumId`, `score`, `motivering`, `datum`; aggregation logic (gemiddeld, mediaan, consensus na bespreking) lives in service.

**Schema `raamovereenkomst_verlenging`** (new). Fields: `raamovereenkomstId`, `verlengingsnummer`, `voorgenomenIngangsdatum`, `voorgenomenDuurMaanden`, `motivering`, `besluit` (enum: in_voorbereiding, akkoord, geweigerd, met_voorwaarden), `besluitDoor`, `besluitDatum`, `voorwaarden` (text), `nieuweEindDatum`.

**Schema `minicompetitie_proces_verbaal`** (new) - PV's per fase. Fields: `minicompetitieId`, `pvType` (enum: opening_inschrijvingen, beoordeling, gunningsadvies), `datum`, `aanwezigen[]`, `inhoud` (text), `bijlageRef` (signed PDF).

## Requirements

**REQ-RMC-001 - Raamovereenkomst aanmaken vanuit gegunde aanbesteding**
The system MUST allow promotion of a gegunde aanbesteding (in werkproces-status `gegund`) to a `raamovereenkomst` object, copying perceelindeling, gegunde leveranciers, looptijd, maximumvolume en gunningskader.
- GIVEN een gegunde aanbesteding met 3 percelen en 5 winnaars (perceel A: 1 winnaar, perceel B: 2 winnaars, perceel C: 2 winnaars), WHEN de inkoper "Promote naar raamovereenkomst" kiest, THEN er wordt 1 raamovereenkomst met modaliteit `meerdere_minicompetitie` gemaakt, 5 `raamovereenkomst_leverancier`-rijen aangemaakt met perceel-referentie, en de looptijd uit de aanbesteding overgenomen.
- GIVEN een gegunde aanbesteding met een looptijd van 6 jaar voor de klassieke sector, WHEN promotion plaatsvindt, THEN het systeem weigert met "Looptijd overschrijdt 4-jaar limiet AW2012 art 2.140; vereist motivering voor uitzondering"; de inkoper kan onder de uitzonderingsgrond verlengen tot maximaal de wettelijke uitzondering met expliciete motivering.
- GIVEN een aanbesteding voor speciale-sector-aanbesteder, WHEN promotion plaatsvindt, THEN geldt de 8-jaar maximum looptijd.

**REQ-RMC-002 - Minicompetitie-trigger en uitnodigingen-selectie**
The system MUST support triggering een minicompetitie vanuit (a) handmatige aanvraag door een afdeling, (b) aflopend onderliggend contract met automatische trigger 90 dagen voor einde, of (c) een budget-aanvraag uit een gekoppelde planning. De selectie van uit te nodigen leveranciers MUST default-gewijs alle actieve raamovereenkomst-leveranciers in het relevante perceel kiezen, en MUST motivering eisen als de inkoper een subset selecteert.
- GIVEN een raamovereenkomst voor IT-inhuur met 8 leveranciers in perceel "Java-ontwikkelaars", WHEN een nieuwe minicompetitie wordt gestart, THEN alle 8 actieve leveranciers staan voor-geselecteerd; deselectie van een leverancier vereist een verplichte `selectieReden` met minimaal 50 karakters.
- GIVEN een onderliggend contract dat afloopt op 31 december, WHEN de scheduled trigger op 2 oktober draait, THEN een concept-minicompetitie wordt aangemaakt met titel "Verlenging/vervanging van contract X", aanvrager = oorspronkelijke contracteigenaar, en een notificatie gaat naar de inkoper.
- GIVEN een leverancier waarvan de actief-vlag op false staat (bijvoorbeeld faillissement, opzegging), WHEN minicompetitie-selectie draait, THEN die leverancier verschijnt niet in de lijst en kan niet handmatig worden toegevoegd zonder admin-override.

**REQ-RMC-003 - Gunningsmodel per minicompetitie binnen raamovereenkomst-marges**
The system MUST allow per-minicompetitie keuze van gunningsmodel uit de set die door de raamovereenkomst is toegestaan, en MUST de wegingsverhouding (prijs/kwaliteit/duurzaamheid) configureren met validatie dat alle percentages bij elkaar 100% opleveren en dat geen criterium onder een minimum-percentage van 10% komt (proportionaliteitsbeginsel).
- GIVEN een raamovereenkomst die de modellen `emvi_prijs_kwaliteit` en `emvi_prijs_kwaliteit_duurzaamheid` toestaat, WHEN een minicompetitie model `laagste_prijs` probeert te kiezen, THEN het systeem weigert met "Gunningsmodel niet toegestaan binnen deze raamovereenkomst".
- GIVEN een EMVI-model met weging prijs 60%, kwaliteit 35%, duurzaamheid 4%, WHEN opslaan, THEN validatie faalt (duurzaamheid < 10%) en de inkoper krijgt advies om duurzaamheid op minimaal 10% te zetten of het criterium te schrappen.
- GIVEN een EMVI-model met weging prijs 50%, kwaliteit 30%, duurzaamheid 20% (totaal 100%), WHEN opslaan, THEN validatie slaagt.

**REQ-RMC-004 - Uitnodiging-publicatie en bilaterale documentuitwisseling**
The system MUST publish minicompetitie-uitnodigingen via secure channel (encrypted email of secure portal link) naar de geselecteerde leveranciers, en MUST een tweezijdig vragenkanaal bieden waarbij vragen door de inkoper geanonimiseerd worden en alle antwoorden naar alle uitgenodigde leveranciers gaan.
- GIVEN 5 uitgenodigde leveranciers, WHEN de inkoper "Verstuur uitnodigingen" kiest, THEN elke leverancier krijgt een uniek e-mailbericht met deeplink naar een leveranciers-portal, alle inschrijvingsdocumenten (PvE, conceptcontract, prijsformulier-template) zijn downloadbaar, en de inkoper ziet een ontvangst-bevestiging per leverancier.
- GIVEN een vraag van leverancier 2 over de PvE, WHEN de inkoper "Beantwoorden" kiest, THEN het antwoord wordt geanonimiseerd opgeslagen, alle 5 leveranciers ontvangen het Q&A-paar, en het Q&A-document wordt aan de openbare-tijdens-procedure documentenset toegevoegd.
- GIVEN een leverancier die antwoordt na de officiele Q&A-sluiting, WHEN de vraag binnenkomt, THEN de inkoper kan kiezen tussen "Beantwoord niet (te laat)" of "Beantwoord toch (vereist motivering en verlenging van inschrijvingsdeadline)".

**REQ-RMC-005 - Inschrijvingen ontvangen en sluiten op tijdstip**
The system MUST inschrijvingen accepteren tot exact de sluitingsDatum (tot op de seconde), MUST late inschrijvingen weigeren met automatische rejectie-bericht en motivering, en MUST de inschrijvingen pas openen na het officiele openingsmoment door de beoordelingscommissie (vier-ogen-principe).
- GIVEN een minicompetitie met sluitingsDatum 2026-06-01 15:00:00, WHEN een leverancier op 2026-06-01 15:00:01 wil inschrijven, THEN het systeem weigert met een rejectie-bericht inclusief server-timestamp en de leverancier ziet "Te laat: deadline overschreden".
- GIVEN drie inschrijvingen ontvangen op tijd, WHEN voor de openingsdatum een lid van de beoordelingscommissie de inschrijvingen probeert in te zien, THEN het systeem weigert en logt de toegangspoging; openen vereist twee commissieleden gelijktijdig aanwezig (digitaal: dual-signature).
- GIVEN openingsmoment is bereikt, WHEN twee commissieleden tegelijk autoriseren, THEN alle inschrijvingen worden ontsleuteld, een PV van opening wordt automatisch gegenereerd met overzicht van inschrijvers, bedragen en tijdstippen, ondertekend door beide commissieleden.

**REQ-RMC-006 - Beoordelingsproces met meerdere beoordelaars en consensus**
The system MUST elk kwaliteitscriterium laten beoordelen door minimaal 3 commissieleden onafhankelijk, MUST individuele scores en motiveringen vastleggen, MUST verschillen >2 punten (op een schaal van 10) flaggen voor bespreking, en MUST een eindscore vastleggen op basis van consensus na bespreking (niet automatisch gemiddelde).
- GIVEN een minicompetitie met 4 kwaliteitscriteria en 5 inschrijvingen en 3 commissieleden, WHEN beoordeling start, THEN het systeem genereert 60 (4x5x3) `minicompetitie_beoordeling`-rijen, alle initieel leeg; elk commissielid kan alleen zijn/haar eigen rijen invullen.
- GIVEN commissielid A geeft inschrijving 1 voor criterium "Plan van aanpak" een 8, commissielid B een 4, commissielid C een 7, WHEN het systeem het verschil ziet (>2 tussen min en max), THEN het criterium wordt geflaggd, een bespreking-actie wordt aangemaakt, en de eindscore blijft `null` tot consensus is vastgelegd via een aparte consensus-rondes-UI met motivering.
- GIVEN alle criteria zijn beoordeeld en consensus is bereikt, WHEN "Bereken eindscore" wordt geklikt, THEN per inschrijving worden gewogen scores berekend, een rangorde wordt vastgesteld, en het systeem genereert een beoordelings-PV (`pvType = beoordeling`).

**REQ-RMC-007 - Volume-bewaking real-time tegen raamovereenkomst-maximum**
The system MUST per minicompetitie-gunning het verbruikte volume bijhouden, per leverancier en in totaal, MUST waarschuwen wanneer 70%, 85% en 95% van het maximumVolumeEur is bereikt, en MUST de gunning blokkeren wanneer een nieuwe minicompetitie het maximum zou overschrijden zonder voorafgaande wijzigings-aankondiging op TenderNed.
- GIVEN een raamovereenkomst met maximumVolumeEur 2.000.000 en `verbruiktVolumeEur` 1.700.000 (85%), WHEN een nieuwe minicompetitie wordt gestart, THEN de inkoper krijgt een banner "85% van het maximumvolume verbruikt; gemiddeld nog ruimte voor X minicompetities" en de raamovereenkomst-eigenaar ontvangt een email.
- GIVEN dezelfde raamovereenkomst met 1.950.000 verbruikt, WHEN een gunning van 100.000 wordt voorbereid, THEN het systeem blokkeert de gunning met "Maximumvolume zou worden overschreden; vereist wijzigingsaankondiging via TenderNed (zie capability `tenderned-publicatie-adapter`) of staking van deze minicompetitie".
- GIVEN per-leverancier-maxima zijn opgegeven (bv. leverancier X max 500k binnen de RO), WHEN een gunning aan X de 500k zou overschrijden, THEN het systeem blokkeert en stelt voor om de minicompetitie aan een andere leverancier te gunnen of de minicompetitie te splitsen.

**REQ-RMC-008 - Verlengingsmanagement met automatische herinneringen**
The system MUST 180, 120 en 60 dagen voor de eindDatum van een raamovereenkomst notificaties sturen naar de raamovereenkomst-eigenaar over verlengingsbeslissing, MUST een verlengings-werkproces ondersteunen met motivering, mandaat-check, en optioneel een lichte heronderhandeling (prijsindexatie, scope-bevestiging) zonder wezenlijke wijziging.
- GIVEN een raamovereenkomst eindigend op 2026-12-31 met `verlengingsopties = [{duur_maanden: 12, eenmalig: false, max_aantal_keer: 2}]`, WHEN de scheduled job draait op 2026-07-04 (180 dagen voor), THEN een notificatie gaat naar de eigenaar met de optie om een `raamovereenkomst_verlenging` aan te maken.
- GIVEN een verlenging in voorbereiding, WHEN de eigenaar prijsindexatie van 2,3% (CPI-koppeling) en geactualiseerde KPI-bijlage wil opnemen, THEN het systeem markeert dit als toegestaan (proportionele wijziging onder AW2012 art 2.163d) en logt de motivering.
- GIVEN een verlengingspoging waarvoor de oorspronkelijke verlengingsopties al maximaal zijn benut, WHEN de eigenaar toch wil verlengen, THEN het systeem weigert en wijst op de noodzaak een nieuwe aanbesteding op te starten (raamovereenkomst kan niet wezenlijk worden gewijzigd om looptijd te verlengen).

**REQ-RMC-009 - Gunningsbeslissing met automatisch motiveringsdocument**
The system MUST een gunningsbeslissing-document genereren op basis van het beoordelingsproces, inclusief samenvatting van scores, motivering van keuze, verliezer-feedback (Alcatel-termijn van 20 dagen vereist), en MUST bezwaartermijn-bewaking activeren voordat het contract definitief wordt.
- GIVEN een minicompetitie met rangorde [winnaar, nr2, nr3], WHEN "Genereer gunningsbeslissing" wordt geklikt, THEN een PDF wordt gemaakt met (a) anonieme rangorde voor verliezers (b) volledige score-toelichting per verliezer over hun eigen inschrijving (c) motivering waarom de winnaar wint (d) Alcatel-clausule met datum waarop bezwaartermijn afloopt.
- GIVEN een gunningsbeslissing is verzonden naar alle inschrijvers, WHEN de 20-dagen Alcatel-termijn loopt, THEN het systeem blokkeert definitieve contractering tot na deze termijn (tenzij minicompetitie onder de drempel valt en partijen schriftelijk afstand doen van bezwaartermijn).
- GIVEN een verliezer dient binnen de Alcatel-termijn bezwaar in via kort geding, WHEN dat in purchaseq wordt geregistreerd, THEN de gunning gaat naar status `bezwaar_ingesteld`, contractering wordt opgeschort, en de inkoper krijgt een juridisch-advies workflow.

**REQ-RMC-010 - Volledige proces-verbaal en audit-trail per minicompetitie**
The system MUST per minicompetitie automatisch drie proces-verbalen genereren (opening_inschrijvingen, beoordeling, gunningsadvies), MUST ze digitaal ondertekenen door de relevante commissieleden, MUST een complete audittrail van wie wat wanneer deed bijhouden, en MUST de dossierset 7 jaar bewaren conform AW2012 art 4.13.
- GIVEN een opening van inschrijvingen door commissieleden X, Y, Z, WHEN het PV automatisch wordt gegenereerd, THEN het PV bevat datum/tijd, namen aanwezigen, inschrijvers, bedragen, bijzonderheden, en wordt door alle drie digitaal ondertekend voordat het als immuable PDF wordt opgeslagen.
- GIVEN een audit-aanvraag voor "alle minicompetities binnen RO-2024-IT-001 in Q2 2025", WHEN een auditor de export-endpoint aanroept, THEN een ZIP wordt gegenereerd met per minicompetitie: uitnodiging, inschrijvingen, beoordelingen per beoordelaar, drie PV's, gunningsbeslissing, en audit-log van mutaties.
- GIVEN een minicompetitie-record ouder dan 7 jaar, WHEN purge-job draait, THEN het record wordt gemarkeerd als `bewaartermijn_verlopen` en kan alleen met expliciete admin-actie + tweede-factor worden verwijderd.

## Standards & Sources

- **AanbestedingsWet 2012** art 2.140-2.144 (raamovereenkomsten klassieke sectoren), 3.50-3.55 (speciale sectoren), 2.163a-g (wijzigingen tijdens looptijd), 2.130 (gunningsbeslissing en motivering), 4.13 (bewaartermijn).
- **EU-richtlijn 2014/24/EU** art 33 (framework agreements) - de Europese basis; art 34 (DAS - Dynamisch Aankoopsysteem, dat verwant is maar buiten scope van deze capability valt).
- **Gids Proportionaliteit** (3e versie, januari 2022) voor de afweging wanneer een raamovereenkomst proportioneel is en wanneer een DAS of een herhaalde individuele opdracht beter past.
- **PIANOo handreiking Raamovereenkomsten** (2020, periodiek geupdate) - de meest gehanteerde praktische gids in NL, met patronen voor cascade, verdeelsleutel en minicompetitie.
- **PIANOo handreiking minicompetities** voor specifiek de minicompetitie-fase.
- **AW2012 motivering en de Alcatel-rechtspraak** (Alcatel/Bundesministerium HvJ-EU C-81/98 en Hoge Raad-jurisprudentie) voor de 20-dagen-bezwaartermijn na gunningsbeslissing.
- **Wet Markt en Overheid** voor de proportionaliteits-afweging bij MKB-vriendelijkheid binnen de leveranciers-selectie.
- **NEN-EN-ISO 9001/14001** als referentie voor kwaliteitscriteria-formulering binnen EMVI.
- **CBS Consumentenprijsindex (CPI)** voor prijsindexatie bij verlengingen.
- **Best Value Procurement-methodiek** (Kashiwagi, vertaald door Rijkswaterstaat / Best Value Nederland) als alternatief gunningsmodel voor dienst-aanbestedingen; de capability ondersteunt het via `gunningsmodel = beste_kwaliteit_binnen_plafond`.
- **EU Court of Justice, C-216/17 (Asmel)** voor de uitleg dat raamovereenkomsten een maximumvolume verplicht moeten kennen.

## Cross-app integration

- **tenderned-publicatie-adapter** (purchaseq zustercapability): de oorspronkelijke aanbesteding voor de raamovereenkomst wordt via die adapter gepubliceerd; gunningsaankondiging voor de RO komt daarvandaan; wijzigingsaankondigingen bij volume-overschrijding of leverancier-wegval lopen ook via die adapter. Minicompetities zelf worden meestal niet gepubliceerd, maar boven-drempel-gunningen wel.
- **aanbesteding-werkproces** (purchaseq base): de oorspronkelijke aanbesteding leeft daar; promotion naar raamovereenkomst is een werkproces-transitie.
- **mvi-sroi-aanbesteding** (purchaseq zustercapability): MVI- en SROI-criteria die in de raamovereenkomst zijn afgesproken worden per minicompetitie overgenomen als default-duurzaamheidscriteria; de inkoper kan ze niet versoepelen onder het RO-minimum.
- **openconnector**: secure-email-versturing voor uitnodigingen via een SMTP- of API-source met DKIM/DMARC; encrypted-attachment via S/MIME of beveiligde portal-link.
- **openregister**: dossiermodel; audit-trail capability voor de PV's en log; file-attachment voor documenten.
- **docudesk**: PV-templates en gunningsbeslissing-templates; documentgeneratie met digital-signing integratie.
- **opentalk** (ExApp): videovergadering voor beoordelingscommissie-bijeenkomsten (consensus-bespreking) met automatische deelname-registratie als PV-aanhangsel.
- **mydash**: KPI-tegels voor "raamovereenkomsten in laatste-jaar-fase", "verbruikspercentage per RO", "doorlooptijd minicompetitie", "gemiddeld aantal inschrijvingen per minicompetitie", "MKB-aandeel in raamovereenkomsten".
- **n8n / pipelinq**: scheduled jobs voor verlenging-notificaties, contract-afloop-triggers, volume-rapportages aan bestuur.
- **larpingapp**: (optioneel) gamified onboarding voor nieuwe inkopers door fictieve minicompetities te oefenen.

## Target users

- **Inkoper / contractmanager**: dagelijkse gebruiker; voert minicompetities uit, bewaakt volume, plant verlengingen.
- **Categoriemanager / portfolio-inkoper**: bewaakt de raamovereenkomst-portefeuille op categorie-niveau (IT, faciliteiten, advies); zorgt voor strategie-conformiteit.
- **Aanvragende afdelingsmanager**: triggert minicompetities namens de business unit; voert kwaliteitscriteria aan; zit in beoordelingscommissie als gebruikersvertegenwoordiger.
- **Beoordelingscommissie-leden**: minimaal 3 personen per minicompetitie; vullen scores in, doen consensus-besprekingen, tekenen PV's.
- **Contractmanager / leveranciersmanager**: monitort de gegunde minicompetities, prestaties van leveranciers, en signaleert input voor RO-verlengingsbesluiten.
- **Juridisch adviseur**: kijkt mee bij grensgevallen (wezenlijke wijziging, bezwaar binnen Alcatel-termijn).
- **Compliance / interne controle**: gebruikt audit-trail voor rechtmatigheidstoetsing.
- **Bestuur / college / directie**: ontvangt periodieke rapportages over raamovereenkomst-portefeuille en grootschalige minicompetities boven mandaatdrempel.
- **Leveranciers in de raamovereenkomst**: indirecte gebruiker; ontvangen uitnodigingen, dienen offertes in via de portal, krijgen gunningsbeslissingen.
