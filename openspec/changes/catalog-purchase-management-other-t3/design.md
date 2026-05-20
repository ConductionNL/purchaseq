# Design: catalog-purchase-management-other-t3

## Context

Shillinq is a Nextcloud app using OpenRegister for all domain data. The entities for catalog management, purchase requisitions, pricing rules, purchase orders, and quotations are pre-defined in the data model (ADR-000). This change builds the UI views, approval workflow hooks, pricing rule evaluation engine, and PO matching logic on top of those existing entities.

The platform provides CRUD, search, file management, audit trails, workflow management, and dashboard widgets out of the box (ADR-001). Custom code is scoped strictly to domain-specific business logic that the platform does not provide.

## Goals / Non-Goals

**Goals:**
- Guided catalog browsing with enriched attributes and pricing display
- Punch-out catalog integration via cXML/OCI
- Purchase requisition lifecycle with customizable intake forms and approval routing
- Rule-based discount engine with first-match / all-match execution modes
- 2-way and 3-way PO matching with configurable tolerance rules
- Quotation revision tracking and professional PDF generation
- Cost center assignment on orders and requisitions
- Sale order line sequence management and stock sourcing rules
- PO status tracking with timeline visualization

**Non-Goals:**
- Custom import/export (platform `ImportService`/`ExportService` handles this)
- Custom search (platform `IndexService` + `CnFacetSidebar` handles this)
- Custom RBAC (platform `AuthorizationService` + `PropertyRbacHandler` handles this)
- Bank reconciliation (separate change)
- Custom dashboard widgets (platform `CnDashboardPage` + `CnChartWidget` handles this)

## Architecture Decisions

### Decision 1: Catalog Browsing — CnIndexPage on ProcurementCatalog + CatalogItem

The catalog view uses `CnIndexPage` backed by `createObjectStore('catalog-item')`. `CatalogItem.properties` stores enriched attributes — classification codes (UNSPSC), standardized descriptions, unit of measure, and stock status. Faceted navigation uses `CnFacetSidebar` with `FacetBuilder` on category, supplier, classification code, and price range. Pricing is displayed inline in the list using `CnCellRenderer` with EUR formatting.

The basket flow uses a transient Pinia store (not an OpenRegister object) to collect selected items and quantities before converting to a `PurchaseRequisition` or `PurchaseOrder` on checkout.

### Decision 2: Punch-out Catalog Integration (cXML/OCI)

`CatalogPunchoutService` (lib/Service/CatalogPunchoutService.php) handles the cXML PunchOut flow:
1. `generateSetupRequest(string $catalogId, string $returnUrl): string` — creates a cXML PunchOutSetupRequest with DUNS identity from the `Administration` config
2. `receiveOrderMessage(string $cxmlPayload): array` — parses PunchOutOrderMessage XML and returns a normalized array of basket items
3. Basket items are converted to `CatalogItem` objects (transient, not persisted) for display and requisition creation

Punch-out URL stored in `ProcurementCatalog.punchoutUrl`. OCI fallback supported via form POST to `ProcurementCatalog.ociUrl`.

### Decision 3: Purchase Requisition Workflow

`PurchaseRequisition` status transitions: `concept` → `ingediend` → `in behandeling` → `goedgekeurd` / `afgewezen`.

On submission, `RequisitionApprovalService::submit(string $reqId): ApprovalRequest`:
1. Loads the `PurchaseRequisition` and its linked `CostCenter`
2. Resolves the applicable `ApprovalChain` based on cost center and total amount thresholds
3. Creates an `ApprovalRequest` linked to the requisition via OpenRegister relation
4. Creates `ApprovalTask` objects for each step in the chain
5. Dispatches `NotificationService` notifications to approvers
6. Transitions requisition status to `ingediend`

Customizable intake forms: `ProcurementCategory.formTemplate` stores a JSON Schema object. `CnAdvancedFormDialog` renders custom fields from this schema at runtime — no custom form components needed. Custom field values stored in `PurchaseRequisition.customFields` (JSON).

### Decision 4: Discount Engine — PricingRule Evaluation

`PricingRuleService::applyRules(array $orderLines, array $couponCodes, string $customerId): array`:
1. Loads all active `PricingRule` objects (status = `actief`, validFrom ≤ today ≤ validUntil)
2. Sorts by `priority` ascending (lower number = evaluated first)
3. Evaluates each rule's `conditions` array against each order line and context
4. In `first-match` mode: stops after first matching rule per order line
5. In `all-match` mode: applies ALL matching rules additively per order line
6. Returns order lines annotated with `appliedDiscounts[]` array showing which rules fired

