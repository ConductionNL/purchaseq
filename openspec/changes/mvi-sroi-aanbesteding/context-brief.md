---
status: draft
---
# MVI- en SROI-eisen in Aanbestedingen

## Placement & Information Architecture

**Placement type:** `SUB_PAGE` — Sub-page beneath a top-level menu entry. Renders as a page inside the parent surface (usually reachable via a router child route or a tab on the parent index page).

**Lives at:** Aanbestedingen > MVI-SROI-eisen

**Rationale:** sustainability criteria branch  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Purpose

The MVI & SROI Aanbesteding capability gives purchaseq the structured framework voor het opnemen, scoren, monitoren en rapporteren van Maatschappelijk Verantwoord Inkopen (MVI) en Social Return on Investment (SROI) in elke aanbesteding die de organisatie uitvoert. MVI en SROI zijn in Nederland niet langer optioneel: het Manifest Maatschappelijk Verantwoord Opdrachtgeven en Inkopen (MVOI) is door alle decentrale overheden plus het Rijk ondertekend (eerste versie 2016, hernieuwd 2023-2026 als MVOI 2026), de Aanbestedingswet 2012 verplicht in art 1.4 het in acht nemen van milieu- en sociale aspecten, en specifieke instrumenten zoals de CO2-Prestatieladder, PSO-trede, en de RVO MVI-Criteriatool zijn standaardonderdeel van het inkoopproces geworden. Voor sociale return geldt sinds 2011 de afspraak van een minimum-SROI-percentage van 2% tot 5% van de opdrachtwaarde bij opdrachten boven EUR 250.000 voor diensten en EUR 500.000 voor werken (per regio en sector afwijkend), en de Bouwblokkenmethode (uniform NL-model voor SROI-verantwoording sinds 2018, beheerd door TNO/PSO Nederland). De capability moet vier dingen leveren: (a) een rijke MVI-criteria-bibliotheek per inkoopcategorie, gevoed door de RVO MVI-criteriatool plus organisatie-eigen aanvullingen, met onderscheid tussen minimumeisen (knock-out), gunningscriteria (gewogen punten) en contracteisen (post-award monitoring); (b) een SROI-rekenmodule die per opdracht de verplichting bepaalt (percentage van opdrachtwaarde), bouwblokken-invulling toelaat (uren x weegfactor per doelgroep), en gegunde SROI-prestaties op de leveranciers monitort; (c) een dashboard voor portefeuille-rapportage waarmee de inkooporganisatie kan aantonen welk percentage van uitgaven MVI-conform is, hoeveel SROI is gerealiseerd, en op welke MVOI-thema's nog progressie nodig is (richting de gestelde doelen, vaak CO2-reductie 49% in 2030 en circulair 50% in 2030); (d) rapportage-uitvoer geschikt voor het college van B&W, de Raad/Staten, het MVO-jaarverslag, en de jaarlijkse uitvraag van Rijksoverheid/RIVM voor het MVOI-monitorbestand. De capability is bewust eigen omdat MVI/SROI dwars door alle andere capabilities heenloopt: het beinvloedt de bestek-tekst (gunningscriteria), de gunning-beslissing (scores), het contract (boetes en bonussen op MVI-prestatie), en de uitvoering (kwartaalrapportages SROI-realisatie). Door deze logica te concentreren in een aparte capability wordt voorkomen dat MVI-implementatie versnipperd raakt over template-tekstfragmenten en handmatige Excel-trackers (de huidige praktijk bij veel organisaties), en kan de organisatie centraal sturen op MVI/SROI-strategie.

## Data Model

**Schema `mvi_criterium`** (new, referentie + organisatie-eigen). Fields: `code` (e.g. RVO-15-CO2-1 of ORG-CIRC-001), `bron` (enum: rvo_criteriatool, cpv_klimaatakkoord, organisatie_eigen, cirkelstad, rijk_mvoi), `categorie` (enum: klimaat_co2, milieu_overig, social_arbeid, social_inclusie, circulair, biodiversiteit, dierenwelzijn, internationaal_iso26000, mkb_friendly, lokale_economie), `inkoopcategorie` (enum koppeling met RVO categorieen: ICT-hardware, ICT-software, kantoormeubilair, bedrijfskleding, catering, schoonmaak, vervoer, energie, bouw, ingenieursdiensten, advies, jeugdzorg, ...), `titel`, `omschrijving`, `niveau` (enum: basis, significant, ambitieus - per RVO drie-trappen-model), `eisType` (enum: minimumeis, gunningscriterium, contracteis), `meetbaarheid` (enum: ja_meetbaar, kwalitatief, certificaat_bewijs), `bewijsmiddelen[]` (enum: co2_prestatieladder_niveau, pso_trede, milieukeur, msc_label, fsc_label, cradle_to_cradle, isae_3000, eigen_verklaring, zelfverklaring_met_audit_recht), `geldigVanaf`, `geldigTot`, `versie`.

