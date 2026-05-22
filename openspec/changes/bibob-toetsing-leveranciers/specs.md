---
status: proposed
app: purchaseq
spec: bibob-toetsing-leveranciers
version: 1.0
---

# Specifications — BIBOB Toetsing Leveranciers en Opdrachtnemers

## REQ-001: Automatische risico-detectie bij nieuwe opdrachten

### REQ-001-001: Risico-indicator scannen bij opdracht-aanmaak

**GIVEN** een nieuwe aanbestedingsopdracht in purchaseq  
**WHEN** de opdracht wordt opgeslagen met:
  - Geraamde waarde ≥ organisatie-drempel (default €50.000, configureerbaar), OF
  - Leverancier-sector in [bouw, horeca, afval, vastgoed, transport], OF
  - Leverancier bedrijfsleeftijd < 24 maanden, OF
  - Eigendomswijziging KvK-data < 24 maanden geleden  
**THEN** stuurt purchaseq een CloudEvent `purchaseq.order.bibob-risk-detected` naar bibob-app

**Acceptance:**
- ✓ Risk detection fires synchronously on save, not async (no delay to purchaser)
- ✓ Threshold and sector list are configurable in bibob_register.json
- ✓ KvK-data enrichment (company age, ownership) via openconnector kvk-adapter

---

### REQ-001-002: Notification naar BIBOB-coordinator

**GIVEN** een `purchob-toetsing-leveranciers.order.bibob-risk-detected` CloudEvent ontvangen  
**WHEN** bibob-app verwerkt het event  
**THEN**:
  - Markeer de oorspronkelijke PurchaseOrder als "bibob-risk-flagged"
  - Maak een NotificationService-call: type "bibob.risk.detected", recipient = rol [integriteitscoordinator], body bevat:
    - Leverancier naam + KvK nummer
    - Risico-factoren (sector, prijs, bedrijfsleeftijd, eigendom)
    - Link naar purchase order detail
    - "Dossier aanmaken?" button → pre-filled form-1 aanmaken

**Acceptance:**
- ✓ Coordinator ontvangt melding in Nextcloud-notificatie-center
- ✓ Melding bevat leverancier-context en risico-samenvatting
- ✓ "Dossier aanmaken" pre-fills BibobDossier met aanleiding=aanbesteding, referentie=order-id

---

### REQ-001-003: Geen blokkering van aanbesteding zelf

**GIVEN** risico-waarschuwing actief  
**WHEN** inkoper wil opdracht naar fase "voorlopige-gunning" zetten  
**THEN** mag dit, TENZIJ een actief BIBOB-dossier bestaat met conclusie="ernstige-mate-van-gevaar" (REQ-005-001 bepaalt blokkering, niet REQ-001)

**Acceptance:**
- ✓ Risk detection is advisory only; coordinator moet expliciet dossier aanmaken
- ✓ No automatic blocking of purchase orders at this stage

---

## REQ-002: BIBOB-formulier 1 (Eigen onderzoek)

### REQ-002-001: Formulier-1-schema en vragen

**GIVEN** BibobFormulier1 entiteit in OpenRegister  
**WHEN** coördinator formulier-1 aanmaakt  
**THEN** toont CnFormDialog met gestructureerde vragenlijst:

1. **Financiering** (open text):
   - Welke financieringsbronnen zijn gebruikt?
   - Zijn alle bronnen legitimate en transparant?

2. **Strafbare feiten** (ja/nee, optioneel detail):
   - Zijn bestuurders/aandeelhouders veroordeeld voor strafbare feiten?
   - Zo ja: welke feiten, wanneer, welke straf?

3. **Schulden** (ja/nee, optioneel detail):
   - Fiscale schulden / belastingachterstanden?
   - Sociale zekerheidspremies openstaand?
   - Andere openstaande schulden bij publieke sector?

4. **Ondernemingsstructuur** (text):
   - Juridische vorm, oprichting-jaar
   - Bestuurders en hun functies elders
   - Aandeelhouders met zeggenschapspercentage
   - Eventuele UBO's (Ultimate Beneficial Owners)