`PricingRule.conditions` is a JSON array of condition objects: `{field, operator, value}`. Supported fields: `category`, `sku`, `catalog`, `quantity`, `totalAmount`, `couponCode`, `customerType`, `orderCount`. Operators: `eq`, `neq`, `gte`, `lte`, `in`, `contains`.

### Decision 5: 2-way and 3-way PO Matching

`PurchaseOrderMatchingService::match(string $poId, string $mode): MatchResult`:
- **2-way**: compares `PurchaseOrder` line items against linked `Invoice` line items — quantity and amount within configured tolerance
- **3-way**: additionally compares against linked `GoodsReceipt` line items
- Tolerance rules read from `Administration.poMatchingConfig` (JSON): `priceVariancePct`, `quantityVarianceUnits`
- `MatchResult` (value object): `status` (matched/partial/discrepancy), `discrepancies[]` array with field, expected, actual values
- Match result stored as metadata on the `PurchaseOrder` object via `ObjectService.saveObject()`
- On discrepancy, `NotificationService` notifies the assigned buyer

### Decision 6: Quotation Lifecycle and Revision

`Quote` status: `concept` → `verzonden` → `herzien` → `geaccepteerd` / `afgewezen` / `verlopen`.

`QuoteRevisionService::createRevision(string $quoteId): Quote`:
1. Loads original `Quote` object
2. Creates a new `Quote` object with all fields copied, `revisionNumber` incremented
3. Sets `previousVersion` OpenRegister relation pointing to original quote
4. Transitions original quote status to `herzien`
5. Returns new revision as the active quote

PDF generation: calls `docudesk` document generation service with quote template `shillinq-quote-v1`. The template renders quote header, customer details, line items table, totals with discounts, validity date, and terms.

### Decision 7: Sale Order Line Sequence and Stock Sourcing

`PurchaseOrder` line items include a `sequence` integer. `SaleOrderSequenceService::reorder(string $orderId, array $lineIdSequenceMap): void` updates sequence values via `ObjectService.saveObject()` for each line.

Stock sourcing: `SaleStockSourcingService::resolveSource(string $catalogItemId, int $quantity): SourcingResult` — looks up preferred supplier/warehouse by matching `CatalogItem.sku` against sourcing rules stored in app config (`IAppConfig`). Returns supplier contact and estimated lead time.

## Data Models (reference only — entities defined in ADR-000)

Entities referenced in this change (do NOT redefine):

| Entity | Role |
|--------|------|
| `CatalogItem` | Catalog product with enriched attributes, pricing, stock status |
| `ProcurementCatalog` | Catalog container per supplier with punchout URL |
| `PurchaseRequisition` | Internal purchase request with custom fields and approval state |
| `PurchaseOrder` | Purchase order with line items, matching result, sequence |
| `PurchaseOrderRevision` | Revision of a PO after supplier negotiation |
| `Quote` | Sales or procurement quotation with revision chain |
| `PricingRule` | Discount/pricing rule with conditions array and execution mode |
| `GoodsReceipt` | Receipt of goods against PO for 3-way matching |
| `Invoice` | Invoice for 2-way/3-way matching |
| `ApprovalChain` | Approval workflow definition per cost center / amount tier |
| `ApprovalRequest` | Instance of an approval process linked to a requisition |
| `ApprovalTask` | Individual step in an approval chain |
| `CostCenter` | Cost allocation unit assigned to orders and requisitions |
| `RequestForQuotation` | RFQ sent to suppliers |
| `ProcurementQuote` | Quote received from supplier in response to RFQ |
| `ProcurementCategory` | Product/service category with customizable form template |

## Seed Data (ADR-001 — 3-5 realistic Dutch objects per key entity)

### ProcurementCatalog

```json
[
  {
    "@self": { "register": "shillinq", "schema": "ProcurementCatalog", "slug": "kantoorartikelen-standaard" },
    "name": "Kantoorartikelen Catalogus 2026",
    "supplier": "Staples Nederland BV",
    "catalogVersion": "2026-Q1",
    "status": "actief",
    "punchoutUrl": "https://punch-out.staples.nl/cxml",
    "ociUrl": null,
    "description": "Standaardcatalogus kantoorbenodigdheden voor gemeenten en MKB"
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementCatalog", "slug": "ict-hardware-raamcontract" },
    "name": "ICT Hardware Raamcontract 2026",
    "supplier": "Dustin BV",
    "catalogVersion": "2026-Q1",
    "status": "actief",
    "punchoutUrl": null,
    "ociUrl": null,
    "description": "ICT-apparatuur conform raamovereenkomst, leverbaar binnen 5 werkdagen"
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementCatalog", "slug": "facilitaire-diensten-mkb" },
    "name": "Facilitaire Diensten & Schoonmaak",
    "supplier": "ISS Facility Services BV",
    "catalogVersion": "2025-H2",
    "status": "actief",
    "punchoutUrl": null,
    "ociUrl": null,
    "description": "Facilitaire diensten raamcontract voor MKB organisaties"
  }
]
```