**Schema `aanbesteding_mvi_set`** (new) - per aanbesteding de geselecteerde MVI-criteria. Fields: `aanbestedingId`, `criteriumRef`, `eisType` (kan afwijken van bron-criterium), `weging` (alleen bij gunningscriteria), `minimumNiveau` (basis, significant, ambitieus), `verplichtBewijsmiddel`, `formuleringInBestek` (text, default uit criterium maar overschrijfbaar), `motivering` (waarom dit criterium in deze aanbesteding), `geintegreerdIn` (enum: gunningskader, programma_van_eisen, conceptcontract).

**Schema `sroi_verplichting`** (new) - per opdracht de SROI-verplichting. Fields: `aanbestedingId`, `contractRef`, `opdrachtwaardeEur`, `sroiPercentage` (default uit regio-policy, e.g. 2.0 of 5.0), `sroiBedragEur` (computed), `peilDatum`, `regiOpolicy` (regio-specifieke regelset zoals "Regio Rijnmond SROI 2024" of "PSW Den Haag"), `bouwblokkenModel` (enum: bouwblokken_2018, bouwblokken_2023, eigen_model), `realisatieperiode` (start/eind), `tussentijdseRapportagesFrequentie` (enum: maandelijks, per_kwartaal, halfjaarlijks), `eindrapportageDatum`, `status` (concept, vastgelegd, in_uitvoering, gerealiseerd, niet_gerealiseerd, vrijgesteld).

**Schema `sroi_bouwblok`** (new, referentie) - de uniforme Bouwblokkenmethode: e.g. "WW-uitkering > 12 mnd, plaatsing 32+ uur per week" weegt EUR 35.000 per FTE per jaar. Fields: `code` (e.g. BB-WW-12), `categorie` (uitkeringsgerechtigden, WW, WIA, Wajong, Participatiewet, statushouders, ex-gedetineerden, leerwerkplek_bbl, leerwerkplek_bol, sociale_werkvoorziening, kwetsbare_jongeren, leerling_stage, scholing_omscholing, lokale_inkoop, MKB-inschakeling), `subcategorie`, `weegfactorEur` (per uur, per FTE-jaar, per geplaatste persoon), `voorwaarden` (text), `geldigVanaf`, `geldigTot`.

**Schema `sroi_realisatie`** (new) - de invulling van een SROI-verplichting. Fields: `verplichtingId`, `periode` (e.g. 2026-Q2), `bouwblokRef`, `aantalUren` (of `aantalFtes` of `aantalPersonen`), `gerealiseerdeWaardeEur` (computed = aantal x weegfactor), `bewijsmiddelen[]` (loonadministratie-extract, ondertekende plaatsingsverklaring, stagecontract), `verantwoordingsbron` (zelf-rapportage_leverancier, audit_door_aanbesteder, audit_door_pso, accountantsverklaring), `goedgekeurd` (boolean), `goedgekeurdDoor`, `opmerkingen`.

**Schema `mvi_monitoring_rapportage`** (new) - per contract per periode. Fields: `contractRef`, `periode`, `mviCriteriumRef`, `gerapporteerdeWaarde` (numeric of text), `bewijsmiddelRef`, `voldoetAanEis` (boolean), `afwijking` (text), `verbetervoorstel`, `boeteOpgelegdEur`, `bonusToegekendEur`.

