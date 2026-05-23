---
status: draft
app: purchaseq
spec: peppol-ubl-inkoop-factuur-ontvangst
depends_on:
  - openconnector peppol-e-invoicing-adapter
  - shillinq
  - purchaseq base
target_users:
  - Inkoper / contractmanager (PO-eigenaar)
  - Magazijnmedewerker / goederen-ontvanger (GR)
  - Crediteuren-administrateur (AP)
  - Budgethouder / kostenplaats-eigenaar
  - Controller / financieel beheerder
  - Auditor / accountant
standards:
  - Peppol BIS Billing 3.0 (Business Interoperability Specifications)
  - EN 16931 — Europese norm elektronisch factureren
  - UBL 2.1 (Universal Business Language) syntax binding
  - Peppol Access Point Service Level (Openpeppol AISBL)
  - NLCIUS (Nederlandse implementatie EN 16931, Forum Standaardisatie)
  - SBR / Digipoort voor B2G-flows (waar van toepassing)
  - Wet elektronische facturering bij overheidsopdrachten (verplichting overheid)
  - Btw-richtlijn 2010/45/EU / Wet OB 1968 (authenticiteit + integriteit)
  - Archiefwet (bewaartermijn 7 jaar fiscaal)
---

# Peppol UBL Inkoopfactuur Ontvangst en Drie-weg-match

## Placement & Information Architecture

**Placement type:** `SUB_PAGE` — Sub-page beneath a top-level menu entry. Renders as a page inside the parent surface (usually reachable via a router child route or a tab on the parent index page).

**Lives at:** Inkopen > Facturen

**Rationale:** 3-way-match lives under invoices  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Purpose

Een volledig geautomatiseerde inbound-stroom voor elektronische inkoopfacturen via het Peppol-netwerk, met UBL-validatie tegen Peppol BIS Billing 3.0 en NLCIUS, drie-weg-matching tegen de bijbehorende inkooporder en goederenontvangst, en doorzetting naar shillinq voor betaling. Elektronisch factureren via Peppol is voor Nederlandse overheden verplicht; voor de zakelijke markt is het de snelst groeiende standaard. De realiteit bij veel organisaties is dat de "Peppol-koppeling" een halve oplossing is: facturen komen weliswaar binnen, maar belanden in een mailbox van waaruit ze handmatig worden ingeklopt in het ERP, waarmee de hele winst (snelheid, foutreductie, audit-trail) verloren gaat.

Deze spec maakt het volledige pad gestroomlijnd: het Peppol Access Point in openconnector ontvangt het UBL-document, valideert het tegen BIS Billing 3.0 + NLCIUS, en triggert in purchaseq een matching-workflow. Het systeem zoekt de bijbehorende PO via het PO-referentie-veld in de UBL (BT-13), haalt de goederenontvangst op (indien aanwezig), en voert per factuurregel een 3-way match uit op artikel/dienst, hoeveelheid en prijs binnen de geconfigureerde toleranties. Bij volledig match gaat de factuur door naar shillinq als "goedgekeurd voor betaling". Bij afwijking start een geleide uitzonderingsworkflow: de PO-eigenaar krijgt de regelafwijking voorgelegd en kan accorderen, prijsupdate aan de PO toevoegen, of de factuur betwisten.

Het doel is dat 70-80% van de inkoopfacturen straight-through-processing haalt, met volledige conformiteit aan de wettelijke eisen rond authenticiteit, integriteit en leesbaarheid (Btw-richtlijn), en met een audit-trail die elke beslissing op factuurregel-niveau reproduceerbaar maakt.

## Data Model

**InboundPeppolFactuur** — ontvangstrecord. Velden: peppol-message-id, ontvangstdatum-tijd, verzender-peppol-id, ontvanger-peppol-id, ubl-document (origineel, immutable), document-type (Invoice/CreditNote), document-nummer, document-datum, valutacode, totaal-excl-btw, totaal-btw, totaal-incl-btw, validatie-status (in-behandeling/valide/ongeldig), validatie-fouten-jsonb, NLCIUS-conformiteit ja/nee.

**FactuurHeader** — geparsed uit UBL. Velden: factuur-id, leverancier-id (gematcht op KvK of OIN), leverancier-naam, leverancier-BTW-nummer, factuurnummer-leverancier, factuurdatum, vervaldatum, betalingsreferentie, PO-referentie (BT-13), totalen, betalingsvoorwaarden-tekst.