5. **Zakelijke relaties** (text):
   - Relatie met andere leveranciers (concurrentie, afhankelijkheid)?
   - Relatie met gemeente-medewerkers of bestuurders (belangenconflict)?
   - Kapitaalsrelaties met externe partijen?

**Acceptance:**
- ✓ Schema matches BibobFormulier1.antwoorden_json structure
- ✓ CnFormDialog renders all 5 question blocks with auto-save
- ✓ File upload slots for KvK extract, financial statements, financing agreements

---

### REQ-002-002: KvK-data automatische enrichment

**GIVEN** leverancier KvK-nummer beschikbaar  
**WHEN** formulier-1 geopend  
**THEN** roept bibob-app openconnector kvk-adapter aan en vult auto in:
  - Bedrijfsnaam
  - Juridische vorm
  - Oprichting-datum
  - Bestuurders (names, roles)
  - Aandeelhouders (names, shares)
  - UBO-register info (if available)

**Acceptance:**
- ✓ KvK call is async (doesn't block form load)
- ✓ Auto-filled fields remain editable (coordinator can override/correct)
- ✓ API errors don't break form; show "KvK data unavailable" notice

---

### REQ-002-003: Bijlagen-upload

**GIVEN** formulier-1 bewerking  
**WHEN** coördinator selecteert bestanden voor upload  
**THEN**:
  - KvK-uittreksel (PDF/scan)
  - Jaarrekening (PDF)
  - Financieringsovereenkomst(en) (PDF)
  - Andere relevante stukken (max 10 MB per bestand, max 5 bestanden)

**Action:** FileService.upload() called, references stored in formulier1.bijgevoegde_stukken array

**Acceptance:**
- ✓ Upload progress shown
- ✓ Files stored in OpenRegister via FileService
- ✓ Filenames + upload date logged in bijgevoegde_stukken

---

### REQ-002-004: Eerste-indicatie vastleggen

**GIVEN** alle vragen ingevuld  
**WHEN** coördinator klikt "Onderzoek afronden"  
**THEN** wordt first-indicatie verplicht gekozen:
  - **geen-twijfel**: Geen aanwijzingen voor integriteitsrisico's
  - **twijfel**: Enkele aanwijzingen, nodig verdiepend onderzoek
  - **sterke-twijfel**: Duidelijke aanwijzingen, escalatie naar formulier-2 sterk aan te raden

**Action:** BibobFormulier1.eerste_indicatie ingesteld, BibobDossier.onderzoeksfase = "formulier-1", status = "onderzoek"

**Acceptance:**
- ✓ First-indicatie cannot be saved as null
- ✓ Choice recorded in auditlog with timestamp + user

---

## REQ-003: Blokkering gunningsbesluit zonder formulier-1

### REQ-003-001: Blokkering "voorlopige-gunning" wanneer risico-dossier actief

**GIVEN** een purchase order met risico-dossier in fase "formulier-1"  
**WHEN** inkoper probeert te setten naar fase "voorlopige-gunning"  
**THEN**:
  - Kontroleer: bestaat er BibobDossier met aanleiding_referentie = order-id EN onderzoeksfase ≠ "geen"?
  - JA: block met message "BIBOB-onderzoek BIBOB-2024-XXXXX is nog actief. Afronding formulier-1 en eerstte-indicatie vereist."
  - NEE: allow transition

**Acceptance:**
- ✓ Check runs synchronously in purchaseq order-state-transition middleware
- ✓ Block is clear and includes dossier number for lookup

---

### REQ-003-002: Gemotiveerde afwijzing onderzoeks-verplichting

**GIVEN** risico is gesignaleerd, maar coordinator wil GEEN dossier aanmaken  
**WHEN** coordinator registreert "Onderzoek niet nodig" in purchaseq order-context  
**THEN**:
  - Creëer BibobDossier met onderzoeksfase = "geen", status = "gesloten"
  - Registreer motivering (vrij tekstveld, min. 100 chars)
  - Auditlog: user, timestamp, motivering
  - Verwijder "bibob-risk-flagged" marking van order

**Acceptance:**
- ✓ Coordinator must provide explicit reasoning
- ✓ Closure is auditable and defensible
- ✓ Order can then proceed to "voorlopige-gunning"

---

## REQ-004: Privacy-gescheiden opslag bijzondere persoonsgegevens

### REQ-004-001: Gescheiden, encrypted-at-rest opslag

**GIVEN** BibobDossier wordt opgeslagen met gevoelige persoonsgegevens (BSN, adres, strafbare feiten)  
**WHEN** ObjectService.saveObject() wordt geactiveerd  
**THEN**:
  - Alle velden in BibobBetrokkene.bsn en BibobBetrokkene.woonadres (en BibobFormulier2.zienswijze-betrokkene indien persoonlijk)
  - Encrpted at rest met app-database-sleutel
  - Opgeslagen in dezelfde OpenRegister-tabel, maar met `_privacy_level: "restricted"` metadata-vlag

**Acceptance:**
- ✓ Encryption is automatic via OpenRegister `EncryptionHandler` (if available in OR 4+), otherwise app-level encryption
- ✓ Fields remain queryable for coordinator (decryption on read)

---

### REQ-004-002: Expliciet autorisatielijst (max 3 personen)

**GIVEN** BibobDossier met `_privacy_level: "restricted"`  
**WHEN** user probeert BibobBetrokkene.bsn/woonadres te lezen  
**THEN**:
  - Check user rol + organisatie + dossier assignment:
    - **BIBOB-coordinator** (rol) → allow
    - **Juridisch adviseur** (rol) → allow
    - **Gemandateerd bestuursorgaan** (rol, mandaatregister lookup in decidesk) → allow
    - **Inkoper** (rol) → DENY (403), show "This dossier contains restricted data"
    - **Anoniem/guest** → DENY

**Action:** PropertyRbacHandler enforces per-field access control

**Acceptance:**
- ✓ RBAC check happens before data leaves app, even in API responses
- ✓ Unauthorized read attempts logged to auditlog
- ✓ Coordinator sees full names + BSN; Inkoper sees only "dossier exists + final conclusion"

---

### REQ-004-003: Regelmäßige autorisatielij st-reviews

**GIVEN** BibobDossier met restricted data  
**WHEN** 90 dagen voorbij zonder access  
**THEN**: Verstuur reminder naar integrity-manager: "Dossier BIBOB-2024-XXXXX has no read access in 90 days. Review access list?" (optional compliance measure)

**Acceptance:**
- ✓ Reminder is advisory (no forced unlock)
- ✓ Logged in auditlog

---

## REQ-005: Escalatie van formulier-1 naar formulier-2

### REQ-005-001: Gestructureerde escalatie-keuze

**GIVEN** BibobFormulier1 afgerond met eerste_indicatie = "twijfel" of "sterke-twijfel"  
**WHEN** coördinator BibobDossier opent en navigeert naar "Escalatie"  
**THEN** toont formulier met twee mutually-exclusive keuzes:

**Optie A: Opwaarderen naar formulier-2**
- Verplicht veld: "Onderbouwing voor escalatie" (min. 250 chars)
- Click: Creëer BibobFormulier2, stel BibobDossier.onderzoeksfase = "formulier-2"
- Status → "onderzoek", deadline +30 days

**Optie B: Afsluiten met afwijzing**
- Verplicht veld: "Waarom verder onderzoek niet proportioneel" (min. 200 chars)
- Click: Stel BibobDossier.status = "gesloten", onderzoeksfase = "formulier-1"
- Registreer in auditlog: user, timestamp, motivering, conclusie → "geen-bezwaar"

**Acceptance:**
- ✓ Both choices recorded in auditlog
- ✓ Escalation to form-2 requires explicit reasoning (prevents lazy escalations)
- ✓ Closure with reasoning creates defensible decision trail

---

### REQ-005-002: Escalatie-guard

**GIVEN** BibobDossier met eerste_indicatie = "geen-twijfel"  
**WHEN** coördinator opent "Escalatie"-sectie  
**THEN** toon grijzed-out message: "Eerste-indicatie is 'geen-twijfel'. Escalatie niet van toepassing. Dossier kan direct worden gesloten."

**Acceptance:**
- ✓ Prevents accidental escalation of non-risk dossiers

---

## REQ-006: Zienswijze-procedure conform Awb

### REQ-006-001: Draft-brief naar betrokkene

**GIVEN** BibobFormulier2 afgerond met voorlopige_conclusie  
**WHEN** coördinator klikt "Zienswijze aanvragen"  
**THEN**:
  - Genereer voorgestelde brief (template in design.md):
    ```
    Concept-brief
    Datum: [today]
    Aan: [BibobBetrokkene.naam] ([BibobBetrokkene.relatie_tot_aanvrager])

    Op verzoek van [gemeente/bestuursorgaan] hebben wij integriteitsonderzoek 
    verricht naar uw bedrijf in de context van aanbestedingsprocedure [reference].

    Gegrond op ons onderzoek, ons voorlopige conclusie is: [voorlopige_conclusie]

    Dit berust op de volgende bevindingen:
    [Samenvatting formulier-2.openbare_bronnen_geraadpleegd + antwoorden]

    U hebt twee weken tijd (tot [today + 14 dagen]) om zienswijze schriftelijk 
    in te dienen. Reactie zenden aan [coördinator-email].

    Na ontvangst van uw zienswijze (of verloop van termijn) zal het final besluit 
    worden getroffen door [Bestuurder/Orgaan].
    ```
  - Draft-brief toont in modal, coördinator kan aanpassen
  - Click "Verzenden": BibobDossier.status = "advies", BibobFormulier2.zienswijze_betrokkene.deadline = now + 14 days
  - Stuur brief (print-klaar PDF of email via NotificationService)

**Acceptance:**
- ✓ Draft-brief is aanpasbaar
- ✓ Deadline vastgelegd in database
- ✓ Brief sent date logged

---

### REQ-006-002: Zienswijze-indiening registreren

**GIVEN** deadline loopt (14 dagen)  
**WHEN** betrokkene dient schriftelijke zienswijze in (upload of scanner)  
**THEN**:
  - Coördinator motiveert upload in BibobFormulier2.zienswijze_betrokkene.document
  - Stel BibobFormulier2.zienswijze_betrokkene.ontvangen_datum = now
  - Auditlog: user "received zienswijze" + document reference

**Acceptance:**
- ✓ Zienswijze kan worden geupload als PDF-bestand
- ✓ Alternativt kan coördinator notulen van mondelinge zienswijze registreren

---

### REQ-006-003: Blokking definitief besluit tot zienswijze-termijn verstreken

**GIVEN** BibobDossier.status = "advies" (zienswijze-termijn actief)  
**WHEN** coördinator probeert BibobBesluit.besluit_type in te vullen en op te slaan  
**THEN**:
  - Controleer: is nu > zienswijze-deadline + 0 dagen? (allow after deadline passes)
  - NEE: block met message "Zienswijze-termijn nog actief tot [deadline]. Verwacht antwoord of laat termijn verlopen."
  - JA: allow besluit-draft

**Acceptance:**
- ✓ Ensures Awb-procedure completeness
- ✓ Prevents premature final decision

---

## REQ-007: LBB-adviesaanvraag met gestructureerde onderbouwing

### REQ-007-001: Verplichte invul-onderdelen

**GIVEN** coördinator start LbbAdviesaanvraag  
**WHEN** CnFormDialog opent voor aanvraag-invulling  
**THEN** verplichte velden (submittable alleen na complete invulling):

1. **Aanleiding** (text, min. 100 chars)
   - Samenvatting: welke opdracht/vergunning, waarom is dossier bij LBB?

2. **Onderzoeksresultaten samengevat** (text, min. 250 chars)
   - Wat hebben formulier-1 en formulier-2 opgeleverd?
   - Welke feiten wijzen op integriteitsrisico's?

3. **Specifieke vragen aan LBB** (list)
   - "Passen deze bevindingen onder Wet Bibob artikel ...?"
   - "Is ondernemingsstructuur verdacht genoeg om weigering te rechtvaardigen?"
   - (etc., min. 2 vragen)

4. **Alle bekende betrokkenen** (table)
   - Naam, type (aanvrager/bestuurder/aandeelhouder/UBO), rol, percentages
   - Automatisch gepopuleerd uit BibobBetrokkene-records

5. **Gewenste antwoord-datum** (date picker)
   - Min. 14 dagen vandaag

**Acceptance:**
- ✓ All fields mandatory (no null submissions)
- ✓ Text-length validation enforced
- ✓ Betrokkene-table auto-populates from dossier

---

### REQ-007-002: Beveiligde verzending

**GIVEN** LbbAdviesaanvraag compleet ingevuld  
**WHEN** coördinator klikt "Verzenden naar LBB"  
**THEN**:
  - Compile aanvraag-tekst (merge all fields + dossier summary)
  - Initiate OpenZaak SOAP-call (or dedicated LBB-adapter via ExApp):
    ```
    POST /lbb/adviesaanvraag/
    {
      "zaaknummer": [dossier-number],
      "onderbouwing": [compiled text],
      "betrokkenen": [list from form],
      "gewenste_antwoord_datum": [date]
    }
    ```
  - LBB responds: `{ "lbb_zaaknummer": "LBB-2024-12345", "ontvangst_bevestiging": "2024-04-10T10:30:00Z" }`
  - Stel LbbAdviesaanvraag.lbb_zaaknummer, verzending_bevestiging, status → "sent"

**Acceptance:**
- ✓ Request signed/encrypted in transit (TLS at minimum)
- ✓ Confirmation receipt stored
- ✓ Failed sends block until retry successful

---

### REQ-007-003: LBB-advies ontvangen en opslaan

**GIVEN** LBB-advies binnenkomst (via webhook of polling)  
**WHEN** bibob-app ontvangt advies met lbb_zaaknummer  
**THEN**:
  - Zoek LbbAdviesaanvraag bij lbb_zaaknummer
  - Stel LbbAdviesaanvraag.advies_datum = [advies-datum van LBB]
  - Stel LbbAdviesaanvraag.advies_conclusie = [LBB conclusion: geen-bezwaar / lichte-mate / ernstige-mate]
  - Advies-tekst zelf: encrypt + store in separate secured blob, reference in advies_tekst (do not inline)
  - Stel BibobDossier.onderzoeksfase = "lbb-advies"

**Acceptance:**
- ✓ Advices encrypted at rest
- ✓ Separate from main dossier object (prevents accidental exposure)

---

## REQ-008: Bestuursbesluit met Awb-motiveringsplicht

### REQ-008-001: Besluit-template met verplichte onderdelen

**GIVEN** onderzoek afgerond (formulier-1, formulier-2, en optioneel LBB-advies)  
**WHEN** bestuurder klikt "Besluiting voorbereiden"  
**THEN** CnAdvancedFormDialog toont template met vier verplichte blokken (Awb artikel 3:46):

**1. Feitelijke grondslag (min. 250 chars)**
```
Beschrijving van de feiten waarop het beslist:
- Werkingsgebied: [aanbestedingsprocedure / vergunning]
- Leverancier: [naam + KvK]
- Onderzoeks-aanleiding: [risico-indicator]
- Bevindingen formulier-1: [samenvatting]
- Bevindingen formulier-2: [samenvatting, if applicable]
- LBB-advies: [conclusion, if received]
```

**2. Juridische grondslag (min. 50 chars)**
```
Wettelijke basis voor deze besluit:
- Wet Bibob, artikel [cite relevant article]
- Awb artikel [3:46]
- Aanbestedingswet 2012, artikel [cite]
- Beleidsregel Bibob van [organisatie], vastgesteld [date]
```

**3. Belangenafweging (min. 100 chars)**
```
Afweging van relevante belangen:
- Belang goed bestuur / integriteit van overheid
- Belang leverancier (bedrijfsvoering, reputatie)
- Belang marktdeelnemers (fair competition)
- Belang burger (kwaliteit dienstverlening)

Conclusie uit afweging: [gekozen belang prevaleert omdat ...]
```

**4. Proportionaliteitstoets (min. 100 chars)**
```
Proportionaliteit van de maatregel:
- Is weigering/niet-gunning noodzakelijk gegeven de bevindingen?
- Zijn er minder invasieve alternatieven (bijv. voorwaardelijke gunning)?
- Staat gewicht van risico in redelijke verhouding tot gevolgen voor leverancier?

Conclusie: [maatregel is/niet proportioneel omdat ...]
```

**Acceptance:**
- ✓ All four sections mandatory + min-length enforced
- ✓ Sections are editable text areas with live character count
- ✓ Save disabled until all sections ≥ min-length

---

### REQ-008-002: Besluittype selectie

**GIVEN** template ingevuld  
**WHEN** bestuurder selecteert besluittype  
**THEN** radiobuttons:
- **Gunning** — Leverancier gunnen ondanks lichte twijfels; voorwaarden kunnen gelden
- **Niet-gunning** — Aanbesteding opnieuw uitschrijven, deze leverancier uitgesloten
- **Weigering** — Vergunning weigeren / niet toekennen
- **Intrekking** — Eerder toegekende vergunning/gunning intrekken
- **Voorwaardelijk** — Gunning onder voorwaarden (bijv. bankgarantie, monitoring)

**Acceptance:**
- ✓ Besluittype bepaalt vervolgstappen (bijv. shillinq payment-block bij niet-gunning)

---

### REQ-008-003: Digitale ondertekening via decidesk

**GIVEN** BibobBesluit template compleet + besluittype gekozen  
**WHEN** bestuurder klikt "Ter ondertekening aanbieden"  
**THEN**:
  - Roep decidesk besluitvorming-workflow aan:
    ```
    POST /decidesk/decision/
    {
      "title": "BIBOB-besluit BIBOB-2024-XXXXX",
      "decision_body": [compiled template + all four sections],
      "required_signatory_role": "gemandateerd-bestuursorgaan",
      "reference": "bibob:BIBOB-2024-XXXXX"
    }
    ```
  - Decidesk genereert ondertekenings-aanvraag, stuurt notificatie naar mandaathouder
  - Bij ondertekening (decidesk callback): stel BibobBesluit.ondertekenaar = [mandaathouder], BibobBesluit.datum_uitreiking = now

**Acceptance:**
- ✓ Signature authority checked via decidesk mandaatregister
- ✓ Signed decision becomes immutable (archived)
- ✓ Digital signature timestamp recorded

---

### REQ-008-004: Minimum-tekstlengte enforcement

**GIVEN** bestuurder probeert besluit op te slaan  
**WHEN** een van de vier secties beneden minimum-lengte  
**THEN** blokkeer opslaan met message: "Feitelijke grondslag nog incomplete (XXX/250 chars). Toelichtingen moet vollediger worden."

**Acceptance:**
- ✓ Prevents hollow compliance (all sections must be substantive)

---

## REQ-009: Bezwaartermijn-bewaking en uitreiking

### REQ-009-001: Uitreikingsdatum registreren

**GIVEN** BibobBesluit getekend  
**WHEN** bestuurder of jurist registreert "Besluit uitgereikt"  
**THEN**:
  - Stel BibobBesluit.datum_uitreiking = [date picker, default = today]
  - Stel BibobBesluit.bezwaartermijn_startdatum = datum_uitreiking
  - Bereken bezwaartermijn einde-datum = datum_uitreiking + 42 dagen (6 weeks)
  - Plan herinnering: -14 dagen (reminder voor jurist), -7 dagen (final notice), -0 dagen (termijn verstrijken)

**Acceptance:**
- ✓ Service date determines legal deadline (not creation date)
- ✓ Reminders scheduled automatically

---

### REQ-009-002: Bezwaarschrift-tracking

**GIVEN** bezwaartermijn actief  
**WHEN** jurist ontvangt bezwaarschrift (fysiek of digitaal)  
**THEN**:
  - Registreer in BibobDossier (extended field: bezwaarschriften[])
    - Ontvangst-datum
    - Indiener (party name)
    - Document-referentie
    - Link naar eventuele hoger-beroep/uitspraak-link
  - Auditlog: inkomend bezwaarschrift BIBOB-2024-XXXXX

**Acceptance:**
- ✓ Bezwaarschrift remains linked to original dossier for complete paper trail
- ✓ Útspraak-link allows closing-out of historical case

---

## REQ-010: Automatische bewaarregels en vernietiging

### REQ-010-001: Bewaartermijn-berekening

**GIVEN** BibobDossier gesloten  
**WHEN** status = "gesloten" opgeslagen  
**THEN**:
  - Creëer/update BibobBewaarregel:
    - If LbbAdviesaanvraag.advies_conclusie exists: bewaarduur_basis = "5-jaar-na-onherroepelijk-besluit"
      (start date = decision final appeal deadline passed, i.e., datum_uitreiking + 42 + 90 days for higher appeal)
    - Else: bewaarduur_basis = "5-jaar-na-sluiting"
      (start date = dossier close date)
    - vernietigingsdatum = start date + 5 years
    - anonimisering_geplanned = vernietigingsdatum - 90 days (advance prep)

**Acceptance:**
- ✓ Calculation is automatic (no manual intervention)
- ✓ Distinguishes LBB-advises (longer retention) from self-assessment (shorter)
- ✓ BibobBewaarregel is audit-locked once created

---

### REQ-010-002: Anonimisering-workflow

**GIVEN** anonimisering_geplanned datum bereikt  
**WHEN** ArchivalService.prepareForAnonymization() triggered (daily job)  
**THEN**:
  - Genereert notificatie: "BIBOB-2024-XXXXX scheduled for anonymization on [vernietigingsdatum]. Confirm to proceed."
  - Archivist klikt "Proceed with anonymization"
  - Voer uit:
    1. Alle BibobBetrokkene records → redact bsn, woonadres, geboortedatum (replace with "***ANONIMIZED***")
    2. BibobFormulier1/2 → redact sensitive question responses (antwoorden_json → anonymized form)
    3. LbbAdviesaanvraag.advies_tekst → DELETE (not anonymize, delete)
    4. Bewaar:
       - Dossier-nummer, aanleiding, conclusie (for statistics)
       - Besluit-type, datum (for audit)
    5. Stel BibobBewaarregel.anonimisering_geplanned = now, status = "anonymized"

**Acceptance:**
- ✓ PII redacted but dossier structure retained (for compliance audits)
- ✓ LBB-advies completely removed (not anonymized)
- ✓ All changes logged to auditlog irreversibly

---

### REQ-010-003: Vernietiging met tweede-handtekening

**GIVEN** vernietigingsdatum bereikt + dossier geanonimiseerd  
**WHEN** ArchivalService.scheduleForDestruction() triggered  
**THEN**:
  - Genereert notificatie naar archivist: "BIBOB-2024-XXXXX ready for destruction on [date]. Confirm second signature to proceed."
  - Archivist klikt "Confirm destruction"
  - Voer uit:
    1. DELETE BibobBetrokkene (alle records voor dit dossier)
    2. DELETE BibobFormulier1/2 (inhoud)
    3. DELETE LbbAdviesaanvraag (geheel)
    4. KEEP BibobBesluit (besluit-tekst) maar redact sensitive parts (feitelijke grondslag → "Anonimized decision from BIBOB-2024-XXXXX")
    5. KEEP BibobBewaarregel (bewaarduur_basis, vernietigingsdatum, vernietigingslog)
    6. Stel vernietigingslog = "Destroyed by [archivist-name] on [date]. [Summary of what was deleted]."
  - Auditlog entry: DESTRUCTION_COMPLETE, BIBOB-2024-XXXXX, signed by archivist

**Acceptance:**
- ✓ Destruction requires two authorized users (coordinator-approved + archivist-confirmed)
- ✓ Vernietigingslog is immutable (append-only)
- ✓ Final decision accessible for statistics (anonimized form)

---

## REQ-011: Integration — shillinq payment blocking

### REQ-011-001: Payment freeze bij "ernstige-mate" conclusie

**GIVEN** BibobDossier.conclusie = "ernstige-mate-van-gevaar" EN onderzoeksfase ≠ "besloten"  
**WHEN** dossier actief blijft (NOT gesloten)  
**THEN**:
  - Stuur CloudEvent naar shillinq: `bibob.decision.payment-block`
    ```json
    {
      "leverancier_kvk": "12345678",
      "dossier_id": "BIBOB-2024-XXXXX",
      "reason": "Bibob-onderzoek lopend met ernstige risico's",
      "block_until_date": null
    }
    ```
  - Shillinq blokkeert alle uitgaande betalingen aan KvK 12345678

**Acceptance:**
- ✓ Block automatic on dossier-conclusie change
- ✓ Block remains until dossier gesloten or conclusie ≠ "ernstige-mate"

---

### REQ-011-002: Payment unblock bij gunning of afsluiting

**GIVEN** BibobDossier.conclusie = "lichte-mate" of "geen-bezwaar" OF dossier gesloten  
**WHEN** conclusie-wijziging opgeslagen  
**THEN**:
  - Stuur CloudEvent naar shillinq: `bibob.decision.payment-unblock`
    ```json
    {
      "leverancier_kvk": "12345678",
      "dossier_id": "BIBOB-2024-XXXXX"
    }
    ```
  - Shillinq lifts payment-block

**Acceptance:**
- ✓ Unblock automatic on status change
- ✓ No manual intervention required

---

## REQ-012: Cross-app integration — openzaak zaak registration

### REQ-012-001: Zaak-creatie bij dossier-aanmaak

**GIVEN** BibobDossier aangemaakt  
**WHEN** ObjectService.saveObject() succeeds  
**THEN**:
  - Roep OpenZaak API aan:
    ```
    POST /zaken/
    {
      "zaaktype": "https://...openzaak.../zaaktypes/bibob-onderzoek",
      "identificatie": "BIBOB-2024-XXXXX",
      "omschrijving": "BIBOB-onderzoek naar [leverancier]",
      "toelichting": "[aanleiding + risico-samenvatting]",
      "zaakstatus": "onderzoek",
      "einddatum": null,
      "resultaat": null
    }
    ```
  - Stel BibobDossier._openzaak_link = [OpenZaak zaak-URI]

**Acceptance:**
- ✓ Zaak created asynchronously (failure doesn't block bibob-dossier creation)
- ✓ Zaak status tracks dossier phase

---

### REQ-012-002: Zaak-closure bij dossier-closure

**GIVEN** BibobDossier.status = "gesloten"  
**WHEN** closure saved  
**THEN**:
  - PATCH OpenZaak zaak:
    ```
    {
      "zaakstatus": "gesloten",
      "einddatum": [closure date],
      "resultaat": [BIBOB-conclusie: "geen-bezwaar" / "lichte-mate" / "ernstige-mate"],
      "toelichting": "[append summary of final decision]"
    }
    ```

**Acceptance:**
- ✓ OpenZaak reflects BIBOB-status end-to-end

---

## REQ-013: Audit trail completeness

### REQ-013-001: Auditlog events

**GIVEN** any state change on BibobDossier, BibobFormulier1/2, LbbAdviesaanvraag, BibobBesluit  
**WHEN** ObjectService.saveObject() called  
**THEN**:
  - AuditTrailService logs automatically:
    - User, timestamp, action (create/update/delete)
    - Before/after snapshot (for form responses: before vs after antwoorden_json)
    - IP address
    - Session ID

**Acceptance:**
- ✓ Auditlog enabled automatically by OpenRegister ObjectService
- ✓ No additional instrumentation required in app code
- ✓ Auditlog is append-only, immutable after 24 hours (configurable)

---

## REQ-014: Compliance validation

### REQ-014-001: Completeness check before decision signature

**GIVEN** BibobBesluit template ready for signing  
**WHEN** bestuurder klikt "Ter ondertekening aanbieden"  
**THEN**:
  - Valideer:
    1. Alle vier Awb-secties ≥ min-length: ✓
    2. Besluittype selected: ✓
    3. Onderzoeks-fases compleet:
       - If onderzoeksfase = "formulier-1": eerste_indicatie must be set: ✓
       - If onderzoeksfase = "formulier-2": voorlopige_conclusie must be set: ✓
       - If onderzoeksfase = "lbb-advies": advies_conclusie must be set: ✓
    4. Zienswijze-termijn expired (if applicable): ✓
    5. Leverancier-details in dossier complete: ✓
  - Fail any check → block with specific error message

**Acceptance:**
- ✓ No decision can be signed prematurely
- ✓ Prevents procedurally-defective decisions