**Schema `mvi_portfolio_doelstelling`** (new, per organisatie). Fields: `organisatieRef`, `thema` (enum: co2_reductie_percentage, circulair_percentage_uitgaven, sroi_realisatie_percentage, mkb_aandeel_percentage, biodiversiteit_index), `doeljaar` (e.g. 2030), `doelwaarde`, `baselineJaar`, `baselineWaarde`, `huidigeWaarde` (computed periodiek), `voortgang` (computed: percentage van traject afgerond), `bestuurlijkAkkoord` (ref to besluit).

## Requirements

**REQ-MVI-001 - MVI-criteria-bibliotheek importeren en up-to-date houden**
The system MUST de RVO MVI-criteriatool periodiek (minimaal kwartaaljks) importeren als bron-data voor `mvi_criterium`, MUST organisatie-eigen criteria mogelijk maken zonder de RVO-criteria te vervuilen, en MUST versiebeheer voeren zodat lopende aanbestedingen gebruik blijven maken van de versie die geldig was bij aanvang.
- GIVEN de RVO publiceert een nieuwe versie van het CO2-criterium voor ICT-hardware (van basis 250g CO2/kWh naar 200g), WHEN de import-job draait, THEN het oude criterium krijgt `geldigTot = importdatum`, een nieuw criterium met nieuwe versie wordt aangemaakt, en lopende aanbestedingen die het oude refereren behouden hun gekoppelde versie.
- GIVEN een inkoper wil een organisatie-eigen criterium "Glasvezel-leverancier met groene stroom-garantie" toevoegen, WHEN dat wordt opgeslagen, THEN het krijgt `bron = organisatie_eigen` en `code` met ORG-prefix, en verschijnt naast RVO-criteria in dezelfde categorie.
- GIVEN een import-job mislukt (RVO-endpoint onbereikbaar), WHEN de retry-logica is uitgeput, THEN de laatste succesvolle versie blijft actief, een ops-alert wordt opgevoerd, en lopende werkprocessen worden niet onderbroken.

**REQ-MVI-002 - Per inkoopcategorie standaard-MVI-set voorstellen**
The system MUST voor elke aanbesteding op basis van de inkoopcategorie (afgeleid uit CPV-hoofdcode of expliciet gekozen) een voorgestelde standaard-MVI-set genereren, gebaseerd op (a) de RVO-aanbevolen criteria voor die categorie, (b) de portfolio-doelstellingen van de organisatie, en (c) de gekozen ambitieniveau (basis, significant, ambitieus).
- GIVEN een aanbesteding voor ICT-hardware (CPV 30200000), WHEN de inkoper "Stel MVI-set voor" kiest met ambitie `significant`, THEN het systeem voegt automatisch toe: CO2-criterium niveau significant (gunningscriterium), Energy-Star verplicht (minimumeis), reparatie-recht-clausule (contracteis), reservedelen-beschikbaarheid 5 jaar (contracteis), en transport-CO2 (gunningscriterium).
- GIVEN dezelfde aanbesteding met ambitie `ambitieus`, WHEN voorstel wordt gegenereerd, THEN bovenop het significant-pakket komen: cradle-to-cradle-certificering (gunningscriterium), refurbished-aanbod (minimumeis 30% refurbished), en take-back-garantie (contracteis).
- GIVEN een aanbesteding met een inkoopcategorie waarvoor de organisatie een portfoliodoel "Circulair 50% in 2030" heeft, WHEN voorstel wordt gegenereerd, THEN circulaire criteria krijgen een hogere default-weging om bij te dragen aan dat portfoliodoel.

**REQ-MVI-003 - MVI-criteria scoren in EMVI-gunningskader**
The system MUST gunningscriterium-MVI-eisen geschikt formuleren voor scoring binnen het EMVI-model, MUST scoring-instructies aan de beoordelingscommissie geven, en MUST de behaalde scores per inschrijver gelinkt opslaan voor contract-monitoring.
- GIVEN een aanbesteding met gunningscriterium "CO2-Prestatieladder niveau", WHEN de inkoper de scoringsmatrix samenstelt, THEN het systeem stelt voor: niveau 5 = 100 punten, niveau 4 = 80, niveau 3 = 60, niveau 2 = 40, niveau 1 = 20, geen ladder = 0; dit is overschrijfbaar maar logged als afwijking.
- GIVEN drie inschrijvers met CO2-Prestatieladder niveau 4, 3 en geen, WHEN de beoordelingscommissie de scores invoert, THEN de scores worden gevalideerd tegen het bewijsmiddel (geupload ladder-certificaat met geldige expiratie-datum) en automatisch volgens de matrix gescoord, met handmatige override mogelijk + motivering.
- GIVEN een gegunde inschrijver met CO2-Prestatieladder niveau 4 in de offerte, WHEN het contract wordt aangemaakt, THEN de MVI-eisen worden gepromoveerd naar contracteisen met "niveau 4 te behouden gedurende looptijd"; downgrade tijdens uitvoering triggert een boeteclausule.