**FactuurRegel** — per UBL InvoiceLine. Velden: factuur-id, regelnummer, artikelcode, omschrijving, hoeveelheid, eenheid, eenheidsprijs, regelbedrag-excl-btw, btw-categorie, btw-percentage, btw-bedrag, PO-regel-referentie (BT-132), kostenplaats-suggestie.

**MatchingPoging** — uitkomst van de drie-weg-match per factuurregel. Velden: factuurregel-id, gematchte-PO-regel-id, gematchte-GR-regel-id (optioneel), match-status (volledig/gedeeltelijk/geen), prijsverschil-bedrag, prijsverschil-percentage, hoeveelheidsverschil, binnen-tolerantie ja/nee, conclusie (auto-akkoord/wacht-op-accordering/auto-afkeur).

**ApprovalTaak** — uitzonderingsworkflow. Velden: factuurregel-id, toegewezen-aan (PO-eigenaar of budgethouder), aangemaakt-op, deadline, status (open/geaccordeerd/afgekeurd/escalatie), actie-genomen, motivering, behandeld-op.

**MatchingTolerantie** — configuratie per organisatie. Velden: drempelwaarde-bedrag-absoluut, drempelwaarde-percentage, hoeveelheid-tolerantie, kostencategorieen-met-eigen-regels-jsonb.

**FactuurDispuut** — als de factuur niet doorgaat. Velden: factuur-id, dispuut-reden, terug-melding-tekst, uitgaand-credit-note-verzoek, status (gemeld/afgehandeld), Peppol-response-message-id.

**ArchiefRecord** — voor de 7-jaar fiscale bewaarplicht. Velden: factuur-id, archief-locatie, hash-checksum (integriteits-bewijs), authenticiteits-bewijs (Peppol-handtekening-keten), leesbaarheids-versie (PDF/A-3 met UBL embedded), vernietigingsdatum.

## Requirements

### REQ-001: Ontvangst en initiële validatie via Peppol AP

GIVEN een binnenkomend Peppol-bericht via de openconnector peppol-e-invoicing-adapter
WHEN het bericht arriveert en is gericht aan een Peppol-ID van deze organisatie
THEN slaat het systeem het originele UBL-document onveranderd op, parseert de kerngegevens, voert een schema-validatie uit tegen Peppol BIS Billing 3.0 en NLCIUS, en plaatst de factuur in de matching-wachtrij; bij schema-fouten wordt een Peppol-response (MLR negative) teruggestuurd en de factuur op status "ongeldig" gezet met de fouten beschikbaar voor de leverancier.

### REQ-002: Leverancier-matching op basis van Peppol-ID en KvK

GIVEN een gevalideerde inkomende factuur
WHEN de leverancier wordt gezocht in het leveranciersregister
THEN matcht het systeem primair op de Peppol Participant Identifier, secundair op KvK-nummer uit het EndpointID-veld, en bij geen match wordt automatisch een "voorgesteld nieuwe leverancier" record aangemaakt met alle uit de UBL afgeleide gegevens, ter goedkeuring door een crediteuren-administrateur voordat de factuur verder kan in de workflow.

### REQ-003: PO-koppeling via BT-13 referentie

GIVEN een factuurheader met een ingevulde BT-13 PO-referentie
WHEN de matching start
THEN zoekt het systeem de bijbehorende inkooporder in purchaseq, controleert of de leverancier op de PO overeenkomt met de leverancier op de factuur, en koppelt beide; bij ontbrekende of niet-vindbare PO-referentie wordt de factuur naar een aparte "geen-PO" wachtrij verplaatst voor handmatige toewijzing of als kostenfactuur zonder PO behandeld volgens de organisatie-regels.

### REQ-004: Drie-weg-match per regel met tolerantie

GIVEN een factuur met gekoppelde PO en (waar van toepassing) goederenontvangst
WHEN het systeem de regelmatching uitvoert
THEN vergelijkt het per factuurregel de hoeveelheid en eenheidsprijs tegen de PO-regel en de GR-regel binnen de geconfigureerde toleranties (bijvoorbeeld 2% prijsafwijking of EUR 25 absoluut), en markeert regels die binnen tolerantie vallen als "auto-akkoord" en regels die afwijken als "wacht-op-accordering".

