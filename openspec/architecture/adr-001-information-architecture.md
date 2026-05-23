# ADR-001: Information Architecture — purchaseq

**Status:** Accepted
**Date:** 2026-05-23
**Source:** Lifted from the cross-app IA design doc
`/tmp/ia-small5.md` (Small-Five IA, designed 2026-05-22), purchaseq
section.

## Context

purchaseq is the procurement suite for Dutch government and large
MKB: catalog-based ordering, three-way-match invoicing via Peppol,
formal aanbestedingen via TenderNed, raamovereenkomsten with
minicompetitie, supplier onboarding incl. BIBOB and MVI/SROI, and
inhuur-derden with Wet DBA/WNT toetsing. Where shillinq sells the
goods, purchaseq buys them.

The capability set is large (~13 specs across catalog-purchase,
formal tender, supplier mastering, and Peppol/TenderNed adapters).
A naive 1-spec-equals-1-menu layout would yield ten-plus top-level
items, with adapter-specs (TenderNed, Peppol) competing for sidebar
real-estate against the operational core. That pattern plagues
legacy gemeente-inkooppakketten: the inkoper is faced with a
sidebar wall and cannot find "bestel uit catalogus".

The Small-Five IA exercise (2026-05-22, covering financeq,
purchaseq, planix, scholiq, openbuilt) deliberately bounded each
app between four and six top-level menus, with tier-suffixed specs
and adapter-specs demoted to sub-pages, tabs, widgets, or settings.
For purchaseq this lands on **four** top-level menus:

1. Inkopen (Catalog & Orders) — operational core where 80% of users live
2. Aanbestedingen — formal-tender surface for inkopers and juristen
3. Leveranciers — supplier master data, onboarding, BIBOB, risk
4. Beheer — configuration, adapters, connectors, rollen

Catalog-purchase is the daily operational core; aanbestedingen is
the formal-tender side; leveranciersbeheer holds master data and
risk profile. Adapters to TenderNed and Peppol are configuration
in Beheer, not user-facing menus — the inkoper sees "Publiceren"
as an action that hides the adapter. purchaseq writes
verplichtingen to financeq, consumes leverancier-stamdata from
openregister, and feeds contract-deadlines to planix so
contract-management has a kanban view without owning the contracts
themselves.

This ADR captures the cross-cutting IA rules for purchaseq so that
every individual spec, change proposal, and frontend feature in
this repository conforms to the same layout without re-deciding it
each time.

## Decision

The purchaseq frontend ships **four top-level menus + a hidden
Beheer section**, governed by the numbered rules below. Every new
spec, page, or capability MUST map into this structure; new
top-level menus require an ADR amendment.

### 1. Adapters are settings, never menus