**REQ-MVI-004 - SROI-verplichting automatisch bepalen per opdracht**
The system MUST per aanbesteding de SROI-verplichting bepalen op basis van (a) opdrachtwaarde, (b) categorie (sommige zoals werken kennen ander percentage dan diensten), (c) regio-policy van de aanbesteder, en (d) eventuele vrijstellingsgronden, en MUST het bedrag in EUR en de invulopties tonen aan de inkoper.
- GIVEN een aanbesteding voor schoonmaakdiensten met geraamde waarde EUR 400.000 in de regio Rotterdam (policy: 5% SROI vanaf 250k diensten), WHEN de SROI-bepaling draait, THEN `sroi_verplichting` wordt aangemaakt met percentage 5%, bedrag EUR 20.000, en bouwblokkenmodel uit de regio-policy.
- GIVEN een aanbesteding voor specialistische ICT-software met waarde EUR 800.000 maar zonder fysieke arbeid in NL (cloud-licenties), WHEN de inkoper "Vrijstelling aanvragen" kiest, THEN een vrijstellingsverzoek wordt opgevoerd met motivering, en pas na akkoord van de SROI-coordinator wordt het verplichtingsbedrag op EUR 0 gezet.
- GIVEN een aanbesteding waarvan de waarde onder de regio-drempel valt (e.g. EUR 100.000 diensten), WHEN bepaling draait, THEN geen SROI-verplichting wordt opgelegd, maar de inkoper krijgt suggestie "Vrijwillige SROI toepassen?" met default-waarde 2%.

**REQ-MVI-005 - Bouwblokken-invulling door leverancier en goedkeuring**
The system MUST leveranciers laten rapporteren op SROI-realisatie via de bouwblokkenmethode, MUST de gerealiseerde waarde berekenen, MUST bewijsmiddelen verzamelen, en MUST een goedkeuringsworkflow door de SROI-coordinator van de aanbestedende organisatie bieden.
- GIVEN een SROI-verplichting van EUR 20.000 met realisatieperiode 1 jaar, WHEN de leverancier in Q2 rapporteert "2 personen WW-12+ geplaatst 32 uur/week voor 13 weken", THEN de gerealiseerde waarde wordt berekend via bouwblok BB-WW-12 weegfactor; de leverancier moet de loonadministratie-extract als bewijs uploaden.
- GIVEN een gerapporteerde realisatie van EUR 8.000 in Q2 met bewijsmiddelen, WHEN de SROI-coordinator de rapportage controleert, THEN goedkeuring of weigering met motivering wordt vastgelegd, en de cumulatieve realisatie wordt bijgewerkt.
- GIVEN een leverancier rapporteert SROI via "lokale inkoop" bouwblokken (e.g. bestelling bij lokale MKB-cateraar), WHEN de SROI-coordinator dit toetst, THEN het systeem vereist een onderaannemerscontract als bewijs en past de strengere validatie toe dat de lokale leverancier zelf de doelgroep moet inzetten (anti-stapeling regel uit bouwblokkenmodel 2023).

**REQ-MVI-006 - Tussentijdse en eindrapportage SROI met escalatie**
The system MUST per `tussentijdseRapportagesFrequentie` een rapportage-deadline bewaken, MUST de leverancier herinneren bij naderende deadline, en MUST escalatie-procedure starten als de realisatie significant achterloopt op de planning (default >15% achterstand).
- GIVEN een halfjaarlijks rapportageschema en de leverancier heeft nog niets gerapporteerd 7 dagen voor deadline, WHEN scheduled herinnering draait, THEN de leverancier krijgt email + portal-notificatie, en de contractmanager wordt CC'd na 3 dagen niet-reageren.
- GIVEN cumulatieve realisatie is 30% van planning op halverwege de periode, WHEN de rapportage wordt verwerkt, THEN het systeem markeert de verplichting als `risicovol`, een escalatie-ticket wordt aangemaakt voor de SROI-coordinator, en een gesprek met de leverancier wordt voorgesteld via een werkproces-step.
- GIVEN het einde van de realisatieperiode is bereikt en realisatie is 80% van verplichting, WHEN eindrapportage wordt opgemaakt, THEN het systeem checkt de contractuele consequentie (boete-clausule), berekent het boete-bedrag (typisch het ontbrekende deel met 25% opslag), en stelt facturatie voor met motivering.

