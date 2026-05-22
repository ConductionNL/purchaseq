---
app: purchaseq
spec: inhuur-derden-wnra-wnt
title: Inhuur Derden met Wet DBA en WNT Toetsing — Requirements & Scenarios
status: draft
version: 1.0
date: 2026-05-22
---

# Specifications: Inhuur Derden met Wet DBA en WNT Toetsing

## Requirement Formatting

Each requirement numbered **REQ-XXX-NNN** where:
- **XXX** = ISO 639-3 language code or numeric spec branch (e.g., `DBA`, `WNT`, `WRK`)
- **NNN** = sequence number within that branch

Requirements defined in **GIVEN/WHEN/THEN** (Gherkin) format, with acceptance criteria and test scenarios.

---

## DBA Assessment & Risk Scoring (Branch: DBA)

### REQ-DBA-001: Inhuuropdracht aanmaken met verplichte DBA-assessment

**Goal**: Every new hire decision is immediately subject to a standardized 12-question DBA risk assessment. No contract can progress to "approved" status without a completed assessment.

**GIVEN** een ingelogde gebruiker met rol "inkoper" of "afdelingshoofd"  
**WHEN** deze een nieuwe inhuuropdracht aanmaakt met inhuurtype "zzp" en op "Opslaan" klikt  
**THEN**  
1. Het systeem validert alle verplichte velden (opdrachtgever, opdrachtnemer, startdatum, uurtarief).
2. Een DBA-assessment-record wordt aangemaakt met status "concept".
3. Een inline formulier-tab "DBA Beoordeling" verschijnt met 12 standaardvragen.
4. Het systeem blokkeert status-overgang naar "goedgekeurd" zolang DBA-conclusie ontbreekt.
5. Bij DBA-conclusie "rood" blokkeert het systeem verdere status-transities tot drie-handtekeningen-flow is voltooid.
6. Audit trail: contract-aanmaak, DBA-aanmaak, status-blokkering — alle timestamps + user.

**Acceptance Criteria**:
- [ ] DBA form renders inline (CnFormDialog) met alle 12 vragen + toelichting.
- [ ] Systeem berekent risicoscore na vraag-invulling (0-100 schaal).
- [ ] Status-blokkade: "goedgekeurd" button is disabled tot assessment compleet.
- [ ] Assessment-status "concept" tot "afgerond" na opslaan.
- [ ] Rood assessment markeert contract automatisch voor escalatie (flag = true).

**Test Scenario 1**: Happy path (Green assessment)  
1. Create ZZP contract (alle velden ingevuld).
2. Fill DBA form, all answers indicate low control/risk.
3. Risk score <30, conclusie = "groen".
4. Status "goedgekeurd" button now enabled.
5. Save contract → status transitions to "goedgekeurd".
6. Verify audit log shows: contract create, DBA create, DBA assessment complete.

**Test Scenario 2**: Red assessment blocks approval  
1. Create ZZP contract.
2. Fill DBA form, answers indicate high control (gezag), low replaceability.
3. Risk score >60, conclusie = "rood".
4. Status button remains disabled with tooltip "Wacht op drie-handtekeningen-goedkeuring".
5. Attempt to click "goedgekeurd" → error alert "Red DBA requires authorization".
6. Verify contract status = "in-onderzoek" until three signatures obtained.

---

### REQ-DBA-002: Standaard DBA-vragenlijst per Belastingdienst-kader

**Goal**: Ensure the 12-question DBA assessment strictly follows Belastingdienst guidelines (gezag, instructiemacht, vervangbaarheid, ondernemersrisico, werkmiddelen, verzekeringen, veelvoudige opdrachtgevers, tarief, factuurfrequentie).

**GIVEN** een DBA-assessment-formulier geopend in een inhuuropdracht  
**WHEN** de gebruiker de vragen bekijkt  
**THEN**  
1. De 12 vragen verschijnen in vaste volgorde en Nederlands.
2. Elke vraag toont:
   - Vraag-ID (Q1-Q12).
   - Vraag-tekst (Belastingdienst-exact).
   - Antwoordopties: Ja / Nee / Deels / Onbekend.
   - Toelichting-veld (max 500 tekens).
   - Link naar Belastingdienst-gids (externe help).
3. Alle antwoorden zijn verplicht; submit button is disabled tot alle vragen beantwoord.
4. Risk-score wordt real-time berekend na elke antwoordwijziging.
5. Conclusie (groen/oranje/rood) wordt automatisch bepaald (groen <30, oranje 30-60, rood >60).

**Acceptance Criteria**:
- [ ] All 12 questions hardcoded in Dutch (niet configurabel voor deze spec).
- [ ] Antwoord-opties: enum {ja, nee, deels, onbekend}.
- [ ] Real-time risk calculation shown to user (score bar, color).
- [ ] Conclusie automatisch bepaald en getoond.
- [ ] Submit button disabled if any question unanswered.

**Test Scenario 1**: Complete assessment  
1. Open DBA form, all 12 questions visible.
2. Answer Q1-Q5 (low risk answers): score updates to ~10.
3. Answer Q6-Q12 (mixed answers): score updates to ~35, conclusie = "oranje".
4. Tooltip shows "Oranje: matig risico, monitoring aanbevolen".
5. Verify submit button enabled.
6. Click submit → assessment saved, risicoscore = 35, conclusie = "oranje".

---

### REQ-DBA-003: Periodieke her-beoordeling bij 6-maanden of budget-overschrijding

**Goal**: Long-running contracts (>6 months) or those exceeding initial budget by >25% must be re-assessed to detect creeping employment relationships.

**GIVEN** een actieve ZZP-inhuuropdracht  
**WHEN** één van de volgende triggers optreedt:  
  - De looptijd bereikt 6 maanden, OF  
  - De cumulatieve uitgaven (van Shillinq-facturen) het initiële geschat totaalbudget met >25% overschrijden  
