# Design: Catalog & Purchase Management — Shillinq

## Context

Shillinq is a complete open-source business administration suite on Nextcloud. The
catalog and purchase management capability covers the source-to-pay (S2P) lifecycle:
from browsing an internal product catalog through raising a purchase requisition, routing
it for approval, converting it to a purchase order, receiving goods, and matching the
vendor bill — all within Nextcloud.

The entities involved (`CatalogItem`, `ProcurementCatalog`, `PurchaseRequisition`,
`PurchaseOrder`, `BlanketPurchaseOrder`, `GoodsReceipt`, `VendorBill`, `PricingRule`,
`RateCard`, `ApprovalChain`, `ApprovalRoute`, `CallOffOrder`, `SpendTransaction`,
`SpendCategory`, `ProcurementCategory`, `ProcurementOrder`, `Location`,
`ProcurementProcedure`) are all defined in the Shillinq data model. This change
implements the UI, API, and domain services to operate these entities; it does NOT
introduce new schemas.

## Goals

- Full source-to-pay workflow: catalog → requisition → PO → receipt → bill matching
- Multi-entity procurement supporting shared service center operations
- Multi-location delivery: `PurchaseOrder` lines reference `Location` entities
- Blanket PO management with call-off release scheduling and consumption monitoring
- EU procurement threshold enforcement and procedure type selection
- Spend-by-category dashboard for CFO/controller visibility
- All CRUD, import/export, search, and audit trail via OpenRegister platform services

## Non-Goals