### CatalogItem

```json
[
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "kopieerpapier-a4-80g" },
    "name": "Kopieerpapier A4 80g/m² Wit",
    "sku": "KP-A4-80-WIT",
    "catalog": "kantoorartikelen-standaard",
    "unitPrice": 4.95,
    "currency": "EUR",
    "unit": "doos (500 vel)",
    "category": "Papier & Afdrukmedia",
    "classificationCode": "44101500",
    "classificationScheme": "UNSPSC",
    "stockStatus": "beschikbaar",
    "minimumOrderQuantity": 5,
    "leadTimeDays": 1
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "laptop-dell-latitude-5540" },
    "name": "Dell Latitude 5540 i5-1345U 16GB 512GB SSD",
    "sku": "DELL-LAT-5540-16-512",
    "catalog": "ict-hardware-raamcontract",
    "unitPrice": 1249.00,
    "currency": "EUR",
    "unit": "stuk",
    "category": "Notebooks",
    "classificationCode": "43211503",
    "classificationScheme": "UNSPSC",
    "stockStatus": "op aanvraag",
    "minimumOrderQuantity": 1,
    "leadTimeDays": 5
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "bureaustoel-ergonomisch-npr1813" },
    "name": "Ergonomische Bureaustoel NPR 1813 gecertificeerd",
    "sku": "BS-ERGO-NPR1813",
    "catalog": "kantoorartikelen-standaard",
    "unitPrice": 349.00,
    "currency": "EUR",
    "unit": "stuk",
    "category": "Kantoormeubilair",
    "classificationCode": "56101004",
    "classificationScheme": "UNSPSC",
    "stockStatus": "beschikbaar",
    "minimumOrderQuantity": 1,
    "leadTimeDays": 3
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "tonercartridge-hp-cf258a" },
    "name": "HP LaserJet Tonercartridge CF258A (zwart)",
    "sku": "HP-CF258A",
    "catalog": "kantoorartikelen-standaard",
    "unitPrice": 39.50,
    "currency": "EUR",
    "unit": "stuk",
    "category": "Toner & Inkt",
    "classificationCode": "44103105",
    "classificationScheme": "UNSPSC",
    "stockStatus": "beschikbaar",
    "minimumOrderQuantity": 1,
    "leadTimeDays": 1
  }
]
```

### PurchaseRequisition

```json
[
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "ink-2026-042" },
    "title": "Inkoopaanvraag kantoorbenodigdheden Q2 2026",
    "requester": "jan.de.vries@gemeente-amsterdam.nl",
    "department": "Facilitair Beheer",
    "costCenter": "GEM-FAC-001",
    "totalAmount": 2450.00,
    "currency": "EUR",
    "status": "ingediend",
    "urgency": "normaal",
    "justification": "Reguliere kwartaalinkoop kantoorartikelen afdeling Facilitair",
    "requestDate": "2026-04-15",
    "customFields": {}
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "ink-2026-043" },
    "title": "Aanvraag ICT-apparatuur nieuwe medewerkers mei 2026",
    "requester": "m.bakker@conduction.nl",
    "department": "ICT",
    "costCenter": "COND-ICT-002",
    "totalAmount": 8750.00,
    "currency": "EUR",
    "status": "in behandeling",
    "urgency": "hoog",
    "justification": "5 nieuwe medewerkers per 1 mei 2026 — laptops en accessoires vereist voor onboarding",
    "requestDate": "2026-04-20",
    "customFields": { "projectCode": "PROJ-2026-088", "leverancierVoorkeur": "Dustin BV" }
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "ink-2026-044" },
    "title": "Aanvraag ergonomisch meubilair thuiswerkplekken",
    "requester": "s.janssen@adviesbureau-fictief.nl",
    "department": "HR & Faciliteiten",
    "costCenter": "ADV-HR-003",
    "totalAmount": 5225.00,
    "currency": "EUR",
    "status": "goedgekeurd",
    "urgency": "normaal",
    "justification": "Arboconform inrichten van 15 thuiswerkplekken conform cao-afspraken 2026",
    "requestDate": "2026-04-18",
    "customFields": { "arboAdvies": "ja", "medewerkersaantal": 15 }
  }
]
```

### PricingRule