**THEN**  
1. Het systeem genereert automatisch een notificatie aan de inkoper.
2. Een nieuwe DBA-assessment-record wordt aangemaakt met status "vereist".
3. Het systeem blokkeert goedkeuring van verdere Shillinq-facturen voor deze opdracht totdat her-assessment compleet.
4. Her-assessment-formulier toont dezelfde 12 vragen + extra veld: "Zijn de gezagsrelatie of vervangbaarheid veranderd sinds begin contract?"
5. Na heronder zoek: als risicoscore is gestegen van groen naar oranje/rood, wordt escalatie getriggerd.
6. Audit trail: trigger-datum, trigger-reden, her-assessment-aanvraag, her-assessment-resultaat.

**Acceptance Criteria**:
- [ ] Scheduled job checks InhuurOpdracht records daily (trigger 1: 6 maanden na startDatum).
- [ ] Shillinq webhook listener checks trigger 2: daadwerkelijkeUitgaven > geschatTotaalbudget * 1.25.
- [ ] New DbaAssessment record created, linked to same contract.
- [ ] Invoice approval blocked (CnDetailPage invoice-tab, approve button disabled).
- [ ] Blocker message: "Vereist her-assessment DBA voordat facturing kan doorgaan".
- [ ] After reassessment complete: if score increased, auto-flag for escalation review.

**Test Scenario 1**: 6-month trigger  
1. Create ZZP contract, startDatum = 2026-03-01.
2. Verify status = "actief", no re-assessment in place.
3. Simulate date to 2026-09-01 (6 months).
4. Scheduled job runs at 00:00.
5. Check contract detail: banner appears "Her-assessment DBA vereist".
6. New DbaAssessment record created with status "vereist".
7. Try to approve pending Shillinq invoice → button disabled, tooltip "Her-assessment vereist".
8. Complete her-assessment, submit.
9. Invoice approval button now enabled.

**Test Scenario 2**: Budget overshoot trigger  
1. Create ZZP contract, geschatTotaalbudget = 50,000.
2. Add Shillinq invoices totaling 62,500 (125% of budget).
3. When invoice #4 is received (total now exceeds 50,000 * 1.25 = 62,500):
4. Webhook listener triggers her-assessment requirement.
5. Verify DbaAssessment created, invoice approval blocked.

---

## Model Agreement & Validation (Branch: MOD)

### REQ-MOD-001: Modelovereenkomst-koppeling verplicht voor ZZP

**Goal**: ZZP and contractor hires must reference a valid model agreement per Belastingdienst or bespoke framework. The system enforces this requirement and guides users through the three registration paths.

**GIVEN** een ZZP-inhuuropdracht in status "in-onderzoek"  
**WHEN** de gebruiker probeert status naar "goedgekeurd" te zetten terwijl:  
  - Opdrachtnemer.modelovereenkomstKenmerk leeg, OR  
  - Opdrachtnemer.datumLaatsteModelovereenkomstCheck > 5 jaar geleden  
**THEN**  
1. Het systeem toont een blokkende dialoog met titel "Modelovereenkomst Verplicht".
2. Drie keuzes worden gepresenteerd (radio buttons):
   - (A) Selecteer bestaande Belastingdienst-modelovereenkomst uit dropdown.
   - (B) Upload branche-modelovereenkomst (PDF) ter beoordeling.
   - (C) Bereid individuele modelovereenkomst-aanvraag voor (link naar externe proces).
3. Voor keuze A: dropdown toont top 20 actieve Belastingdienst-modellen (gefilterd op opdrachtnemers industrie/sector).
4. Voor keuze B: file-upload veld (max 10 MB, PDF/DOCX), toont preview.
5. Voor keuze C: link opent externe gids + e-mailsjabloon (geen verder système-stappen).
6. Na keuze + bevestiging: Opdrachtnemer-record updated met modelovereenkomstKenmerk + datumCheck.
7. Dialoog sluit, status-overgang "goedgekeurd" nu allowed.
8. Audit trail: dialoog-toon, gebruikerskeuze, modelovereenkomst-update.

**Acceptance Criteria**:
- [ ] Status-overgang "goedgekeurd" validation checks Opdrachtnemer model-agreement fields.
- [ ] Blokkade dialoog toont iff kenmerk leeg OR datumCheck > 5 jaar.
- [ ] Three radio-button options rendered.
- [ ] Option A: dropdown query Belastingdienst model registry (via API of hardcoded list).
- [ ] Option B: file upload (client-side PDF validation, backend storage).
- [ ] Option C: static guidance text + mailto link.
- [ ] On save: Opdrachtnemer.modelovereenkomstKenmerk + datumLaatsteModelovereenkomstCheck updated.
- [ ] Audit: dialog_shown, choice_made, opdrachtnemer_updated.

**Test Scenario 1**: Select existing Belastingdienst model  
1. Create ZZP contract, Opdrachtnemer has no model reference.
2. Try to approve → blokkade dialoog appears.
3. Select radio option A.
4. Dropdown shows 5+ Belastingdienst models.
5. User selects "BD-2024-003-ICT".
6. Click "Bevestig Modelovereenkomst".
7. Opdrachtnemer record updated: modelovereenkomstKenmerk = "BD-2024-003-ICT", datumLaatsteModelovereenkomstCheck = today.
8. Dialoog sluit, contract status transitions to "goedgekeurd".

**Test Scenario 2**: Upload branch model  
1. Create ZZP contract, Opdrachtnemer industrie = "Onderwijs".
2. Try to approve → blokkade dialoog.
3. Select radio option B.
4. Upload PDF "CAO-Onderwijsberoepen-Model-2024.pdf".
5. Backend stores file, associates with Opdrachtnemer + contract.
6. modelovereenkomstType set to "branche-specifiek", kenmerk set to file-id.
7. Status transitions to "goedgekeurd", status changes to "in-review" (awaiting legal approval, optional).
8. (Later) Legal approves, Opdrachtnemer.datumLaatsteModelovereenkomstCheck stamped.

---

### REQ-MOD-002: Jaarlijkse modelovereenkomst-geldigheid-check

**Goal**: Model agreements expire or become invalid. Annual renewal ensures contracts always reference current agreements.