- Custom invoicing or payment processing (handled by Shillinq's Accounts Payable module)
- External marketplace or e-procurement portal integration (future change)
- TenderNed / DigiInkoop API integration (separate spec)
- Supplier portal / supplier-facing UI (separate spec)
- Custom document generation for POs (future — uses Shillinq's existing PDF generation)

## Decisions

### Decision 1: Three-way matching via `ThreeWayMatchService`

Three-way matching (PO quantity/price ↔ GoodsReceipt quantity ↔ VendorBill amount) is
domain-specific business logic with no equivalent in OpenRegister's platform services.
`ThreeWayMatchService` computes match status (`matched`, `partial`, `disputed`) and stores
it as a field on `PurchaseOrder`. The service is stateless; it is called:
- When a `GoodsReceipt` is saved against a `PurchaseOrder`
- When a `VendorBill` is linked to a `PurchaseOrder`
- On-demand via `POST /api/purchase-orders/{id}/match`

Match result is surfaced as a `CnStatusBadge` on the PO index and detail pages.

### Decision 2: Approval routing via `ApprovalRoutingService`

`ApprovalChain` and `ApprovalRoute` are existing Shillinq entities. `ApprovalRoutingService`
evaluates which chain applies to a `PurchaseRequisition` based on:
1. `ProcurementCategory` of the requisition
2. Total value of the requisition
3. Organisation entity (for multi-entity shared service centers)

It creates the appropriate `ApprovalRequest` records and dispatches Nextcloud notifications
via `NotificationService`. No custom workflow engine is built — `WorkflowEngineController`
from OpenRegister handles state transitions.

### Decision 3: Multi-entity procurement support

The `Organisation` field on `PurchaseRequisition` and `PurchaseOrder` enables shared service
center operations. A procurement officer belonging to entity A may raise a requisition on
behalf of entity B. `ApprovalRoutingService` resolves the approval chain per the target
entity. Multi-tenancy isolation is enforced at the API/service layer per ADR-005; UI
shows a cross-entity scope selector when the user has multi-entity permissions.

### Decision 4: EU procurement threshold table via `ProcurementThresholdService`

EU procurement thresholds (works, supplies, services — central/sub-central/utilities)
are stored as `ProcurementProcedure` seed objects. `ProcurementThresholdService` compares
a requisition or PO value against current thresholds and returns the applicable procedure
type (`openbaar`, `niet-openbaar`, `meervoudig onderhands`, `enkelvoudig onderhands`).
Thresholds are updated in the seed data on each EU revision cycle — no external API call
required at runtime.

### Decision 5: Spend analysis via aggregated `SpendTransaction` records

When a `VendorBill` is matched and approved, a `SpendTransaction` record is created by
`ProcurementController`, referencing the `SpendCategory` derived from the
`ProcurementCategory` on the originating requisition. The `SpendAnalysisDashboard.vue`
page queries `SpendTransaction` objects grouped by `SpendCategory` using
`ObjectService.findAll()` with filter parameters — no custom analytics endpoint.

### Decision 6: Blanket PO consumption tracking

`BlanketPurchaseOrder` has `maxAmount`, `releasedAmount`, and `consumedAmount` fields.
When a `CallOffOrder` is created against a blanket PO, `ProcurementController` increments
`releasedAmount`. When the linked `VendorBill` is matched, `consumedAmount` is updated.
Consumption is visualised as `CnProgressBar` on the blanket PO detail page.

## Data Model

All entities are pre-defined in the Shillinq data model. Key relations used by this change:

```
PurchaseRequisition
  ├── lines[]        → PurchaseRequisitionLine (embedded array)
  ├── approvalChain  → ApprovalChain (OpenRegister relation)
  ├── category       → ProcurementCategory
  ├── organisation   → Organisation
  └── costAllocation → CostAllocation

PurchaseOrder
  ├── requisition    → PurchaseRequisition
  ├── supplier       → Supplier
  ├── lines[]        → PurchaseOrderLine (embedded)
  ├── deliveryLocation → Location
  ├── goodsReceipt   → GoodsReceipt
  ├── vendorBill     → VendorBill
  └── matchStatus    string: matched | partial | disputed | pending

BlanketPurchaseOrder
  ├── supplier       → Supplier
  ├── callOffOrders[]→ CallOffOrder
  ├── maxAmount      → MonetaryAmount
  ├── releasedAmount → MonetaryAmount
  └── consumedAmount → MonetaryAmount

CatalogItem
  ├── catalog        → ProcurementCatalog
  ├── supplier       → Supplier
  ├── pricingRule    → PricingRule
  ├── category       → ProcurementCategory
  └── components[]  → CatalogItem (for kits/bundles)

SpendTransaction
  ├── category       → SpendCategory
  ├── vendorBill     → VendorBill
  └── purchaseOrder  → PurchaseOrder
```

## Seed Data

Seed objects use the `@self` envelope per ADR-001. All values are realistic Dutch data
for a fictitious municipality (Gemeente Westerveld) and a consultancy (Conduction BV).

### ProcurementCatalog (3 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCatalog",
      "slug": "catalog-ict-apparatuur-2025"
    },
    "name": "ICT-apparatuur & Software 2025",
    "description": "Goedgekeurde raamovereenkomst voor ICT-hardware en softwarelicenties. Geldig tot 31 december 2025.",
    "status": "active",
    "validFrom": "2025-01-01",
    "validUntil": "2025-12-31",
    "organisation": "gemeente-westerveld"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCatalog",
      "slug": "catalog-kantoorinrichting-2025"
    },
    "name": "Kantoorinrichting & -benodigdheden 2025",
    "description": "Standaard kantoormeubilair en benodigdheden via raamovereenkomst Kinnarps en Staples.",
    "status": "active",
    "validFrom": "2025-01-01",
    "validUntil": "2025-12-31",
    "organisation": "gemeente-westerveld"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCatalog",
      "slug": "catalog-facilitaire-diensten-2025"
    },
    "name": "Facilitaire diensten & schoonmaak 2025",
    "description": "Facilitaire dienstverlening inclusief schoonmaak, catering en beveiliging.",
    "status": "active",
    "validFrom": "2025-01-01",
    "validUntil": "2025-12-31",
    "organisation": "gemeente-westerveld"
  }
]
```

### CatalogItem (5 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "catalogitem-canon-iradv-c5540i"
    },
    "name": "Canon iR-ADV C5540i multifunctionele printer",
    "description": "A3/A4 kleurenprinter, kopiëren, scannen, printen. 40 ppm kleur. Inclusief 3 jaar on-site service.",
    "sku": "CANON-IRADV-C5540I",
    "unitPrice": 8950.00,
    "currency": "EUR",
    "unit": "stuk",
    "category": "ict-apparatuur",
    "catalog": "catalog-ict-apparatuur-2025",
    "supplier": "supplier-canon-nederland",
    "status": "active",
    "leadTimeDays": 10
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "catalogitem-microsoft-365-bp"
    },
    "name": "Microsoft 365 Business Premium — jaarlicentie per gebruiker",
    "description": "Volledige Microsoft 365 suite inclusief Teams, Exchange, SharePoint en Defender. Jaarcontract, per gebruiker per maand gefactureerd.",
    "sku": "MS-365-BP-NL-1Y",
    "unitPrice": 22.60,
    "currency": "EUR",
    "unit": "gebruiker/maand",
    "category": "software-licenties",
    "catalog": "catalog-ict-apparatuur-2025",
    "supplier": "supplier-microsoft-nederland",
    "status": "active",
    "leadTimeDays": 1
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "catalogitem-hag-capisco-stoel"
    },
    "name": "HAG Capisco ergonomische bureaustoel",
    "description": "Ergonomische zadelstoel, in hoogte verstelbaar 47–95 cm. NEN-EN 1335 gecertificeerd. Kleur: zwart/zilver.",
    "sku": "KINN-HAG-CAP-BLK",
    "unitPrice": 1290.00,
    "currency": "EUR",
    "unit": "stuk",
    "category": "kantoormeubilair",
    "catalog": "catalog-kantoorinrichting-2025",
    "supplier": "supplier-kinnarps-nl",
    "status": "active",
    "leadTimeDays": 14
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "catalogitem-printpapier-a4-80g"
    },
    "name": "Navigator Universal printpapier A4 80g/m², doos 5 pakken",
    "description": "Wit printpapier geschikt voor alle laser- en inkjetprinters. 500 vel per pak, 2500 vel per doos.",
    "sku": "STAP-NAV-A4-80-5PK",
    "unitPrice": 34.95,
    "currency": "EUR",
    "unit": "doos",
    "category": "kantoorbenodigdheden",
    "catalog": "catalog-kantoorinrichting-2025",
    "supplier": "supplier-staples-nl",
    "status": "active",
    "leadTimeDays": 2
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "catalogitem-schoonmaak-uurprijs"
    },
    "name": "Kantoorschoonmaak — uurprijs",
    "description": "Reguliere kantoorschoonmaak op werkdagen. Tariefstelling per uur inclusief materiaal en middelen.",
    "sku": "ISS-CLEAN-HOUR-NL",
    "unitPrice": 38.50,
    "currency": "EUR",
    "unit": "uur",
    "category": "facilitaire-diensten",
    "catalog": "catalog-facilitaire-diensten-2025",
    "supplier": "supplier-iss-nederland",
    "status": "active",
    "leadTimeDays": 5
  }
]
```