**REQ-MVI-007 - MVI-contracteisen monitoren post-award**
The system MUST contracteis-MVI-criteria (zoals "CO2-Prestatieladder niveau 4 behouden", "jaarlijkse circulariteitsrapportage", "geen schending kinderarbeids-due-diligence") tijdens contractuitvoering monitoren, MUST jaarlijkse herijking van bewijsmiddelen vragen, en MUST koppeling leggen met de boete/bonus-clausules.
- GIVEN een contract met eis "CO2-ladder niveau 4 jaarlijks aantoonbaar", WHEN de jaarlijkse herijkingsdatum nadert (30 dagen), THEN de leverancier krijgt een verzoek om het actuele certificaat te uploaden via de portal.
- GIVEN een leverancier upload een certificaat met expiratie die meer dan 12 maanden geleden is verlopen, WHEN systeem dit detecteert, THEN het marker `eisStatus = niet_voldaan` en triggert het boetebepalings-werkproces uit het contract.
- GIVEN positieve over-prestatie (e.g. leverancier behaalt niveau 5 i.p.v. de geeiste niveau 4), WHEN dit wordt gerapporteerd, THEN het systeem kan een bonus-clausule activeren (indien in contract opgenomen) en logt de bovenmatige prestatie voor portfolio-rapportage.

**REQ-MVI-008 - Portfolio-dashboard voor MVI/SROI-prestatie**
The system MUST een portfolio-dashboard bieden waarop de organisatie real-time ziet welk percentage van uitgaven MVI-conform is per categorie, hoeveel SROI is gerealiseerd, en welke voortgang op portfolio-doelstellingen is geboekt, met drill-down per aanbesteding/contract.
- GIVEN een organisatie met portfoliodoel "50% circulair in 2030, baseline 5% in 2023", WHEN het dashboard wordt geopend in Q2 2026, THEN een visualisatie toont de huidige stand (e.g. 18%), de geplande trajectorie, en de status (groen/oranje/rood t.o.v. lineair pad).
- GIVEN de gebruiker filtert het dashboard op categorie "Schoonmaak", WHEN drill-down plaatsvindt, THEN per actief schoonmaakcontract wordt MVI-conformiteit, SROI-realisatie en CO2-getal getoond, met clickthrough naar het contract.
- GIVEN het dashboard toont 80% MVI-conformiteit voor ICT maar de organisatie heeft een doel van 95%, WHEN de portfolio-manager hierop klikt, THEN een lijst verschijnt van ICT-aanbestedingen die nog niet MVI-conform zijn, met voorstellen voor de volgende aanbestedingsronde.

**REQ-MVI-009 - Rapportage MVOI-monitor en bestuurlijke verantwoording**
The system MUST jaarlijks (rond december voor volgend jaar februari) de MVOI-monitor-uitvoer genereren conform het door PIANOo/RIVM vastgestelde format, MUST een bestuurlijk rapport voor college van B&W of Raad genereren, en MUST de cijfers koppelen aan het MVO-jaarverslag.
- GIVEN het einde van een kalenderjaar, WHEN de MVOI-export wordt gestart, THEN een CSV+JSON wordt gegenereerd met per categorie: totaal uitgaven, MVI-conform percentage, SROI-realisatie, CO2-baseline en realisatie, circulariteitsindex; volgens de actuele MVOI-monitor template.
- GIVEN een quarterly bestuurlijk rapport, WHEN de portfolio-manager "Genereer collegerapport" kiest, THEN een opgemaakte PDF wordt geproduceerd met executive summary, voortgang per portfoliodoel, top-10 succescases (best presterende contracten), en aandachtspunten.
- GIVEN een externe accountant doet de jaarrekeningcontrole en wil bewijs zien van MVI-implementatie, WHEN de auditor-export wordt aangevraagd, THEN een gesigned bundle wordt geleverd met cijfers, onderliggende rapportages, en sample-bewijsmiddelen per portfoliodoel.