**GIVEN** een Opdrachtnemer met modelovereenkomstKenmerk ingesteld  
**WHEN** het contract verjaardag bereikt (contract-anniversary) of jaarlijkse batch-check loopt  
**THEN**  
1. Systeem query de Belastingdienst-registerAPI (of lokale kopie) naar geldigheid.
2. Indien kenmerk niet meer geldig: Opdrachtnemer-record marked "model-agreement-expired".
3. Een notificatie wordt verzonden naar de inkoper: "Modelovereenkomst [kenmerk] voor [opdrachtnemernaam] is verlopen".
4. Verdere invoicing geblokkeerd totdat nieuwe modelovereenkomst is registered.
5. Audit trail: validation date, result, status update.

**Acceptance Criteria**:
- [ ] Scheduled job runs monthly or on contract anniversary.
- [ ] KvK adapter (or API mock) called to validate model-agreement kenmerk.
- [ ] If invalid: Opdrachtnemer.modelovereenkomstKenmerk stays same, but flag added (e.g., deprecated_at timestamp).
- [ ] Invoice approval blocked if agreement expired.
- [ ] Notification sent to contract owner.
- [ ] Manual re-registration path available (REQ-MOD-001).

---

## WNT Wage-Cap Monitoring (Branch: WNT)

### REQ-WNT-001: Loongrens-aggregatie realtime per opdrachtnemer per jaar

**Goal**: Track cumulative payments to each contractor within a calendar year against the WNT norm. Invoice receipts from Shillinq trigger automatic aggregation.

**GIVEN** een actieve ZZP-inhuuropdracht bij een WNT-plichtige organisatie (wntRelevant = true)  
**WHEN** een nieuwe Shillinq-factuurregel binnenkomt voor deze opdrachtnemer in dezelfde kalenderjaar  
**THEN**  
1. Shillinq dba-compliance-marker webhook listener wordt getriggerd.
2. Het systeem aggregeert alle Shillinq-factuurregels voor deze Opdrachtnemer in het huidige kalenderjaar.
3. Een WntToets-record (of bestaande) wordt updated: totaleuiutkeringTotNuToe = som van alle factuurregels.
4. Percentage = totaleuiutkeringTotNuToe / wntNormJaar * 100.
5. Als percentage >= 80: notificatie naar Bestuurder + WNT-verantwoordelijke.
6. Als percentage >= 100: automatisch publicatieplicht-record aangemaakt (zie REQ-WNT-002).
7. Audit trail: factuurregel-ontvangst, aggregatie-resultaat, percentage-berekening, drempel-trigger.

**Acceptance Criteria**:
- [ ] Shillinq webhook listener on invoice created/paid.
- [ ] Query: all Shillinq invoices for this Opdrachtnemer in calendar year.
- [ ] WntToets record created (if none exists) or updated.
- [ ] Percentage calculation shown in WntToets.percentageNormBereikt.
- [ ] Threshold 80% check → NotificationService.notify("Bestuurder", "WNT-verantwoordelijke", message).
- [ ] Threshold 100% check → call REQ-WNT-002 (publication record creation).
- [ ] Audit log: webhook_triggered, invoice_id, totals, percentage, thresholds_crossed.

**Test Scenario 1**: Aggregation up to 80%  
1. Create ZZP contract, wntRelevant = true, wntNormJaar = 150,000.
2. Receive Shillinq invoice #1: 50,000.
3. Webhook triggers, WntToets created: totaleuiutkeringTotNuToe = 50,000, percentage = 33%.
4. Receive Shillinq invoice #2: 50,000.
5. WntToets updated: totaleuiutkeringTotNuToe = 100,000, percentage = 67%.
6. Receive Shillinq invoice #3: 20,000.
7. WntToets updated: totaleuiutkeringTotNuToe = 120,000, percentage = 80%.
8. Notification sent to Bestuurder: "Contractant X heeft 80% van WNT-norm bereikt (€120k / €150k)".
9. Contract detail shows yellow warning: "Approaching WNT limit".

**Test Scenario 2**: Crossing 100% threshold  
1. WntToets at 80% (scenario 1).
2. Receive Shillinq invoice #4: 35,000.
3. WntToets updated: totaleuiutkeringTotNuToe = 155,000, percentage = 103%.
4. Notification sent: "Contractant X overschrijdt WNT-norm (€155k / €150k, 103%)".
5. Publication record auto-created (see REQ-WNT-002).
6. Contract detail shows red alert: "WNT-norm exceeded, publication required".

---

### REQ-WNT-002: Publicatieplicht-record automatisch aanmaken bij >100%

**Goal**: When a contractor's compensation crosses the WNT cap, a publication disclosure record is auto-created with all required fields for annual board report.

**GIVEN** een WntToets-record waar percentageNormBereikt > 100  
**WHEN** de status van publicatieplichtGetriggerd automatisch gezet wordt naar true  
**THEN**  
1. Een PublicatieRecord wordt aangemaakt met de volgende verplichte velden:
   - Contractantnaam (van Opdrachtnemer).
   - Functie/Opdrachtomschrijving.
   - Dienstbetrekking-aard (inhuurType, bijv. "ZZP").
   - Totale beloning (WntToets.totaleuiutkeringTotNuToe).
   - Einddatum (InhuurOpdracht.geplande EindDatum of actueleEindDatum).
   - Contractstartdatum (InhuurOpdracht.startDatum).
2. PublicatieRecord krijgt status "concept" voor review door WNT-verantwoordelijke.
3. Notificatie: WNT-verantwoordelijke krijgt alert "Publicatieplicht getriggerd voor [contractantnaam]".
4. In het jaarlijkse WNT-export (REQ-WNT-004) worden alle PublicatieRecords opgenomen.
5. Audit trail: trigger-datum, publicatierecord-aanmaak, velden-gepopuleerd.