### PurchaseRequisition (3 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "PurchaseRequisition",
      "slug": "pr-2026-001-bureaustoelen-hr"
    },
    "title": "10 ergonomische bureaustoelen afdeling HR",
    "description": "Vervanging van verouderd kantoormeubilair voor de afdeling HRM, locatie Gemeentehuis Westerveld.",
    "requisitionNumber": "PR-2026-001",
    "status": "approved",
    "totalAmount": 12900.00,
    "currency": "EUR",
    "category": "kantoormeubilair",
    "organisation": "gemeente-westerveld",
    "requestedBy": "j.bakker@westerveld.nl",
    "requestedDate": "2026-03-10",
    "requiredByDate": "2026-04-01"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "PurchaseRequisition",
      "slug": "pr-2026-002-microsoft-365"
    },
    "title": "Microsoft 365 Business Premium — 50 gebruikers",
    "description": "Uitbreiding Microsoft 365 licenties voor nieuwe medewerkers en contractverlengingen FY2026.",
    "requisitionNumber": "PR-2026-002",
    "status": "pending_approval",
    "totalAmount": 13560.00,
    "currency": "EUR",
    "category": "software-licenties",
    "organisation": "gemeente-westerveld",
    "requestedBy": "t.hendriks@westerveld.nl",
    "requestedDate": "2026-04-15",
    "requiredByDate": "2026-05-01"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "PurchaseRequisition",
      "slug": "pr-2026-003-schoonmaak-q3"
    },
    "title": "Schoonmaakdiensten uitbreiding Q3 2026",
    "description": "Extra schoonmaakuren voor gemeentearchief tijdens verbouwing. Schatting 120 uur verspreid over Q3.",
    "requisitionNumber": "PR-2026-003",
    "status": "draft",
    "totalAmount": 4620.00,
    "currency": "EUR",
    "category": "facilitaire-diensten",
    "organisation": "gemeente-westerveld",
    "requestedBy": "a.devries@westerveld.nl",
    "requestedDate": "2026-05-20",
    "requiredByDate": "2026-07-01"
  }
]
```

### PurchaseOrder (3 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "PurchaseOrder",
      "slug": "po-2026-001-kinnarps-stoelen"
    },
    "orderNumber": "PO-2026-001",
    "title": "10x HAG Capisco ergonomische bureaustoel",
    "supplier": "supplier-kinnarps-nl",
    "requisition": "pr-2026-001-bureaustoelen-hr",
    "status": "sent",
    "totalAmount": 12900.00,
    "currency": "EUR",
    "orderDate": "2026-03-18",
    "expectedDeliveryDate": "2026-04-01",
    "deliveryLocation": "locatie-gemeentehuis-westerveld",
    "matchStatus": "pending"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "PurchaseOrder",
      "slug": "po-2026-002-microsoft-365"
    },
    "orderNumber": "PO-2026-002",
    "title": "Microsoft 365 Business Premium 50 gebruikers — jaarcontract 2026",
    "supplier": "supplier-microsoft-nederland",
    "requisition": "pr-2026-002-microsoft-365",
    "status": "confirmed",
    "totalAmount": 13560.00,
    "currency": "EUR",
    "orderDate": "2026-04-22",
    "expectedDeliveryDate": "2026-05-01",
    "matchStatus": "matched"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "PurchaseOrder",
      "slug": "po-2026-003-iss-schoonmaak"
    },
    "orderNumber": "PO-2026-003",
    "title": "Kantoorschoonmaak Q3 2026 — ISS Nederland",
    "supplier": "supplier-iss-nederland",
    "status": "draft",
    "totalAmount": 4620.00,
    "currency": "EUR",
    "orderDate": "2026-05-20",
    "matchStatus": "pending"
  }
]
```

