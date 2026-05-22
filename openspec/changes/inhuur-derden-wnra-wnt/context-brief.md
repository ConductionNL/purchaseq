---
status: draft
app: purchaseq
spec: inhuur-derden-wnra-wnt
depends_on:
  - purchaseq base
  - shillinq dba-compliance-marker
target_users:
  - Inkoper / contractmanager
  - HR business partner
  - Controller / financieel beheerder
  - Bestuurder (WNT-verantwoordelijke)
  - Auditor (intern + extern)
standards:
  - Wet DBA (Wet Deregulering Beoordeling Arbeidsrelaties)
  - WNT (Wet normering topinkomens), norm 2026
  - Belastingdienst modelovereenkomsten (algemeen, branche, individueel)
  - DBA-handhavingsmoratorium uitfasering 2025/2026
  - CAO Uitzendkrachten (NBBU/ABU)
  - Wet Allocatie Arbeidskrachten Door Intermediairs (WAADI)
  - Wet Aanpak Schijnconstructies (WAS)
  - Cao-loonverhouding 1:5 (publiek) / Balkenende-norm
---

# Inhuur Derden met Wet DBA en WNT Toetsing

## Purpose

Beheer van externe inhuur (ZZP, uitzendkrachten, consultants, detachering) met ingebouwde compliance-checks voor de Wet DBA, de WNT, WAADI-registratieplicht en interne inhuurplafonds. Doel: aantoonbaar voorkomen dat de organisatie als opdrachtgever achteraf wordt geconfronteerd met loonheffingsnaheffingen, WNT-publicatieboetes of een schijnzelfstandigheids-verklaring richting de Belastingdienst, met name nu het handhavingsmoratorium per 1 januari 2026 volledig is afgelopen en de fiscus daadwerkelijk handhaaft op gezagsrelatie en vrije vervangbaarheid.

De spec maakt elke individuele inhuuropdracht traceerbaar van eerste behoefte tot uitstroom: behoefteformulering, leverancier/ZZP-keuze, modelovereenkomst-keuze, DBA-risico-assessment, tariefonderbouwing tegen WNT-norm, periodieke heronderzoeken (geen "te lange" inhuur die doorslaat in fictief dienstverband), en eindafrekening met bewijsstukken. Het systeem genereert het management-rapport voor de WNT-publicatieplicht en de jaarlijkse inhuur-ratio (interne medewerkers vs externe inhuur) die voor overheidsopdrachtgevers verplicht in het jaarverslag staat.

Inhuur is voor de meeste overheidsorganisaties de grootste verborgen risicocategorie: contracten lopen vaak via afdelingsbudgetten zonder centrale inkoop, en pas bij een belastingcontrole of WNT-onderzoek blijkt dat niet aantoonbaar is dat aan de criteria is voldaan. Deze spec verplaatst die controle naar voren in het proces.

## Data Model

Primaire entiteiten:

**InhuurOpdracht** — kerneenheid. Velden: opdrachtgevende afdeling, opdrachtnemer-id (ZZP of leverancier), opdrachtomschrijving, startdatum, einddatum, geplande einddatum, verlengingsoptie ja/nee, uurtarief (exclusief btw), geschat totaalbudget, daadwerkelijke uitgaven (gekoppeld aan shillinq factuurregels), kostenplaats, projectcode, inhuurtype (zzp/uitzend/detachering/consultancy/interim), gezagsrelatie-score, vervangbaarheid-score, dba-conclusie (groen/oranje/rood), wnt-relevant ja/nee, status (concept/in-onderzoek/goedgekeurd/actief/verlengd/afgerond/geblokkeerd).

**Opdrachtnemer** — natuurlijk persoon (bij ZZP) of rechtspersoon (bij leverancier). Velden: KvK-nummer, btw-nummer, geboortedatum (ZZP), woonadres (ZZP), zakelijk adres, IBAN, modelovereenkomst-type, modelovereenkomst-kenmerk (Belastingdienst nummer), datum laatste modelovereenkomst-check, WAADI-registratie ja/nee + nummer, G-rekeningnummer (bij uitzend), aantal-opdrachtgevers-laatste-12-maanden (ZZP-zelfstandigheids-indicator).