**Acceptance Criteria**:
- [ ] New entity: PublicatieRecord schema (name, function, employment_type, remuneration, start_date, end_date, status).
- [ ] WntToets.publicatieplichtGetriggerd = true triggers auto-creation of PublicatieRecord.
- [ ] All required fields populated from linked InhuurOpdracht + WntToets.
- [ ] Status = "concept" (awaiting review).
- [ ] Notification sent to WNT officer.
- [ ] PublicatieRecords persisted and queryable for annual export.

**Test Scenario**: Publication record generation  
1. WntToets shows percentage = 105% (from REQ-WNT-001, Test Scenario 2).
2. System auto-creates PublicatieRecord:
   - naam = "Jan de Wit"
   - functie = "Data architect"
   - dienstbetrekking_aard = "ZZP"
   - beloning = €155,000
   - startdatum = 2026-03-01
   - einddatum = 2026-09-01
3. Status = "concept".
4. WNT-verantwoordelijke receives notification.
5. In WntToets detail view, publicatieplichtGetriggerd = true, link to PublicatieRecord shown.
6. When annual export runs (REQ-WNT-004), record appears in output.

---

### REQ-WNT-003: WNT-norm jaarlijks configureerbaar

**Goal**: WNT salary caps change annually. The system allows configuration of per-year norms to ensure accurate aggregation.

**GIVEN** een WNT-verantwoordelijke in de instellingen  
**WHEN** deze de WNT-norm voor een toekomstig jaar wil aanpassen (bijv. 2027: €155,000)  
**THEN**  
1. Een instellingen-pagina "WNT Normering" toont een tabel met jaren + normen.
2. Gebruiker kan voor elk jaar de norm (EUR) invoeren.
3. Wijzigingen worden opgeslagen in de app config (IAppConfig, niet OpenRegister).
4. Bij toekomstige aggregaties (REQ-WNT-001) worden de aangepaste normen gebruikt.
5. Audit trail: config-wijziging, oude waarde, nieuwe waarde, user, timestamp.

**Acceptance Criteria**:
- [ ] Settings page: WNT Normering table (year, norm_eur, last_updated_by, last_updated_date).
- [ ] Edit form for each year (inline or modal).
- [ ] Save updates to IAppConfig (ConfigurationService).
- [ ] Queries for WntToets aggregation use current config.
- [ ] Audit trail on config change.
- [ ] Default norms pre-populated (2026, 2027, etc., based on official WNT list).

---

## WAADI & Vendor Validation (Branch: VEN)

### REQ-VEN-001: WAADI-registratie-controle bij uitzendinhuur

**Goal**: Temporary staffing and secondment vendors must be registered with WAADI (intermediary registration per Dutch law). The system validates this at vendor selection.

**GIVEN** een gebruiker selecteert een leverage voor inhuurtype "uitzend" of "detachering"  
**WHEN** de leverancier-selector-form ("Kies Opdrachtnemer") wordt ingevuld  
**THEN**  
1. Systeem query de OpenConnector KvK-adapter met het KvK-nummer van de leverancier.
2. KvK-adapter retourneert: naam, vestigingsadres, WAADI-registratiestatus (true/false), WAADI-nummer.
3. Indien WAADI-registratie = false: blokkade dialoog getoond met rode foutmelding:
   - "Leverancier [naam] heeft geen geldige WAADI-registratie".
   - Link naar KvK-pagina van leverancier.
   - Alternatief: "Controleer KvK-gegevens van de leverancier en probeer opnieuw".
4. Gebruiker kan niet verdergaan totdat een geldige WAADI-leverancier is geselecteerd.
5. Opdrachtnemer-record updated: waadiRegistratie = true, waadiNummer = [geretourneerde nummer].
6. Audit trail: KvK-query, WAADI-status, validation-result.

**Acceptance Criteria**:
- [ ] Vendor selector form includes a "WAADI Check" button or auto-triggers on KvK entry.
- [ ] OpenConnector KvK adapter called with vendor KvK number.
- [ ] WAADI status extracted from KvK response.
- [ ] If false: blocking error dialog shown, submit button disabled.
- [ ] If true: Opdrachtnemer.waadiRegistratie = true, waadiNummer stored.
- [ ] Audit: kvk_query, waadi_status, validation_result.

**Test Scenario 1**: Valid WAADI vendor  
1. Create new ZZP contract, inhuurType = "uitzend".
2. Vendor selector: enter KvK number "67345678" (StaffForce Interim BV, from design.md seed).
3. Click "Controleer WAADI".
4. KvK adapter returns: name = "StaffForce Interim BV", waadi_registration = true, waadi_number = "UWV-WG/20245/1".
5. Opdrachtnemer record updated, WAADI fields populated.
6. No blocking error, form can proceed.
7. Audit: waadi_check_passed, waadi_number_stored.

**Test Scenario 2**: Missing WAADI registration  
1. Create new ZZP contract, inhuurType = "detachering".
2. Vendor selector: enter KvK number "99999999" (fictional, no WAADI).
3. Click "Controleer WAADI".
4. KvK adapter returns: waadi_registration = false.
5. Blocking error dialog appears: "Leverancier heeft geen geldige WAADI-registratie. [link to KvK]".
6. Submit button disabled.
7. Audit: waadi_check_failed, vendor_kvk_99999999.

---

### REQ-VEN-002: Jaarlijkse WAADI-geldigheid-check

**Goal**: WAADI registrations can lapse. Existing contracts' vendors must be validated annually.

**GIVEN** een actieve inhuuropdracht met inhuurType "uitzend" of "detachering"  
**WHEN** het contract-verjaardag bereikt (1 jaar na startDatum)  
**THEN**  
1. Scheduled job query OpenConnector KvK-adapter voor de Opdrachtnemer.
2. Controleer huidige WAADI-status.
3. Indien status is veranderd van true → false:
   - Mark contract "geblokkeerd" met reden "WAADI verlopen".
   - Notificatie aan inkoper: "WAADI-registratie van [opdrachtnemernaam] is verlopen".
   - Verdere invoicing geblokkeerd totdat leverancier is bijgewerkt/vervangen.
4. Audit trail: validation-date, old-status, new-status, blocker-applied.