### BlanketPurchaseOrder (2 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "BlanketPurchaseOrder",
      "slug": "bpo-2026-001-staples-kantoor"
    },
    "orderNumber": "BPO-2026-001",
    "title": "Raambestelling kantoorbenodigdheden — Staples 2026",
    "supplier": "supplier-staples-nl",
    "status": "active",
    "maxAmount": 50000.00,
    "releasedAmount": 12340.00,
    "consumedAmount": 9875.00,
    "currency": "EUR",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "BlanketPurchaseOrder",
      "slug": "bpo-2026-002-canon-onderhoud"
    },
    "orderNumber": "BPO-2026-002",
    "title": "Onderhoud en reparatie Canon printers 2026",
    "supplier": "supplier-canon-nederland",
    "status": "active",
    "maxAmount": 25000.00,
    "releasedAmount": 8750.00,
    "consumedAmount": 6200.00,
    "currency": "EUR",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31"
  }
]
```

### PricingRule (3 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "PricingRule",
      "slug": "pricingrule-volumekorting-ict"
    },
    "name": "Volumekorting ICT-apparatuur boven €10.000",
    "description": "5% korting op alle ICT-hardware wanneer het orderbedrag de €10.000 overschrijdt.",
    "discountType": "percentage",
    "discountValue": 5.0,
    "minimumOrderAmount": 10000.00,
    "currency": "EUR",
    "category": "ict-apparatuur",
    "status": "active"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "PricingRule",
      "slug": "pricingrule-raamovereenkomst-schoonmaak"
    },
    "name": "Raamovereenkomst schoonmaak Gemeente Westerveld",
    "description": "Vaste uurprijs schoonmaakdiensten conform raamovereenkomst 2025–2026.",
    "discountType": "fixed_price",
    "fixedUnitPrice": 38.50,
    "currency": "EUR",
    "category": "facilitaire-diensten",
    "supplier": "supplier-iss-nederland",
    "status": "active"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "PricingRule",
      "slug": "pricingrule-early-payment"
    },
    "name": "Vroegbetalingskorting leveranciers (2/10 netto 30)",
    "description": "2% korting bij betaling binnen 10 dagen, anders netto betaalbaar binnen 30 dagen.",
    "discountType": "percentage",
    "discountValue": 2.0,
    "paymentTermDays": 10,
    "status": "active"
  }
]
```

## Reuse Analysis

Per ADR-012, the following OpenRegister platform services are leveraged — no rebuild:

| Capability | OpenRegister / @conduction/nextcloud-vue service | Notes |
|---|---|---|
| CRUD for all entities | `ObjectService.saveObject()` / `deleteObject()` | Standard OpenRegister |
| List with pagination, sort, filter | `ObjectService.findAll()` + `CnDataTable` | Used on all index pages |
| Schema-driven create/edit forms | `CnFormDialog` | Auto-generated from schema |
| Bulk catalog import | `CnMassImportDialog` + `ImportService` | No custom parser |
| Export catalog items | `CnMassExportDialog` + `ExportService` | CSV/JSON/Excel |
| Faceted catalog search | `CnFacetSidebar` + `FacetBuilder` | Category/supplier facets |
| Full-text search | `IndexService` + `CnFilterBar` | Catalog item search |
| Approval workflow state | `WorkflowEngineController` | State transitions |
| Notifications on approval events | `NotificationService` | Requestor + approver |
| Audit trail on all entities | `AuditTrailService` (automatic) | CnObjectSidebar tab |
| File attachments on POs | `FileService` + `CnObjectSidebar` | Quote docs, delivery notes |
| Dashboard KPI widgets | `CnDashboardPage` + `CnStatsBlock` + `CnChartWidget` | Spend analysis |
| Object stores | `createObjectStore` with plugins | Per entity |
| Multi-tenancy isolation | `TenantLifecycleService` (automatic) | Per ADR-005 |
| Authorization / RBAC | `AuthorizationService` + `PropertyRbacHandler` | Role-based access |
| Timeline stages | `CnTimelineStages` | Requisition approval progress |

**Custom logic built by this change (no overlap found):**

- `ThreeWayMatchService`: PO/GoodsReceipt/VendorBill matching — no equivalent in OpenRegister
- `ApprovalRoutingService`: Category/amount/entity-based chain selection — domain-specific
- `ProcurementThresholdService`: EU procurement threshold comparison — domain-specific

No overlap with `ObjectService`, `RegisterService`, `SchemaService`, or shared Vue components found.
