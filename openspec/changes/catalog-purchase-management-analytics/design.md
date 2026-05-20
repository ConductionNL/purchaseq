# Design: Catalog & Purchase Management — Shillinq — Analytics

## Context

Shillinq stores procurement data (purchase orders, spend transactions, catalog items, supplier
KPIs, goods receipts) as OpenRegister objects. This change layers analytics, dashboards, and
delivery operations **on top of** those existing objects — no new entities are introduced.
The platform provides `CnDashboardPage`, `CnChartWidget`, `CnStatsBlock`, and `CnKpiGrid`
for free; custom code is limited to aggregation endpoints, a PDF generation service, and a
scheduled statistics job.

## Goals / Non-Goals

**Goals:**
- Real-time spend analytics dashboard with drill-down into categories and suppliers
- Estimated-vs-actual spend variance computation and display
- SOW-linked PO creation with milestone progress tracking
- Delivery note PDF generation from goods receipt/proof-of-delivery objects
- Catalog item usage statistics and inventory level widgets
- CO2/sustainability spend tracking linked to procurement categories
- Configurable KPI scorecards and executive dashboards
- Annual procurement statistics background job

**Non-Goals:**
- New entity/schema definitions — all entities already exist in OpenRegister
- Custom chart library — use `CnChartWidget` (ApexCharts) exclusively
- Custom file upload — use `FileService` + `CnObjectSidebar`
- Custom audit logging — use `AuditTrailService` automatically
- Competitor benchmarking data ingestion (out of scope for this change)

## Reuse Analysis (ADR-012)

| Capability | OpenRegister / Platform Service Used |
|---|---|
| Dashboard layout + widgets | `CnDashboardPage`, `CnChartWidget`, `CnStatsBlock`, `CnKpiGrid` |
| Spend object listing + filtering | `ObjectService.findAll()` + `CnDataTable` |
| CSV/Excel export of analytics data | `ExportService` + `CnMassExportDialog` |
| Audit trail on all analytics views | `AuditTrailService` (automatic) |
| PDF generation (delivery notes) | docudesk integration via `FileService` |
| Full-text + faceted search | `IndexService` + `CnFacetSidebar` |
| KPI object CRUD | `createObjectStore` + `CnFormDialog` |
| Background job scheduling | `IJobList` + `QueuedJob` |
| Role-based access control | `AuthorizationService` + `PropertyRbacHandler` |
| Notifications | `NotificationService` |

No overlap found with `ObjectService`, `RegisterService`, `SchemaService`, or
`@conduction/nextcloud-vue` built-in components that would require duplication.
Custom code is limited to: spend aggregation logic, CO2 computation, delivery note
PDF templating, and annual statistics rollup — none of these exist in OpenRegister core.

## Architecture Decisions

### Decision 1: Aggregation endpoint for estimated-vs-actual spend

Raw `SpendTransaction` and `ProcurementCategory` objects do not carry pre-aggregated totals.
A lightweight `GET /api/analytics/spend-variance` endpoint in `AnalyticsController` queries
OpenRegister via `ObjectService`, groups by category, and returns `{ category, estimated,
actual, variance }` arrays. No custom DB query — uses OpenRegister's `findAll()` with
`_filters` on date range and category.

### Decision 2: Delivery note PDF via docudesk template

`DeliveryNoteService` accepts a `GoodsReceipt` object ID, resolves the linked `PurchaseOrder`,
`Supplier`, and `ProofOfDelivery` via OpenRegister relations, and renders a Twig template
registered with docudesk. The generated PDF is attached to the `GoodsReceipt` object via
`FileService`. No custom file controller.

### Decision 3: SOW → PO milestone linking via OpenRegister relations

`StatementOfWork` objects already carry a `milestones` relation to `ContractMilestone`. The
dashboard widget reads the linked `PurchaseOrder` relation and displays milestone status using
`CnTimelineStages`. No new fields or schemas needed.

### Decision 4: Annual statistics via scheduled background job

`AnnualStatisticsJob` runs as a `QueuedJob` on 1 January each year. It aggregates
`SpendTransaction` totals, supplier counts, PO volumes, and category breakdowns for the
prior fiscal year, and writes a summary `Report` object to OpenRegister. Dashboard widgets
read from this `Report` object for fast display.

### Decision 5: CO2 tracking via ProcurementCategory annotation

`ProcurementCategory` objects include a `co2EmissionFactor` field (kg CO2 per EUR spend).
`SpendAggregationService` multiplies actual spend by this factor to compute CO2 totals per
category. The sustainability widget on the dashboard reads from `GET /api/analytics/co2`.

## Data Model (Referenced Entities — ADR-001, ADR-011)

All entities are defined in OpenRegister. This change references only:

| Entity | Register | Usage |
|---|---|---|
| `CatalogItem` | shillinq | Catalog usage statistics, inventory tracking |
| `ProcurementCatalog` | shillinq | Parent catalog grouping for items |
| `ProcurementCategory` | shillinq | Spend classification, CO2 factor |
| `PurchaseOrder` | shillinq | PO creation, delivery tracking, bill matching |
| `StatementOfWork` | shillinq | SOW-based PO creation, milestone tracking |
| `ContractMilestone` | shillinq | Milestone progress on SOW/PO |
| `GoodsReceipt` | shillinq | Delivery note generation, delivery tracking |
| `ProofOfDelivery` | shillinq | Delivery confirmation |
| `VendorBill` | shillinq | Bill matching against PO |
| `SpendTransaction` | shillinq | Real-time expense data, analytics |
| `ExpenseReport` | shillinq | Expense aggregation |
| `InventoryItem` | shillinq | Inventory tracking widget |
| `SupplierKPI` | shillinq | KPI scorecard |
| `SupplierPerformanceScore` | shillinq | KPI dashboard metrics |
| `SupplierPerformanceReport` | shillinq | Executive dashboard |
| `ProcurementOrder` | shillinq | Order confirmation tracking |
| `Report` | shillinq | Annual statistics output |

## Seed Data (ADR-001)

Seed data uses the `@self` envelope. All values use Dutch organizational context.

### CatalogItem (5 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "kantoorbenodigdheden-a4-papier"
    },
    "name": "A4 Kopieerpapier 80gr (500 vel)",
    "description": "Wit kopieerpapier geschikt voor alle laserprinters en kopieerapparaten.",
    "sku": "KAN-001",
    "unitPrice": 4.95,
    "currency": "EUR",
    "category": "Kantoorbenodigdheden",
    "unit": "ream",
    "active": true
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "ict-laptop-standaard"
    },
    "name": "Laptop zakelijk — standaard configuratie",
    "description": "14-inch zakelijke laptop, 16 GB RAM, 512 GB SSD, Windows 11 Pro.",
    "sku": "ICT-045",
    "unitPrice": 1249.00,
    "currency": "EUR",
    "category": "ICT-hardware",
    "unit": "stuks",
    "active": true
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "schoonmaak-allesreiniger"
    },
    "name": "Allesreiniger concentraat 5L",
    "description": "Professionele allesreiniger voor kantoor en gemeentelijke gebouwen.",
    "sku": "SCH-012",
    "unitPrice": 18.50,
    "currency": "EUR",
    "category": "Facilitair",
    "unit": "fles",
    "active": true
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "vergaderstoel-ergonomisch"
    },
    "name": "Ergonomische vergaderstoel NEN-EN 1335",
    "description": "In hoogte verstelbare vergaderstoel met armleggers, NEN-gecertificeerd.",
    "sku": "MEU-089",
    "unitPrice": 349.00,
    "currency": "EUR",
    "category": "Meubilair",
    "unit": "stuks",
    "active": true
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "CatalogItem",
      "slug": "catering-koffie-bonen"
    },
    "name": "Koffiebonen biologisch 1kg",
    "description": "Fairtrade biologische koffiebonen, geschikt voor volautomaten.",
    "sku": "CAT-003",
    "unitPrice": 12.75,
    "currency": "EUR",
    "category": "Catering",
    "unit": "zak",
    "active": true
  }
]
```

### ProcurementCategory (4 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCategory",
      "slug": "ict-hardware"
    },
    "name": "ICT-hardware",
    "code": "ICT-HW",
    "description": "Computers, laptops, servers en randapparatuur.",
    "co2EmissionFactor": 0.35,
    "sustainabilityLabel": "Hoog",
    "budgetCode": "44010"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCategory",
      "slug": "kantoorbenodigdheden"
    },
    "name": "Kantoorbenodigdheden",
    "code": "KAN",
    "description": "Papier, pennen, ordners en overige kantoorartikelen.",
    "co2EmissionFactor": 0.08,
    "sustainabilityLabel": "Laag",
    "budgetCode": "44020"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCategory",
      "slug": "facilitair"
    },
    "name": "Facilitaire diensten",
    "code": "FAC",
    "description": "Schoonmaak, beveiliging en onderhoud gebouwen.",
    "co2EmissionFactor": 0.12,
    "sustainabilityLabel": "Gemiddeld",
    "budgetCode": "44030"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ProcurementCategory",
      "slug": "ict-diensten"
    },
    "name": "ICT-diensten",
    "code": "ICT-SVC",
    "description": "Softwarelicenties, SaaS-abonnementen en IT-consultancy.",
    "co2EmissionFactor": 0.05,
    "sustainabilityLabel": "Laag",
    "budgetCode": "44040"
  }
]
```