**Acceptance Criteria**:
- [ ] Scheduled job runs on contract anniversary.
- [ ] KvK adapter queried for WAADI status update.
- [ ] If status changed from true → false: contract.status = "geblokkeerd".
- [ ] Notification sent to contract owner.
- [ ] Invoice approval blocked if contract blocked.
- [ ] Audit: anniversary_check, waadi_status_change, blocker_applied.

---

## Hiring Ratio Monitoring (Branch: RAT)

### REQ-RAT-001: Inhuur-ratio dashboard per kostenplaats

**Goal**: Controllers and executives monitor external hiring as a percentage of total workforce spend per cost center, with historical trends and alarm thresholds.

**GIVEN** een ingelogde controller of bestuurder  
**WHEN** deze het inhuur-dashboard opent (menu: "Governance > Inhuur Ratio")  
**THEN**  
1. Dashboard toont een kaart per kostenplaats:
   - Titel: kostenplaats + omschrijving.
   - Huidgende ratio: grote getallen, bijv. "16.0%", kleurgecodeerd (groen <30%, geel 30-50%, oranje 50-70%, rood >70%).
   - Drempel indicator: "Norm: 25% | Actueel: 16% | ✓ Compliant".
   - 8-kwartaal trendgrafiek (lijn-diagram, y-as = percentage, x-as = kwartaal).
2. Bij klik op een kaart: drill-down-tabel verschijnt met:
   - Contract-ID.
   - Opdrachtnemernaam.
   - Hirdate.
   - Spend YTD (EUR).
   - DBA-status (groen/oranje/rood).
   - WNT-vlag (ja/nee).
   - Status (actief/verlengd/afgerond).
3. Filters beschikbaar: datum-bereik, kostenplaats, ratio-status (compliant/non-compliant).
4. Export knop: CSV van huidi gen ratio's + trend-data.

**Acceptance Criteria**:
- [ ] Dashboard page with CnDashboardPage + GridStack.
- [ ] InhuurRatio records calculated/updated nightly (scheduled job).
- [ ] Color coding based on ratio % vs. drempel.
- [ ] 8-quarter trend line rendered (CnChartWidget).
- [ ] Drill-down table (CnDataTable) filtered by kostenplaats.
- [ ] Filters work (date range, cost center, compliance status).
- [ ] Export CSV button functional.

**Test Scenario 1**: Dashboard load + drill-down  
1. Login as controller.
2. Navigate to "Governance > Inhuur Ratio".
3. Dashboard loads, shows 3 cost centers (CC-IT-INFRA, CC-HR-ADMIN, CC-STRATEGY).
4. CC-IT-INFRA shows: "16.0% (Norm: 25%) ✓ Compliant", green card.
5. CC-HR-ADMIN shows: "45% (Norm: 25%) ⚠ Above Norm", orange card.
6. Trend line shows CC-IT-INFRA has been 15-18% over 8 quarters.
7. Click CC-HR-ADMIN → drill-down table appears with 5 active/recent contracts.
8. See columns: ID, vendor name, hire date, spend YTD, DBA status, WNT flag, contract status.
9. One contract marked red DBA — click for details.
10. Audit: dashboard_viewed, drill_down_opened.

**Test Scenario 2**: Filter non-compliant  
1. Dashboard loaded.
2. Filter: "Show Non-Compliant Only".
3. Only CC-HR-ADMIN (45%) shown.
4. Export CSV: ratio_cc_hr_admin_2026.csv downloaded.
5. File contains: kostenplaats, quarter, ratio%, drempel, compliant (boolean).

---

## Three-Signature Approval (Branch: APP)

### REQ-APP-001: Drie-handtekeningen-flow voor rode DBA-uitkomst

**Goal**: "Red" DBA assessments (high sham-employment risk) require explicit written approval from three organizational roles: department head, controller, and executive. Each provides a required motivation.

**GIVEN** een inhuuropdracht met dbaConlusie = "rood"  
**WHEN** de status naar "goedgekeurd" wil overgaan  
**THEN**  
1. Systeem roept Decidesk workflow-engine aan met decision-type "drie-handtekeningen-rood-dba".
2. Drie approval-tasks worden aangemaakt (delegeren naar):
   - Taak 1: Afdelingshoofd (department head).
   - Taak 2: Controller (financieel beheerder).
   - Taak 3: Bestuurder (executive).
3. Elke taak bevat:
   - Contract-details (opdrachtnemernaam, tarief, DBA-score, reden-voor-rood).
   - Verplicht motiverings-veld (min. 50 tekens, max. 1000).
   - Goedkeuren / Weigeren buttons (Decidesk workflow).
4. Taak-assignees ontvangen notificatie (email + in-app).
5. Na goedkeuring: workflow wacht op ALLE DRIE signalen (unanimiteit).
6. Na alle drie goedgekeurd: InhuurOpdracht.red_overrideAutorisatie wordt ingevuld:
   - goedgekeurd = true.
   - autoriseerderAfdeling, autoriseerderController, autoriseerderBestuurder = user IDs.
   - datumGoedkeuring = timestamp.
   - motivering = samengestelde motiverings-teksten.
7. Contract-status mag nu naar "goedgekeurd" overgaan.
8. Audit trail: workflow-gestart, taak-aangemaakt, goedkeuringen-verzameld, contract-vrijgegeven.

**Acceptance Criteria**:
- [ ] Integration with Decidesk workflow engine (decision creation).
- [ ] Three task assignments (role-based routing).
- [ ] Motivation field validation (min/max length).
- [ ] All-three-sign gate (unanimous approval).
- [ ] Approval data persisted in InhuurOpdracht.red_overrideAutorisatie.
- [ ] Contract status transition unblocked after unanimous approval.
- [ ] Audit trail: workflow_initiated, task_created, approvals_collected, contract_approved.