### REQ-005: Geleide uitzonderingsworkflow voor afwijkingen

GIVEN een factuurregel met match-status "gedeeltelijk" of "geen"
WHEN de PO-eigenaar de approval-taak opent
THEN toont het systeem zij-aan-zij de PO-regel, GR-regel en factuurregel met de verschillen visueel gemarkeerd, en biedt drie acties: (a) accorderen met motivering, (b) PO-prijs of -hoeveelheid bijwerken met automatische re-match, (c) factuur betwisten met conceptbericht aan leverancier; elke actie wordt gelogd en de keuze bepaalt de vervolgstap.

### REQ-006: Doorzetting naar shillinq bij volledige goedkeuring

GIVEN een factuur waarvan alle regels status "auto-akkoord" of "geaccordeerd" hebben
WHEN de header-status overgaat naar "klaar-voor-betaling"
THEN levert het systeem de factuur via een interne event-bus aan shillinq met de volledig geverifieerde betaalgegevens (IBAN uit de UBL, vervaldatum, korting-bij-snelle-betaling indien aanwezig in BT-20), en blokkeert tegelijk verdere wijzigingen aan de factuur in purchaseq.

### REQ-007: Dispuut-workflow met Peppol-respond

GIVEN een factuur die wordt betwist
WHEN de PO-eigenaar of crediteuren-administrateur het dispuut bevestigt
THEN genereert het systeem een Peppol Invoice Response (UBL ApplicationResponse) met de juiste statuscode en motivatie, verstuurt deze via het Access Point terug naar de leverancier, en houdt de factuur intern op status "in-dispuut" tot een credit-note of correctie-factuur binnenkomt die het dispuut sluit.

### REQ-008: Automatische detectie dubbele facturen

GIVEN een nieuwe binnenkomende factuur
WHEN het systeem de header verwerkt
THEN controleert het of de combinatie leverancier-id + leverancier-factuurnummer + factuurdatum al bestaat in het systeem, en bij detectie van een mogelijke dubbele factuur blokkeert het de verwerking met een alert naar de crediteuren-administrateur die handmatig moet beoordelen of het een legitieme herziening, een dubbele verzending door de leverancier, of een poging tot dubbel-factureren betreft.

### REQ-009: Authenticiteit, integriteit en leesbaarheid borgen

GIVEN een gearchiveerde factuur
WHEN deze in het archief wordt geplaatst
THEN bewaart het systeem het originele UBL-document met de Peppol-handtekening-keten als authenticiteits-bewijs, berekent een SHA-256 hash als integriteits-bewijs, en genereert een PDF/A-3 visualisatie met de UBL als embedded attachment voor leesbaarheid, alles met een bewaartermijn van minimaal zeven jaar conform de fiscale bewaarplicht en automatische monitoring tegen onbedoelde verwijdering.

### REQ-010: Dashboard straight-through-rate en SLA-bewaking

GIVEN een controller die de proces-prestaties wil monitoren
WHEN deze het inkoopfactuur-dashboard opent
THEN toont het systeem de straight-through-processing rate over de laatste 30/90/365 dagen, de gemiddelde doorlooptijd van ontvangst tot betaling, het aantal facturen per status, de top-10 leveranciers met afwijkingen, en de openstaande approval-taken per gebruiker met deadline-overschrijdingen rood gemarkeerd.

## Cross-app

- **openconnector peppol-e-invoicing-adapter** is de noodzakelijke voorganger: levert het Peppol Access Point (SMP-registratie, AS4-transport, message-level-respons) en publiceert ontvangen documenten op een interne event-stream waar deze spec op luistert.
- **shillinq** ontvangt goedgekeurde facturen voor betaling-workflow en boekhouding.
- **purchaseq base** levert de inkooporders en de goederenontvangst-registraties.
- **docudesk** voor de wettelijke 7-jaar archivering met integriteit/authenticiteit/leesbaarheid-borging.
- **decidesk** voor het escaleren van disputen of significante afwijkingen die bestuurlijke besluitvorming vereisen.
- **openconnector kvk-adapter** voor het verifieren van nieuwe leveranciers die voor het eerst binnenkomen via Peppol.