### SpendTransaction (5 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "SpendTransaction",
      "slug": "spend-2026-ict-laptops-q1"
    },
    "description": "Aanschaf 20 laptops — Q1 2026",
    "amount": 24980.00,
    "currency": "EUR",
    "transactionDate": "2026-01-15",
    "category": "ICT-hardware",
    "supplier": "Dustin Nederland BV",
    "purchaseOrderRef": "PO-2026-0042",
    "status": "Goedgekeurd"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SpendTransaction",
      "slug": "spend-2026-schoonmaak-jan"
    },
    "description": "Schoonmaakdiensten januari 2026",
    "amount": 3850.00,
    "currency": "EUR",
    "transactionDate": "2026-01-31",
    "category": "Facilitaire diensten",
    "supplier": "CSU Schoonmaak BV",
    "purchaseOrderRef": "PO-2026-0018",
    "status": "Betaald"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SpendTransaction",
      "slug": "spend-2026-catering-q1"
    },
    "description": "Cateringbenodigdheden Q1 2026",
    "amount": 1240.50,
    "currency": "EUR",
    "transactionDate": "2026-02-03",
    "category": "Catering",
    "supplier": "Sligro Food Group NV",
    "purchaseOrderRef": "PO-2026-0031",
    "status": "Goedgekeurd"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SpendTransaction",
      "slug": "spend-2026-saas-microsoft"
    },
    "description": "Microsoft 365 licenties — jaarabonnement 2026",
    "amount": 18600.00,
    "currency": "EUR",
    "transactionDate": "2026-01-01",
    "category": "ICT-diensten",
    "supplier": "Microsoft Nederland BV",
    "purchaseOrderRef": "PO-2026-0005",
    "status": "Betaald"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SpendTransaction",
      "slug": "spend-2026-kantoor-papier-q1"
    },
    "description": "Kantoorpapier en verbruiksartikelen Q1 2026",
    "amount": 892.00,
    "currency": "EUR",
    "transactionDate": "2026-02-14",
    "category": "Kantoorbenodigdheden",
    "supplier": "Staples Nederland BV",
    "purchaseOrderRef": "PO-2026-0055",
    "status": "Goedgekeurd"
  }
]
```

### StatementOfWork (3 objects)

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "StatementOfWork",
      "slug": "sow-digitalisering-zaakgericht-2026"
    },
    "title": "Digitalisering zaakgericht werken — fase 2",
    "description": "Implementatie van zaaksysteem koppeling en documentbeheer voor de afdeling Burgerzaken.",
    "supplier": "Conduction BV",
    "startDate": "2026-02-01",
    "endDate": "2026-07-31",
    "totalValue": 85000.00,
    "currency": "EUR",
    "status": "Actief"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "StatementOfWork",
      "slug": "sow-beveiliging-audit-2026"
    },
    "title": "Informatiebeveiliging audit en advies 2026",
    "description": "Onafhankelijke audit van de informatiebeveiliging conform BIO 2.0.",
    "supplier": "Fox-IT BV",
    "startDate": "2026-03-01",
    "endDate": "2026-05-31",
    "totalValue": 32500.00,
    "currency": "EUR",
    "status": "Concept"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "StatementOfWork",
      "slug": "sow-training-privacy-2026"
    },
    "title": "AVG-bewustwordingstraining medewerkers 2026",
    "description": "E-learning en klassikale training voor 250 medewerkers over gegevensbescherming.",
    "supplier": "Hoffmann BV",
    "startDate": "2026-04-01",
    "endDate": "2026-06-30",
    "totalValue": 14750.00,
    "currency": "EUR",
    "status": "Concept"
  }
]
```

## Frontend Architecture

```
src/
  views/
    Dashboard.vue           ← CnDashboardPage with 6 widget slots
    SpendAnalytics.vue      ← CnIndexPage reading SpendTransaction objects
    CatalogAnalytics.vue    ← CnIndexPage reading CatalogItem + InventoryItem
    ProcurementKpi.vue      ← CnDetailPage for SupplierPerformanceReport
  components/
    widgets/
      SpendVarianceWidget.vue       ← CnChartWidget (bar) — estimated vs actual
      SustainabilityWidget.vue      ← CnChartWidget (donut) — CO2 by category
      CatalogUsageWidget.vue        ← CnTableWidget — top catalog items
      KpiScorecardWidget.vue        ← CnKpiGrid — SupplierKPI values
      MilestoneProgressWidget.vue   ← CnTimelineStages — SOW milestones
      AnnualStatsWidget.vue         ← CnStatsBlock — annual totals
  store/
    modules/
      spendTransactions.js    ← createObjectStore('SpendTransaction')
      catalogItems.js         ← createObjectStore('CatalogItem')
      procurementCategories.js← createObjectStore('ProcurementCategory')
      statementOfWorks.js     ← createObjectStore('StatementOfWork')
      supplierKpis.js         ← createObjectStore('SupplierKPI')
      inventoryItems.js       ← createObjectStore('InventoryItem')
```

## Backend Architecture

```
lib/
  Controller/
    AnalyticsController.php       ← GET /api/analytics/spend-variance
                                     GET /api/analytics/co2
                                     GET /api/analytics/catalog-usage
                                     GET /api/analytics/annual-stats
  Service/
    SpendAggregationService.php   ← estimated-vs-actual + CO2 computation
    DeliveryNoteService.php       ← PDF generation from GoodsReceipt via docudesk
  BackgroundJob/
    AnnualStatisticsJob.php       ← QueuedJob — rolls up annual spend on 1 Jan
```

All controllers are thin (<10 lines/method). Business logic lives exclusively in services.
DI via constructor injection with `private readonly`. `@spec` tags on every class and method.