**Test Scenario**: Red DBA authorization flow  
1. DBA assessment completed with score = 72, conclusie = "rood".
2. Inkoper tries to approve contract → blokkade: "Red DBA requires three-signature approval".
3. Decidesk workflow initiated.
4. Three tasks created:
   - Task 1 → Afdelingshoofd Sjaak.
   - Task 2 → Controller Anna.
   - Task 3 → Bestuurder Dirk.
5. Sjaak receives email notification, opens task in Decidesk.
6. Enters motivation: "Urgent operational need, staffing agency has established track record".
7. Clicks "Goed keuren".
8. Anna opens task, sees Sjaak's motivation + DBA details.
9. Enters motivation: "Financial review passed, 3-month term limits exposure".
10. Clicks "Goedkeuren".
11. Dirk opens task, sees both prior motivations.
12. Enters motivation: "Approved by department and finance, proceeding with 3-month pilot".
13. Clicks "Goedkeuren".
14. Workflow completes (all three signed).
15. InhuurOpdracht.red_overrideAutorisatie populated: goedgekeurd = true, names/dates/motivations filled.
16. Inkoper can now save contract with status = "goedgekeurd".
17. Audit log shows: workflow_initiated, task_create (3x), approvals_collected (3x), contract_approved.

---

## End-of-Contract Closeout (Branch: CLO)

### REQ-CLO-001: Eindafrekening met verplichte checklist

**Goal**: Contract conclusion is formalized through a mandatory checklist ensuring all compliance and operational loose ends are tied (final invoice, NDAs, access revocation, IT cleanup, knowledge transfer, exit eval).

**GIVEN** een inhuuropdracht die de einddatum nadert (binnen 30 dagen)  
**WHEN** de inkoper de contract-status naar "afronding" zet  
**THEN**  
1. Een afsluitings-checklist wordt getoond met 6 items:
   - [ ] Laatste factuurregel ontvangen.
   - [ ] Geheimhoudingsverklaring (NDA) geretourneerd.
   - [ ] Toegangspassen/badges ingenomen.
   - [ ] IT-accounts beëindigd.
   - [ ] Kennisoverdracht-document geüpload.
   - [ ] Eindevaluatie-formulier ingevuld.
2. Gebruiker kan items afvinken (checkboxen).
3. Archiveringsfunctie (status → "afgerond") is geblokkeerd totdat ALLE items afgevinkt.
4. Bij het aanvinken van "kennisoverdracht" en "eindevaluatie": file-upload velden verschijnen (optioneel link naar externe docs).
5. Na voltooiing checklist + status "afgerond": dossier kan gearchiveerd worden in Docudesk (REQ-DOC-001).
6. Audit trail: checklist-toon, item-afvinken (user + timestamp per item), archival-moment.

**Acceptance Criteria**:
- [ ] Status transition "actief" → "afronding" shows checklist dialog.
- [ ] Checklist items (6 total) rendered as checkboxes.
- [ ] Archival button ("Archiveer & Sluit") disabled until all items checked.
- [ ] File upload for knowledge transfer / exit eval.
- [ ] Audit: checklist_shown, item_checked (per item, user, timestamp).
- [ ] Audit: archival_initiated, contract_archived.

**Test Scenario 1**: Checklist completion  
1. Contract end-date = 2026-09-01.
2. Date advances to 2026-08-25 (within 30 days).
3. Inkoper opens contract, sees banner "Einddatum nadert, start afsluitingsproces".
4. Clicks "Ga naar Afsluitings-Checklist".
5. Checklist dialog appears with 6 items, all unchecked.
6. Check item 1 (laatste factuurregel ontvangen) — timestamp recorded.
7. Check items 2-4 (NDA, access, IT) — timestamps recorded.
8. Item 5 (kennisoverdracht): checkbox enabled, file-upload field appears.
9. User uploads "Knowledge-Transfer-DataArch.pdf".
10. Check item 5 — timestamp + file-id recorded.
11. Item 6 (eindevaluatie): file-upload, user uploads "Exit-Eval-DataArch.pdf".
12. Check item 6 — timestamp + file-id recorded.
13. All 6 items now checked, "Archiveer & Sluit" button enabled.
14. Click "Archiveer & Sluit".
15. Contract status → "afgerond", dossier marked for Docudesk archival.
16. Verify audit: all 6 checklist items logged with user/timestamp.

---

## Annual Board Report Export (Branch: EXP)

### REQ-EXP-001: WNT-jaarverslag export PDF + XML

**Goal**: WNT-obligated organizations export annual disclosure (PDF + XML) suitable for publication on website and submission to regulatory bodies.

**GIVEN** de WNT-verantwoordelijke wil het jaarverslag voorbereiden  
**WHEN** deze de export-functie start ("Export > Jaarverslag WNT [jaar]")  
**THEN**  
1. Gebruiker selecteert kalenderjaar (dropdown, huiij 2026, future/past years).
2. Systeem query alle PublicatieRecords en WntToets-records voor het geselecteerde jaar.
3. Export genereert twee bestanden:
   - **PDF**: Formatted report met:
     - Titel: "Jaarverslag Extern Personeel & WNT-Publicatie [Organisatienaam] [Jaar]".
     - Inleiding (template).
     - Tabel: alle WNT-contractanten (naam, functie, arbeidstype, totale beloning, startdatum, einddatum).
     - Samenvatting: totaal external hiring spend, ratio vs. total payroll, WNT-norm compliance status.
     - Footer: generatie-datum, organisatie-contact.
   - **XML**: Structured data (XML-schema per WNT-publicatiestandaard):
     - Container: `<jaarverslag><jaar>2026</jaar><organisatie>...</organisatie><contractanten>...</contractanten></jaarverslag>`.
     - Per contractant: `<contractant><naam>...</naam><functie>...</functie><arbeidstype>...</arbeidstype><beloning>...</beloning><start>...</start><einde>...</einde></contractant>`.
4. PDF gereed voor website-publicatie; XML gereed voor geautomatiseerde processing (toekomstig).
5. Audit trail: export-aanvraag, jaar, aantal records, bestanden-gegenereerd.