**REQ-MVI-010 - Integratie met inkoopstrategie en bestuursbesluiten**
The system MUST `mvi_portfolio_doelstelling` koppelen aan bestuursbesluiten (collegebesluit, raadsbesluit) met datum, kenmerk en motivering, MUST bij wijziging van een doelstelling een nieuwe besluit-koppeling eisen, en MUST de doelstellingen visueel in de aanbestedings-UI tonen zodat inkopers tijdens een aanbesteding zien aan welke organisatiedoelen ze bijdragen.
- GIVEN een nieuw collegebesluit "CO2-reductie 49% in 2030 t.o.v. 1990", WHEN dit in purchaseq wordt vastgelegd, THEN een `mvi_portfolio_doelstelling` wordt aangemaakt met `bestuurlijkAkkoord` koppeling naar het besluit, en de baseline-waarde wordt vastgelegd.
- GIVEN een inkoper start een nieuwe aanbesteding voor energieleverantie, WHEN de aanbesteding-UI wordt geopend, THEN een banner toont "Deze aanbesteding draagt bij aan: 49% CO2-reductie 2030 (huidig: 32%)" met clickthrough naar de portfoliodoelstelling en advies voor gepaste MVI-criteria.
- GIVEN een portfolio-doelstelling wordt gewijzigd (e.g. ambitie omhoog naar 55% in 2030), WHEN dit wordt opgeslagen, THEN een nieuw bestuursbesluit-referentie verplicht wordt, oude doelstelling-versie behouden blijft voor historische rapportage, en lopende aanbestedingen optioneel kunnen worden geherevalueerd.

## Standards & Sources

- **Manifest Maatschappelijk Verantwoord Opdrachtgeven en Inkopen (MVOI 2026)** - het kader waaraan alle Rijks- en decentrale overheden zich committeren; behandelt zes themas (klimaat, circulair, sociaal/ketens, biodiversiteit, MKB-vriendelijk, internationale sociale voorwaarden ISV).
- **AanbestedingsWet 2012** art 1.4 (in acht nemen van milieuaspecten en sociale aspecten), art 2.78a (life-cycle costs), art 2.114 (mvi gunningscriteria).
- **EU-richtlijn 2014/24/EU** art 18 (algemene beginselen waaronder milieu/sociaal/arbeid), art 67 (EMVI met life-cycle), art 70 (uitvoeringsvoorwaarden).
- **RVO MVI-criteriatool** (mvicriteria.nl) - bron-database voor MVI-criteria per inkoopcategorie, gestructureerd in basis/significant/ambitieus.
- **CO2-Prestatieladder** (SKAO, niveaus 1-5) - meestgebruikte CO2-management-norm in NL bouw en infra.
- **PSO-Prestatieladder Sociaal Ondernemen** (TNO/PSO-Nederland, treden Aspirant/1/2/3) - voor sociaal ondernemerschap-meting.
- **Bouwblokkenmethode SROI** (versies 2018 en 2023) - de uniforme NL-methodiek voor SROI-verantwoording.
- **NEN-ISO 26000** voor maatschappelijke verantwoordelijkheid breed; **OECD Due Diligence Guidance** en de **Wet zorgplicht kinderarbeid** voor de internationale ketenketenverantwoordelijkheid.
- **Klimaatakkoord NL (2019)** en het **Convenant Klimaatakkoord Inkoop** met de daaruit voortvloeiende 49% CO2-reductie-doelstelling.
- **Nationaal Programma Circulaire Economie 2023-2030** met het 50%-circulair-in-2030 doel.
- **Cirkelstad-instrumentarium** voor circulair bouwen specifiek (R-ladder, CB23-meetmethodes).
- **CRBI / RVO Milieukeur, Europees Ecolabel, Cradle-to-Cradle** als certificeringsbronnen.
- **PIANOo handreiking Social Return** voor procesimplementatie.
- **TNO publicaties** over de wetenschappelijke onderbouwing van bouwblok-weegfactoren.

## Cross-app integration