Adapters (TenderNed, Peppol, banking, KVK, financeq-bridge) live
as configuration under **Beheer > Connectors**. The user-facing
action ("Publiceren naar TenderNed", "Factuur ontvangen via
Peppol", "Verplichting boeken in financeq") hides the adapter
behind a domain-meaningful verb. No adapter ever gets its own
top-level menu, sidebar group, or "Integrations" hub — that anti-
pattern surfaces plumbing as a destination and forces the inkoper
to learn the protocol layer.

### 2. AI and analytics are widgets, not menus

AI suggestions (best supplier, best price, duurzaamste alternatief)
and uitgaven-analytics belong as widgets **on the operational
surface where the decision is made** — the catalogus card, the
order detail, the category-dashboard tab. They never live in a
sidebar "AI" or "Insights" silo. The user always sees them in
context of the artefact being decided, never as a separate noun.
This also keeps the `catalog-purchase-management-ai` and
`-analytics` specs from inflating the menu count.

### 3. Inhuur-derden is a sub-page of Inkopen, not a fifth menu

Inhuur-derden is a specialised purchase flow with extra Wet
DBA/WNT/WNRA checks — it is **not a different process**, it is a
contracted-purchase with additional juridical guards. It lives at
**Inkopen > Inhuur-derden** alongside Catalogus, Aanvragen,
Bestellingen, Ontvangsten, and Facturen. Do not split it out into
a "Personeel" / "HR" / "Inhuur" top-level menu just because the
legal frame differs — the verb is still "inkopen", the budget is
still consumed, the leverancier is still a leverancier.

### 4. Drempelbedragen live in Beheer, surface inline as compliance warnings

Goedkeuringsdrempels, aanbestedingsdrempels, en
rechtmatigheidsregels worden geconfigureerd in **Beheer >
Goedkeuringsdrempels** (per categorie + per kostenplaats), maar
worden **inline op de order/aanvraag getoond** als compliance
warning ("dit bedrag vraagt een onderhandse aanbesteding bij drie
partijen", "dit bedrag overschrijdt de Europese drempel — start
een aanbesteding"). De inkoper leert de regel **in context** van
de beslissing, niet in een aparte regelschermen. Same for MVI/SROI
criteria: configured in Beheer, surfaced as inline guidance on the
aanbesteding-eisen tab.

### 5. Tier-suffixed and variant specs collapse into the parent module

Specs with `-other-t1`, `-other-t2`, `-other-t3`, `-ai`,
`-analytics` suffixes are sub-pages, tabs, or widgets of their
parent module (typically catalog-purchase-management) — never
top-level menus. Concretely: the seven
`catalog-purchase-management*` variants all fold into Inkopen (one
as the menu, two as widgets, three as sub-pages, one as embedded
tab on bestelling > Goedkeuring). This rule keeps the inkoper's
sidebar under five items even as the spec count grows.

### 6. Beheer is hidden from reguliere gebruikers

Beheer is the **only place adapters worden zichtbaar gemaakt** and
the only place where catalog-feeds, drempelbedragen,
MVI-criteria, BIBOB-templates, vragenlijst-templates, en rol-
configuratie wonen. Reguliere medewerkers (inkoper, budgethouder,
ontvanger) zien Beheer **niet**. Only inkoop-coordinatoren and
admins have the role to see it. This protects the daily-ops UI
from configuration noise.

### 7. Canonical mapping table

Every spec in this repository maps to exactly one placement
(menu / sub-page / tab / widget / settings) per the table below.
New specs MUST add a row before being merged; new placements
require an ADR amendment.

| spec_slug | placement | parent | rationale |
|---|---|---|---|
| `catalog-purchase-management` | menu | Inkopen | operational core |
| `catalog-purchase-management-ai` | widget | Inkopen > Catalogus | AI suggestion overlay |
| `catalog-purchase-management-analytics` | widget | Inkopen (dashboard tab) | spend analytics on the orders surface |
| `catalog-purchase-management-other-t1` | sub-page | Inkopen > Aanvragen | non-catalog request flow |
| `catalog-purchase-management-other-t2` | sub-page | Inkopen > Ontvangsten | goods receipt branch |
| `catalog-purchase-management-other-t3` | tab | Inkopen > Bestelling > Goedkeuring | approval flow embedded |
| `peppol-ubl-inkoop-factuur-ontvangst` | sub-page | Inkopen > Facturen | 3-way-match lives under invoices |
| `inhuur-derden-wnra-wnt` | sub-page | Inkopen > Inhuur-derden | hiring is a special-case purchase (see rule 3) |
| `raamovereenkomst-minicompetitie` | sub-page | Aanbestedingen > Raamovereenkomsten/Minicompetities | framework lots |
| `mvi-sroi-aanbesteding` | sub-page | Aanbestedingen > MVI-SROI-eisen | sustainability criteria branch |
| `supplier-onboarding-vragenlijst` | sub-page | Leveranciers > Onboarding | onboarding workflow |
| `bibob-toetsing-leveranciers` | sub-page | Leveranciers > BIBOB-toetsing | integrity check |
| `tenderned-publicatie-adapter` | settings | Beheer > Connectors > TenderNed | adapter, not user menu (see rule 1) |

The thirteen specs in this app collapse into **4 menus + Beheer**:
seven catalog-purchase variants fold into Inkopen, all formal-
tender specs sit under Aanbestedingen, the two supplier-facing
specs sit under Leveranciers, and the only adapter
(tenderned-publicatie-adapter) is correctly hidden behind a Beheer
settings page. The Peppol-UBL spec lives as a sub-page under
Inkopen > Facturen because inkomende facturen are operationeel
werk, niet configuratie.

### 8. Cross-app chain placements are widgets/links, not menus

purchaseq writes verplichtingen to financeq, consumes leverancier-
stamdata from openregister, and feeds contract-deadlines to
planix. These chain-relations surface as **widgets on the relevant
detail screen** (verplichting-status op de bestelling,
contract-deadline-link naar planix) or as deep-links from the
order to the consuming app — **never as a "Chains" or
"Integrations" top-level menu**. The user sees the result inline
where the work happens; the integration is invisible.

## Consequences

### Positive

- The inkoper's sidebar stays at four items; no
  ten-Inkopen-tabs sprawl.
- Adapter specs (TenderNed, Peppol) can be added or replaced
  without UI churn — they live behind verbs in Beheer.
- AI/analytics specs land where decisions are made, raising
  adoption (decision-support is contextual, not exploratory).
- Inhuur-derden + Wet DBA stays inside the inkoop-mental-model;
  juristen still get the extra checks without a parallel HR
  sidebar.
- Drempelbedragen-as-inline-warning teaches rechtmatigheid in
  context, reducing audit findings later.
- New specs have a deterministic placement: every new spec adds a
  row to the mapping table or proposes an ADR amendment.

### Negative / trade-offs

- A maker adding a new adapter (e.g. Mercell, GHX) must remember
  to surface the user action under Beheer > Connectors, not as a
  menu. The ADR text makes this explicit, but it relies on
  reviewer discipline.
- Tier-suffixed specs lose visibility — a `-other-t4` capability
  is one click deeper than a top-level menu would be. The
  trade-off is intentional: the operational core stays clean.
- Inhuur-derden buried under Inkopen may surprise HR-oriented
  users; we mitigate with prominent landing-card on the Inkopen
  home (sub-page tile, not sidebar item).
- Mapping table maintenance becomes mandatory: every PR that adds
  a spec must update this ADR. We accept this as the cost of
  keeping IA coherent.

### Forward links

- Per-spec frontend changes reference this ADR in their
  `## Architecture` section (rule numbers, e.g. "per ADR-001
  rule 1, the Peppol adapter is hidden under Beheer >
  Connectors").
- Future top-level menus require an ADR amendment (ADR-001a, or
  a successor ADR-NNN with `**Supersedes:** ADR-001`).
- Cross-app IA decisions for financeq, planix, scholiq, and
  openbuilt live in the corresponding ADRs in their own repos;
  the shared source-of-truth document is the Small-Five IA
  doc (`/tmp/ia-small5.md`, 2026-05-22).