**Acceptance Criteria**:
- [ ] Export page: year selector (dropdown, 2020-2030).
- [ ] Query all PublicatieRecords + WntToets for year.
- [ ] PDF generation: use template + merge data.
- [ ] XML generation: valid per WNT schema (or simplified internal schema).
- [ ] Both files downloadable (zip archive or separate downloads).
- [ ] Audit: export_requested, year, record_count, files_generated.

**Test Scenario 1**: Annual export for 2026  
1. WNT officer navigates to "Export > Jaarverslag WNT".
2. Selects year 2026.
3. Click "Genereer Export".
4. System queries all PublicatieRecords for 2026 (2 found: Jan de Wit, another contractor).
5. System queries WntToets for all active contractors in 2026.
6. PDF generated: lists both contractors, totals, compliance status.
7. XML generated: structured data, all fields present.
8. Both files packaged in "Jaarverslag-WNT-2026.zip".
9. Download starts automatically.
10. Audit: export_requested_2026, 2_records, pdf_generated, xml_generated.

---

## Immutable Audit Trail (Branch: AUD)

### REQ-AUD-001: Auditlog onveranderlijk (hash-chain)

**Goal**: Every change to a contract, DBA assessment, or WNT record is cryptographically logged with full before/after transparency, immutable and tamper-evident via hash chaining.

**GIVEN** elke wijziging op InhuurOpdracht, DbaAssessment, of WntToets  
**WHEN** het record wordt opgeslagen  
**THEN**  
1. AuditTrailService (platform-provided, zie ADR-001) automatisch een log-entry aangemaakt met:
   - Entity-type (InhuurOpdracht / DbaAssessment / WntToets).
   - Entity-ID (opdrachtId / assessmentId / toetsId).
   - Veldnaam (wat is gewijzigd, bijv. "status", "risicoscore").
   - Oude waarde (before).
   - Nieuwe waarde (after).
   - Gebruiker (ID / email).
   - Timestamp (ISO 8601).
   - Hash van vorige entry (tamper-detection).
2. Alle entries opgeslagen in immutable log-tabel (geen delete/update permissions).
3. Exportfunctie beschikbaar: audit-trail per contract als JSON / CSV.
4. Tax auditor-rapport: reproduceerbare audit-trail incl. hashes.

**Acceptance Criteria**:
- [ ] AuditTrailService captures all changes (automatic via ObjectService hooks).
- [ ] Log fields: entity_type, entity_id, field_name, old_value, new_value, user, timestamp, prev_hash, current_hash.
- [ ] Hash chain validates: hash(entry[n-1]) == entry[n].prev_hash.
- [ ] Export endpoint: /api/contracts/{id}/audit-trail → JSON/CSV.
- [ ] Log table immutable (no DELETE/UPDATE on log rows).

**Test Scenario 1**: Hash-chain validation  
1. Create InhuurOpdracht (opdrachtId = "abc123").
2. AuditTrailService logs: create_event, entry_0, hash_0 = hash(entry_0).
3. Update contract: status → "goedgekeurd".
4. AuditTrailService logs: update_event, entry_1, hash_1 = hash(entry_1), prev_hash_1 = hash_0.
5. Update again: dbaConlusie → "rood".
6. AuditTrailService logs: entry_2, hash_2 = hash(entry_2), prev_hash_2 = hash_1.
7. Export audit trail: GET /api/contracts/abc123/audit-trail.
8. Response JSON shows 3 entries with hashes linked.
9. Verify: hash_1.prev_hash == hash_0, hash_2.prev_hash == hash_1.
10. Attempt to alter entry_1 in database → hash_2.prev_hash no longer matches, tamper detected.

---

## Cross-App Integration

### REQ-INT-001: Shillinq invoice webhook listener

**Goal**: Invoices received in Shillinq (with dba-compliance-marker flag) automatically aggregate cumulative payments to contractors and trigger WNT/ratio checks.

**GIVEN** een Shillinq-factuurregel arriveert met dba-compliance-marker ZZP-flag  
**WHEN** de webhook `/api/webhooks/shillinq-invoice` wordt getriggerd  
**THEN**  
1. Webhook listener ontvangt CloudEvents-payload (factuurnummer, amount, vendor-id, invoice-date).
2. Match vendor-id → Opdrachtnemer record.
3. Match Opdrachtnemer → active InhuurOpdracht (same startDatum ≤ invoice-date < endDate).
4. Aggregate: InhuurOpdracht.daadwerkelijkeUitgaven += invoice.amount.
5. Trigger WNT aggregation (REQ-WNT-001): update WntToets, check thresholds.
6. Trigger ratio recalculation (scheduled, not real-time).
7. Audit trail: webhook_received, vendor_matched, contract_matched, aggregation_updated.

**Acceptance Criteria**:
- [ ] Webhook endpoint `/webhooks/shillinq-invoice` (or via standard webhook registry).
- [ ] CloudEvents parsing (vendor-id, amount, date).
- [ ] Vendor+contract matching logic.
- [ ] InhuurOpdracht.daadwerkelijkeUitgaven updated.
- [ ] Call to REQ-WNT-001 (WNT aggregation).
- [ ] Audit: webhook_processed, invoice_aggregated.

---

### REQ-INT-002: OpenConnector KvK-adapter integration

**Goal**: Vendor registration data (KvK lookups, WAADI validation) seamlessly integrated via OpenConnector.

**GIVEN** een leverancier-selector-formulier of jaarlijkse WAADI-controle  
**WHEN** KvK-query nodig is  
**THEN**  
1. OpenConnector KvK-adapter aanroepen (method: `lookup_vendor(kvk_number)`).
2. Adapter query KvK Trade Register, retourneert:
   - Naam, adres, contact, WAADI-registratiestatus, WAADI-nummer.
3. Opdrachtnemer-record updated met KvK-data.
4. WAADI-validatie ingevoerd in blokkade-logica (REQ-VEN-001, REQ-VEN-002).
5. Audit trail: kvk_query, kvk_result, record_updated.