- **aanbesteding-werkproces** (purchaseq base): MVI/SROI-set wordt aan een aanbesteding gekoppeld; werkproces-statemachine vraagt MVI-keuze aan voor publicatie.
- **tenderned-publicatie-adapter** (purchaseq zustercapability): MVI-criteria worden naar eForms BT-fields gemapt (BT-755 Accessibility, BT-805 GreenProcurement, BT-806 GreenProcurement Criteria); SROI-eis komt in BT-756 als sociale uitvoeringsvoorwaarde.
- **raamovereenkomst-minicompetitie** (purchaseq zustercapability): MVI-criteria uit de raamovereenkomst worden default doorgegeven aan minicompetities; SROI per minicompetitie kan apart worden bepaald.
- **openconnector**: import-bron voor RVO MVI-criteriatool (HTTP API), CO2-Prestatieladder register (SKAO API), PSO-register, MVO Nederland leveranciersbase.
- **openregister**: alle schema's; audit-trail voor portfoliodoel-mutaties met bestuursbesluit-koppeling.
- **docudesk**: bestek-tekstfragmenten per MVI-criterium worden uit een docudesk-template-library gehaald, automatisch ingevoegd op de juiste plaats in de leidraad; gunningsbeslissing en jaarrapportage genereerd via docudesk.
- **mydash**: het portfolio-MVI-dashboard is een mydash widget op basis van OR GraphQL queries; tegels per portfoliodoel, drill-down naar contracten.
- **opencatalogi**: organisatie-eigen MVI-criteria-bibliotheek kan worden gedeeld via opencatalogi met andere overheden (federation).
- **softwarecatalog**: voor IT-aanbestedingen kan de SWC inzicht geven in welke leveranciers welke MVI-certificeringen hebben (geintegreerd met BIO-2.0 audit-rapporten).
- **n8n / pipelinq**: scheduled jobs voor jaarlijkse MVOI-export, kwartaalrapportages naar college, SROI-deadline-herinneringen.
- **decidesk**: bestuursbesluit-koppeling (portfolio-doelstelling vereist een decidesk-besluit-record); raadsvragen over MVOI-voortgang worden vanuit decidesk gelinkt naar purchaseq-rapportages.

## Target users

- **Inkoper / aanbestedingsadviseur**: dagelijkse gebruiker; selecteert MVI-criteria per aanbesteding, beoordeelt MVI-scores van inschrijvingen.
- **SROI-coordinator**: per organisatie of regio; beheert het bouwblokkenmodel, accordeert leverancier-rapportages, escaleert achterstanden.
- **MVI-adviseur / duurzaamheidscoordinator inkoop**: bewaakt de portfoliostrategie, adviseert categoriemanagers, levert input voor MVOI-monitor.
- **Categoriemanager**: per categorie (IT, faciliteiten, bouw, zorg) verantwoordelijk voor MVI-implementatie binnen de portefeuille.
- **Contractmanager**: monitort MVI/SROI-naleving tijdens contractuitvoering, signaleert boete/bonus-momenten.
- **Bestuurder (wethouder Duurzaamheid, Sociaal Domein, Inkoop / gedeputeerde / portefeuillehouder Rijk)**: ontvangt periodieke portfolio-rapportage, neemt besluiten over portfoliodoelen.
- **Gemeenteraadslid / Statenlid / Kamerlid**: ontvangt jaarlijkse MVOI-rapportage, stelt schriftelijke vragen die via decidesk doorlopen.
- **Compliance / interne controle / accountant**: gebruikt audit-trail om MVI-beleid op rechtmatigheid en doelmatigheid te toetsen.
- **PIANOo / RIVM**: indirecte gebruiker; ontvangt de geuniformeerde MVOI-monitor-uitvoer voor de nationale aggregatie.
- **Leveranciers**: rapporteren SROI-realisatie via leveranciers-portal, leveren MVI-bewijsmiddelen, ontvangen herinneringen.
- **PSO-Nederland / SKAO / certificerende instellingen**: indirecte gebruiker; hun keurmerken worden via API uitgelezen voor bewijsmiddelvalidatie.
- **MKB-leveranciers**: indirecte begunstigde; MKB-vriendelijke MVI-criteria en passende eisen-formulering vergroten hun toegang tot opdrachten.