**DbaAssessment** — invulformulier per opdracht, periodiek te herhalen. Velden: opdracht-id, assessment-datum, beoordelaar, antwoorden op 12 standaardvragen (gezagsrelatie, instructiemacht, vervangbaarheid, ondernemersrisico, eigen werkmiddelen, eigen verzekeringen, meerdere opdrachtgevers, tariefstelling, factuurfrequentie), risicoscore 0-100, conclusie (groen <30 / oranje 30-60 / rood >60), advies, escalatie-vereist ja/nee.

**WntToets** — alleen voor WNT-plichtige organisaties (zorg, onderwijs, woningcorporaties, decentrale overheid). Velden: opdracht-id, kalenderjaar, totale uitkering-tot-nu-toe-in-jaar, WNT-norm-jaar, percentage-norm-bereikt, vrijstelling-van-toepassing (kortdurend <12 mnd), publicatieplicht-getriggerd ja/nee.

**InhuurRatio** — afgeleide entiteit. Per kostenplaats per kwartaal: aantal-fte-intern, aantal-fte-extern, kosten-intern, kosten-extern, ratio-percentage, drempelwaarde (organisatie-norm), overschreden ja/nee.

Relaties: opdracht 1:n facturen (shillinq), opdracht 1:n assessments (historie), opdracht 1:1 actieve modelovereenkomst, opdrachtnemer 1:n opdrachten, opdracht n:1 budget (purchaseq base).

## Requirements

### REQ-001: Inhuuropdracht aanmaken met verplichte DBA-assessment

GIVEN een ingelogde gebruiker met rol "inkoper" of "afdelingshoofd"
WHEN deze een nieuwe inhuuropdracht aanmaakt met inhuurtype "zzp"
THEN dwingt het systeem af dat een DBA-assessment is ingevuld voordat de opdracht de status "goedgekeurd" kan krijgen, en blokkeert verzending naar de opdrachtnemer zolang dba-conclusie ontbreekt of "rood" is zonder expliciete bestuurlijke vrijgave met onderbouwing.

### REQ-002: Modelovereenkomst-koppeling verplicht voor ZZP

GIVEN een ZZP-opdracht in status "in-onderzoek"
WHEN de opdrachtnemer geen geldige modelovereenkomst-kenmerk heeft of het kenmerk ouder is dan 5 jaar
THEN toont het systeem een blokkerende waarschuwing met de drie keuzes: (a) bestaande modelovereenkomst Belastingdienst selecteren uit de meegeleverde lijst, (b) branchemodelovereenkomst uploaden ter beoordeling, (c) individuele modelovereenkomst-aanvraag voorbereiden, en pas na keuze + vastlegging kan de opdracht verder.

### REQ-003: WNT-norm-bewaking realtime

GIVEN een actieve inhuuropdracht bij een WNT-plichtige organisatie
WHEN een nieuwe factuurregel binnenkomt via shillinq die de cumulatieve uitkering aan deze opdrachtnemer in het kalenderjaar boven 80% van de WNT-norm brengt
THEN stuurt het systeem een waarschuwing aan de bestuurder en de WNT-verantwoordelijke, en bij overschrijding boven 100% wordt automatisch een publicatieplicht-record aangemaakt voor opname in het jaarverslag inclusief alle wettelijk verplichte velden (naam, functie, dienstbetrekking-aard, beloning, einddatum).

### REQ-004: Maximale looptijd-bewaking en heronderzoek

GIVEN een actieve ZZP-inhuuropdracht
WHEN de doorlooptijd 6 maanden bereikt OF de cumulatieve uitgaven het initiële budget met 25% overschrijden
THEN trigger het systeem een verplicht her-assessment door de inkoper en blokkeert nieuwe factuurgoedkeuringen tot het her-assessment is afgerond, met expliciete vraag of de gezagsrelatie inmiddels is gewijzigd of de vervangbaarheid is afgenomen.