**Acceptance Criteria**:
- [ ] OpenConnector KvK service available and callable.
- [ ] Adapter method: lookup_vendor(kvk_number) → {name, address, waadi_status, waadi_number}.
- [ ] Integration in vendor selector + annual check.
- [ ] Error handling: if adapter unavailable, flag contract for manual review (do not block).

---

### REQ-INT-003: Decidesk three-signature workflow integration

**Goal**: "Red" DBA approvals routed through Decidesk decision engine for task distribution and sign-off tracking.

**GIVEN** een red DBA contract  
**WHEN** drie-handtekeningen-flow gestart wordt  
**THEN**  
1. Decidesk workflow aanroepen: `create_decision(type="drie-handtekeningen-rood-dba", contract_id, contract_details)`.
2. Decidesk maakt 3 tasks aan (afdelingshoofd, controller, bestuurder).
3. Upon approval: Decidesk callback notifies this system.
4. InhuurOpdracht.red_overrideAutorisatie populated.
5. Audit trail: decidesk_workflow_initiated, callback_received, approval_recorded.

**Acceptance Criteria**:
- [ ] Decidesk API available (decision creation, approval polling/callback).
- [ ] Three task assignments (role-based routing).
- [ ] Callback handling: update InhuurOpdracht upon unanimous approval.
- [ ] Error handling: if Decidesk unavailable, manual approval fallback path available.

---

### REQ-INT-004: HRMQ personnel lookup (hiring ratio)

**Goal**: Populate internal FTE counts for hiring ratio calculation from HRMQ personnel register.

**GIVEN** ratio-calculation scheduled job läuft  
**WHEN** InhuurRatio records für einen Kostenplatz aktualisiert werden  
**THEN**  
1. Query HRMQ employee register for all employees with kostenplaats = [cost center].
2. Calculate FTE (arbeidsuren / 40 hrs/week, or from HRMQ field).
3. Calculate total payroll (sum of salaries for filtered employees).
4. InhuurRatio.aantalFteIntern, kostenIntern populated.
5. Ratio calculation completes (see REQ-RAT-001).
6. Audit trail: hrmq_query, fte_count, payroll_total.

**Acceptance Criteria**:
- [ ] HRMQ API/service available for employee queries (by kostenplaats).
- [ ] FTE calculation based on arbeidsuren or HRMQ field.
- [ ] Payroll aggregation.
- [ ] Integration in ratio-calculation scheduled job.

---

### REQ-INT-005: Docudesk archival (end-of-contract storage)

**Goal**: Completed contracts and associated documents archived in Docudesk for 7-year retention and auditor access.

**GIVEN** een contract-afsluitings-checklist voltooid  
**WHEN** status → "afgerond"  
**THEN**  
1. Archival-service roept Docudesk aan: `create_dossier(contract_id, documents[])`.
2. Documents included: contract PDF, DBA assessment (PDF), WNT-toets (if applicable), knowledge-transfer-doc, exit-eval.
3. Docudesk dossier gets retention policy: 7 years (Belastingdienst audit window).
4. Dossier linked to InhuurOpdracht record (for future lookups).
5. Audit trail: archival_initiated, docudesk_dossier_created, dossier_id.

**Acceptance Criteria**:
- [ ] Docudesk API available (dossier creation, document upload).
- [ ] Archival triggered on contract status = "afgerond".
- [ ] Documents: contract, assessment, WNT record (as PDFs), knowledge-transfer, exit-eval.
- [ ] Retention policy: 7 years.
- [ ] Dossier-ID linked back to InhuurOpdracht.

---

## Non-Functional Requirements

### NFR-001: Performance

- Dashboard load time <2 seconds (CnChartWidget caching).
- Invoice aggregation <100ms per invoice (async webhook processing).
- Annual export generation <5 seconds (batch PDF/XML).

### NFR-002: Availability

- System availability: 99.5% (standard SLA).
- Webhook delivery: retry 3x with exponential backoff (Shillinq integrations).

### NFR-003: Security

- All audit-log queries require "auditor" or "administrator" role.
- Three-signature approvals use OpenID Connect (no custom auth).
- IBAN, postcode, BSN validation via schema types (no custom regex).
- DBA assessment data (gezag scores, risk) treated as sensitive (audit-log only, no export).

### NFR-004: Compliance

- Audit trail hash-chained (REQ-AUD-001), reproducible for tax audits.
- WNT publication records formatted per WNT schema.
- Data retention: contracts archived 7 years (Docudesk).
- GDPR: personal data (contractor name, address, IBAN) in OpenRegister with standard DPO controls.

---

## Test Coverage

| Feature | Unit Tests | Integration Tests | E2E Tests |
|---------|-------|---------|-----------|
| DBA Assessment (REQ-DBA-001, REQ-DBA-002, REQ-DBA-003) | Risk-score calc, threshold tests | Contract-create flow, database persistence | Form fill → save → status-block |
| Model Agreement (REQ-MOD-001, REQ-MOD-002) | Validation logic | KvK lookup, record update | Selector dialog → override flow |
| WNT Wage-Cap (REQ-WNT-001, REQ-WNT-002, REQ-WNT-003) | Aggregation logic, threshold detection | Shillinq webhook simulation, WntToets update | Invoice receive → notification → publication record |
| WAADI (REQ-VEN-001, REQ-VEN-002) | Validation logic | KvK lookup, contract block | Vendor select → WAADI check → block on fail |
| Hiring Ratio (REQ-RAT-001) | Ratio calc, sorting/filtering | HRMQ lookup, InhuurRatio update | Dashboard load, drill-down, export |
| Three-Signature (REQ-APP-001) | Workflow initiation, task creation | Decidesk integration, approval callbacks | Full approval flow (3 signers) |
| Closeout (REQ-CLO-001) | Checklist logic, item tracking | Docudesk archival | Checklist completion → archival |
| Annual Export (REQ-EXP-001) | PDF/XML generation, template merging | Query PublicatieRecords, export encoding | Year selection → download |
| Audit Log (REQ-AUD-001) | Hash chain, entry creation | Change detection, log persistence | Modify contract → audit entry logged |