```json
[
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "volumekorting-papier-10plus" },
    "name": "Volumekorting Papier (min. 10 dozen)",
    "conditions": [
      { "field": "category", "operator": "eq", "value": "Papier & Afdrukmedia" },
      { "field": "quantity", "operator": "gte", "value": 10 }
    ],
    "discountType": "percentage",
    "discountValue": 8.0,
    "executionMode": "all-match",
    "priority": 10,
    "status": "actief",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31"
  },
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "welkomstkorting-nieuwe-klant" },
    "name": "Welkomstkorting Eerste Bestelling (WELKOM2026)",
    "conditions": [
      { "field": "couponCode", "operator": "eq", "value": "WELKOM2026" },
      { "field": "orderCount", "operator": "eq", "value": 0 }
    ],
    "discountType": "percentage",
    "discountValue": 10.0,
    "executionMode": "first-match",
    "priority": 1,
    "status": "actief",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31"
  },
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "overheidskorting-raamcontract-2026" },
    "name": "Overheidskorting Raamcontract 2026 (5%)",
    "conditions": [
      { "field": "customerType", "operator": "eq", "value": "overheid" },
      { "field": "catalog", "operator": "eq", "value": "kantoorartikelen-standaard" }
    ],
    "discountType": "percentage",
    "discountValue": 5.0,
    "executionMode": "all-match",
    "priority": 20,
    "status": "actief",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31"
  },
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "per-item-korting-toner" },
    "name": "Toner Per-Item Korting (€ 3,00 per stuk)",
    "conditions": [
      { "field": "category", "operator": "eq", "value": "Toner & Inkt" },
      { "field": "quantity", "operator": "gte", "value": 5 }
    ],
    "discountType": "fixed",
    "discountValue": 3.00,
    "executionMode": "all-match",
    "priority": 15,
    "status": "actief",
    "validFrom": "2026-01-01",
    "validUntil": "2026-06-30"
  }
]
```

### Quote

```json
[
  {
    "@self": { "register": "shillinq", "schema": "Quote", "slug": "offerte-2026-0312" },
    "quoteNumber": "OFF-2026-0312",
    "title": "Offerte ICT-infrastructuur Fase 2",
    "customer": "Gemeente Utrecht",
    "issueDate": "2026-04-10",
    "expiryDate": "2026-05-10",
    "totalAmount": 45800.00,
    "currency": "EUR",
    "status": "verzonden",
    "revisionNumber": 1,
    "previousVersion": null
  },
  {
    "@self": { "register": "shillinq", "schema": "Quote", "slug": "offerte-2026-0313-r2" },
    "quoteNumber": "OFF-2026-0313-R2",
    "title": "Offerte Schoonmaakdiensten Jaarcontract (Rev. 2)",
    "customer": "Conduction BV",
    "issueDate": "2026-04-20",
    "expiryDate": "2026-05-20",
    "totalAmount": 17800.00,
    "currency": "EUR",
    "status": "geaccepteerd",
    "revisionNumber": 2,
    "previousVersion": "offerte-2026-0313"
  },
  {
    "@self": { "register": "shillinq", "schema": "Quote", "slug": "offerte-2026-0314" },
    "quoteNumber": "OFF-2026-0314",
    "title": "Offerte Ergonomisch Meubilair Bulk (15 stuks)",
    "customer": "Adviesbureau Fictief NL",
    "issueDate": "2026-04-15",
    "expiryDate": "2026-05-15",
    "totalAmount": 12350.00,
    "currency": "EUR",
    "status": "concept",
    "revisionNumber": 1,
    "previousVersion": null
  }
]
```

## Reuse Analysis (ADR-012)

| Capability | Platform Service / Component Used |
|---|---|
| Catalog list & detail views | `CnIndexPage` + `CnDetailPage` + `createObjectStore('catalog-item')` |
| Faceted catalog search | `CnFacetSidebar` + `FacetBuilder` + `IndexService` |
| Requisition intake forms | `CnAdvancedFormDialog` rendering `ProcurementCategory.formTemplate` JSON Schema |
| Approval routing | `WorkflowEngineController` + `TasksController` (NotificationService for notifications) |
| PO matching notifications | `NotificationService` |
| Quote PDF generation | `docudesk` PDF service with Shillinq quote template |
| Audit trail on all entities | `AuditTrailService` (automatic via OpenRegister) |
| File attachments on requisitions | `FileService` + `CnObjectSidebar` → `CnFilesTab` |
| Import/export | `ImportService` / `ExportService` + `CnMassImportDialog` / `CnMassExportDialog` |
| Dashboard KPIs | `CnDashboardPage` + `CnStatsBlock` + `CnChartWidget` |

**Deduplication check result:** No overlap found with `ObjectService`, `RegisterService`, `SchemaService`, `ConfigurationService`, or `@conduction/nextcloud-vue` components for the custom business logic — specifically: `PricingRuleService` (discount evaluation), `PurchaseOrderMatchingService` (PO matching), `CatalogPunchoutService` (cXML integration), `RequisitionApprovalService` (approval routing), and `QuoteRevisionService` (quote versioning) are all domain-specific and have no platform equivalent.