### REQ-005: WAADI-registratie-controle bij uitzendinhuur

GIVEN een nieuwe inhuuropdracht met inhuurtype "uitzend" of "detachering"
WHEN de opdrachtnemer (uitzendbureau) geen geldige WAADI-registratie heeft in het Handelsregister
THEN weigert het systeem de opdracht aan te maken en toont een rode foutmelding met verwijzing naar de KvK-pagina van de opdrachtnemer; de check gebeurt via openconnector kvk-adapter op het moment van leveranciers-selectie en wordt jaarlijks geverifieerd.

### REQ-006: Inhuur-ratio dashboard per kostenplaats

GIVEN een ingelogde controller of bestuurder
WHEN deze het inhuur-dashboard opent
THEN toont het systeem per kostenplaats de actuele inhuur-ratio (kosten extern / kosten totaal), met kleurcodering tegen de organisatie-norm, een trendgrafiek over de laatste 8 kwartalen, en een drill-down naar individuele opdrachten boven een drempelbedrag.

### REQ-007: Drie-handtekeningen-flow voor "rode" DBA-uitkomst

GIVEN een DBA-assessment met conclusie "rood" (hoog risico schijnzelfstandigheid)
WHEN de inkoper desondanks wil doorzetten met de opdracht
THEN vereist het systeem schriftelijke goedkeuring (digitale handtekening) van drie functies: afdelingshoofd, controller, en bestuurder, met verplichte motivering per goedkeurder die in de auditlog wordt vastgelegd; pas na alle drie kan de opdracht actief worden.

### REQ-008: Eindafrekening met bewijsstukken-checklist

GIVEN een inhuuropdracht die de einddatum nadert (binnen 30 dagen)
WHEN de opdracht naar status "afronding" gaat
THEN dwingt het systeem een checklist af: laatste factuur ontvangen, geheimhoudingsverklaring geretourneerd, toegangspassen ingenomen, IT-accounts beëindigd, kennisoverdracht-document geüpload, eindevaluatie-formulier ingevuld; alle items moeten afgevinkt voordat het dossier kan archiveren.

### REQ-009: Bestuursverslag-export jaarlijks

GIVEN de WNT-verantwoordelijke wil het jaarverslag voorbereiden
WHEN deze de jaar-export start over een afgesloten kalenderjaar
THEN levert het systeem een PDF en XML in het WNT-publicatieformaat met alle WNT-plichtige opdrachten, alle topfunctionarissen-vergelijkingen, en alle externe inhuur boven de norm, klaar voor publicatie op de organisatie-website binnen de wettelijke termijn van 1 juli.

### REQ-010: Auditlog onveranderlijk

GIVEN elke wijziging op een inhuuropdracht, DBA-assessment of WNT-toets
WHEN de wijziging wordt opgeslagen
THEN registreert het systeem in een hash-keten welk veld door welke gebruiker op welk moment is gewijzigd, met de oude en nieuwe waarde, zodat bij een Belastingdienst- of accountantscontrole het volledige bewijsspoor reproduceerbaar is.

## Cross-app

- **shillinq dba-compliance-marker** levert per factuurregel een "ZZP-vlag" die door deze spec wordt gebruikt om cumulatieve uitkeringen te aggregeren.
- **openconnector kvk-adapter** voor WAADI-controle en KvK-validatie van opdrachtnemers.
- **openconnector belastingdienst-modelovereenkomst-register** (toekomstig) voor automatische geldigheidscontrole van modelovereenkomst-kenmerken.
- **hrmq personeelsregister** voor de inhuur-ratio (intern aantal-fte als noemer).
- **decidesk besluitvorming** voor de drie-handtekeningen-flow bij rode DBA-uitkomst.
- **docudesk dossiervorming** voor archivering van afgeronde inhuurdossiers (bewaartermijn 7 jaar Belastingdienst).
