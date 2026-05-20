# Design: Catalog & Purchase Management — Shillinq — Other T2

## Overview

This document describes the technical design for the Shillinq catalog and purchase
management module. All data is stored as OpenRegister objects. No custom
Entity/Mapper classes are introduced — domain data exclusively uses `ObjectService`.

---

## Reuse Analysis (ADR-012)

| Platform capability | How it is reused |
|---------------------|-----------------|
| `ObjectService.saveObject()` / `findObjects()` | All CRUD for every entity in this change |
| `CnIndexPage` + `useListView` | Catalog browser, requisition list, PO list |
| `CnDetailPage` + `CnDetailCard` | PO detail, requisition detail, catalog item detail |
| `CnFormDialog` | Schema-driven create/edit for all entities |
| `CnFormDialog` / `CnAdvancedFormDialog` | Quote creation with expiry date field |
| `ApprovalChain` / `ApprovalRequest` / `ApprovalTask` | Requisition approval routing (existing entities) |
| `CnDashboardPage` + `CnStatsBlock` + `CnChartWidget` | Procurement KPI dashboard |
| `WorkflowEngineController` | Procurement automation rules (no-code autopilot) |
| `NotificationService` | Quote expiry reminders, approval notifications |
| `FileService` + `CnObjectSidebar` → `CnFilesTab` | Supporting documents attached to requisitions |
| `AuditTrailService` | Automatic change tracking for all procurement objects |
| `AuthorizationService` + `PropertyRbacHandler` | Role-based access for procurement teams |
| `CnMassExportDialog` | Export procurement reports (CSV/JSON) |
| `CnTimelineStages` | Requisition → PO lifecycle progression view |
| `IndexService` + `CnFacetSidebar` | Catalog search and faceted filtering |
| `createObjectStore` + store plugins | Pinia stores for all procurement entities |

**Finding:** No custom logic needed for CRUD, search, file attachment, audit trails, or
dashboard widgets — all provided by OpenRegister + @conduction/nextcloud-vue.
Custom code required only for: PO PDF generation, quote expiry background job,
DigiInkoop API integration, procurement automation rule engine wrapper.

---

## Architecture

### Layer diagram

```
┌─────────────────────────────────────────────────────┐
│  Frontend (Vue 2 + Pinia)                           │
│  CnIndexPage / CnDetailPage / CnDashboardPage       │
│  Stores: createObjectStore per entity type          │
├─────────────────────────────────────────────────────┤
│  Controllers (thin, <10 lines/method)               │
│  ProcurementCatalogController                       │
│  PurchaseRequisitionController                      │
│  PurchaseOrderController                            │
│  ProcurementQuoteController                         │
│  ProcurementAnalyticsController                     │
├─────────────────────────────────────────────────────┤
│  Services (business logic)                          │
│  ProcurementCatalogService  — catalog management    │
│  PurchaseRequisitionService — requisition + approval│
│  PurchaseOrderService       — PO lifecycle + PDF    │
│  ProcurementQuoteService    — quote + expiry        │
│  ProcurementAutomationService — rule engine bridge  │
│  ProcurementAnalyticsService  — KPIs, savings       │
├─────────────────────────────────────────────────────┤
│  OpenRegister (ObjectService)                       │
│  Register: shillinq-procurement                     │
│  Schemas: see below                                 │
└─────────────────────────────────────────────────────┘
```

### Register definition

**Register slug:** `shillinq-procurement`
**File:** `lib/Settings/shillinq_register.json` (OpenAPI 3.0 + x-openregister)

---

## Entity Schemas

All entities are **existing** app-level entities referenced from the master entity
list. Schema definitions below describe the fields relevant to this change.
Implementations add these schemas to the shillinq-procurement register.

### ProcurementCatalog

Purpose: An internal product/service catalog made available to requesters. Supports
hosted buyer catalogs with managed content and custom internal catalogs.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | yes | Catalog name (e.g. "Kantoorartikelen 2026") |
| `description` | string | no | Catalog description |
| `status` | string (enum) | yes | `draft` \| `active` \| `archived` |
| `owner` | relation → Organization | yes | Owning organizational unit |
| `categories` | relation[] → ProcurementCategory | no | Top-level categories |
| `supplier` | relation → Supplier | no | Supplier for hosted buyer catalogs |
| `validFrom` | date | no | Catalog validity start |
| `validUntil` | date | no | Catalog validity end |
| `isHostedBuyer` | boolean | no | True if managed by supplier |
| `pricingRules` | relation[] → PricingRule | no | Volume/tiered pricing rules |

### CatalogItem

Purpose: A product or service available in a ProcurementCatalog. Supports 300+
product classification categories (CPV codes).

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | yes | Item name |
| `description` | string | no | Item description |
| `sku` | string | no | Stock-keeping unit code |
| `cpvCode` | string | no | CPV product classification code |
| `catalog` | relation → ProcurementCatalog | yes | Parent catalog |
| `category` | relation → ProcurementCategory | no | Product category |
| `unitPrice` | number | yes | Unit price (excl. VAT) |
| `currency` | string | yes | ISO 4217 currency code (e.g. "EUR") |
| `unit` | string | no | Unit of measure (stuk, uur, kg, etc.) |
| `vatRate` | number | no | VAT percentage (0, 9, 21) |
| `minOrderQuantity` | integer | no | Minimum order quantity |
| `leadTimeDays` | integer | no | Typical delivery lead time |
| `isActive` | boolean | yes | Whether item is orderable |
| `imageUrl` | string | no | Product image URL |
| `pricingTiers` | array | no | Volume/tiered pricing breakpoints |
| `customFields` | object | no | Free-form custom metadata fields |

### ProcurementCategory

Purpose: Hierarchical classification of products and services for procurement.
Supports both direct (goods) and indirect (services) categories.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | yes | Category name |
| `code` | string | no | Category code (CPV prefix) |
| `type` | string (enum) | yes | `direct` \| `indirect` |
| `parent` | relation → ProcurementCategory | no | Parent category (hierarchical) |
| `description` | string | no | Category description |
| `approvalRequired` | boolean | no | Override approval threshold for this category |
| `budgetCode` | string | no | Default budget/cost-centre code |

### PurchaseRequisition

Purpose: Internal request to procure goods or services. Supports free-text and
catalog-based ordering, configurable custom form fields, and multi-entity requisitions.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `title` | string | yes | Requisition title |
| `requester` | relation → Person | yes | Person who raised the requisition |
| `entity` | relation → Organization | yes | Organizational unit making the request |
| `status` | string (enum) | yes | `draft` \| `submitted` \| `pending_approval` \| `approved` \| `rejected` \| `converted_to_po` \| `cancelled` |
| `category` | relation → ProcurementCategory | no | Procurement category |
| `lineItems` | array | yes | Items (catalog or free-text) |
| `totalAmount` | number | no | Estimated total amount excl. VAT |
| `currency` | string | yes | Currency code |
| `requiredByDate` | date | no | Date goods/services are required |
| `justification` | string | no | Business justification |
| `approvalChain` | relation → ApprovalChain | no | Assigned approval chain |
| `approvalRequest` | relation → ApprovalRequest | no | Active approval request |
| `purchaseOrder` | relation → PurchaseOrder | no | Resulting PO after approval |
| `attachments` | relation[] → DigitalDocument | no | Supporting documents |
| `customFields` | object | no | App-configured custom form fields |
| `autoApproved` | boolean | no | True if auto-approved below threshold |
| `isMultiEntity` | boolean | no | True if cross-entity requisition |

### PurchaseOrder

Purpose: Legally binding order issued to a supplier. Supports create, generate,
revise, sign, and track delivery status. Also generates from approved requisitions.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `orderNumber` | string | yes | Unique PO number (auto-generated) |
| `supplier` | relation → Supplier | yes | Supplier receiving the order |
| `requisition` | relation → PurchaseRequisition | no | Source requisition (if any) |
| `template` | relation → PurchaseOrder | no | Template used to create this PO |
| `status` | string (enum) | yes | `draft` \| `sent` \| `acknowledged` \| `partially_delivered` \| `delivered` \| `invoiced` \| `closed` \| `cancelled` |
| `lineItems` | array | yes | Order line items |
| `totalAmount` | number | yes | Total amount excl. VAT |
| `currency` | string | yes | Currency code |
| `deliveryAddress` | object | no | Delivery address |
| `requiredDeliveryDate` | date | no | Required delivery date |
| `paymentTerms` | string | no | Payment terms (e.g. "30 dagen netto") |
| `signedBy` | relation → Person | no | Digital signatory |
| `signedAt` | datetime | no | Timestamp of digital signature |
| `isTemplate` | boolean | no | True if this PO is a reusable template |
| `blanketOrder` | relation → BlanketPurchaseOrder | no | Parent blanket order (if call-off) |
| `revisions` | relation[] → PurchaseOrderRevision | no | Revision history |

### PurchaseOrderRevision

Purpose: Tracks changes made to a PurchaseOrder after initial issuance.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `purchaseOrder` | relation → PurchaseOrder | yes | Parent PO |
| `revisionNumber` | integer | yes | Sequential revision number |
| `revisedBy` | relation → Person | yes | Person who made the revision |
| `revisedAt` | datetime | yes | Timestamp of revision |
| `changes` | array | yes | List of changed fields with before/after values |
| `reason` | string | yes | Reason for revision |
| `newTotalAmount` | number | no | Updated total after revision |

### BlanketPurchaseOrder

Purpose: Framework order with a supplier covering a defined period, quantity limit,
and/or value limit. Generates call-off purchase orders on schedule.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `orderNumber` | string | yes | Unique blanket PO number |
| `supplier` | relation → Supplier | yes | Supplier |
| `status` | string (enum) | yes | `active` \| `expired` \| `exhausted` \| `cancelled` |
| `validFrom` | date | yes | Agreement start date |
| `validUntil` | date | yes | Agreement end date |
| `maxValue` | number | no | Maximum total value of call-offs |
| `maxQuantity` | number | no | Maximum total quantity of call-offs |
| `consumedValue` | number | no | Value of call-offs issued to date |
| `consumedQuantity` | number | no | Quantity of call-offs issued to date |
| `releaseSchedule` | string (enum) | no | `manual` \| `weekly` \| `monthly` \| `quarterly` |
| `nextReleaseDate` | date | no | Next scheduled call-off date |
| `callOffOrders` | relation[] → PurchaseOrder | no | Call-off orders issued |
| `lineItems` | array | yes | Items covered by the blanket order |

### ProcurementQuote

Purpose: A price quotation received from a supplier, with configurable expiry date
and one-click acceptance to generate a PurchaseOrder.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `quoteNumber` | string | yes | Unique quote reference |
| `supplier` | relation → Supplier | yes | Quoting supplier |
| `status` | string (enum) | yes | `draft` \| `received` \| `under_review` \| `accepted` \| `rejected` \| `expired` |
| `expiryDate` | date | yes | Quote expiry date |
| `expiryReminderDays` | integer | no | Days before expiry to send reminder (default: 7) |
| `lineItems` | array | yes | Quoted items with prices |
| `totalAmount` | number | yes | Total quoted amount excl. VAT |
| `currency` | string | yes | Currency code |
| `purchaseOrder` | relation → PurchaseOrder | no | PO generated on acceptance |
| `requisition` | relation → PurchaseRequisition | no | Source requisition |
| `notes` | string | no | Internal notes |

### SavingsOpportunity

Purpose: Tracks procurement savings against a baseline, enabling KPI reporting.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `title` | string | yes | Savings initiative title |
| `category` | relation → ProcurementCategory | no | Associated category |
| `baselineAmount` | number | yes | Baseline spend (before savings) |
| `targetAmount` | number | yes | Target spend after savings initiative |
| `actualAmount` | number | no | Actual spend achieved |
| `currency` | string | yes | Currency code |
| `savingsType` | string (enum) | no | `price_reduction` \| `volume_consolidation` \| `demand_reduction` \| `process_improvement` |
| `status` | string (enum) | yes | `identified` \| `in_progress` \| `realised` \| `cancelled` |
| `targetDate` | date | no | Target date for realisation |
| `achievedDate` | date | no | Date savings were confirmed |
| `supplier` | relation → Supplier | no | Related supplier |
| `evidence` | relation[] → DigitalDocument | no | Supporting evidence |

---

## Seed Data (Dutch values, 3–5 objects per key entity)

### ProcurementCatalog seed

```json
[
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCatalog", "slug": "kantoorartikelen-2026" },
    "name": "Kantoorartikelen 2026",
    "description": "Interne catalogus voor kantoorartikelen en verbruiksmateriaal",
    "status": "active",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31",
    "isHostedBuyer": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCatalog", "slug": "ict-hardware-catalogus" },
    "name": "ICT Hardware Catalogus",
    "description": "Laptops, monitoren, randapparatuur en netwerkmaterialen",
    "status": "active",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31",
    "isHostedBuyer": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCatalog", "slug": "facilitaire-diensten" },
    "name": "Facilitaire Diensten",
    "description": "Schoonmaak, catering en facilitaire ondersteuning",
    "status": "active",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31",
    "isHostedBuyer": true
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCatalog", "slug": "inhuur-zzp-2026" },
    "name": "Inhuur ZZP 2026",
    "description": "Raamovereenkomst voor inhuur van zelfstandig professionals",
    "status": "active",
    "validFrom": "2026-01-01",
    "validUntil": "2026-12-31",
    "isHostedBuyer": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCatalog", "slug": "drukwerk-en-media" },
    "name": "Drukwerk en Media",
    "description": "Brochures, folders, banners en digitale media-uitingen",
    "status": "draft",
    "validFrom": "2026-07-01",
    "validUntil": "2027-06-30",
    "isHostedBuyer": false
  }
]
```

### CatalogItem seed

```json
[
  {
    "@self": { "register": "shillinq-procurement", "schema": "CatalogItem", "slug": "laptop-dell-latitude-5540" },
    "name": "Dell Latitude 5540 Laptop",
    "description": "15,6 inch laptop, Intel Core i5, 16 GB RAM, 256 GB SSD",
    "sku": "DL-LAT-5540-i5",
    "cpvCode": "30213100-6",
    "unitPrice": 879.00,
    "currency": "EUR",
    "unit": "stuk",
    "vatRate": 21,
    "minOrderQuantity": 1,
    "leadTimeDays": 5,
    "isActive": true
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "CatalogItem", "slug": "a4-kopieerpapier-500vel" },
    "name": "A4 Kopieerpapier 80g/m² (500 vel)",
    "description": "Wit kopieerpapier, geschikt voor laser- en inkjetprinters",
    "sku": "PP-A4-80G-500",
    "cpvCode": "30197630-1",
    "unitPrice": 4.95,
    "currency": "EUR",
    "unit": "riem",
    "vatRate": 21,
    "minOrderQuantity": 5,
    "leadTimeDays": 2,
    "isActive": true
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "CatalogItem", "slug": "schoonmaakdienst-maandelijks" },
    "name": "Schoonmaakdienst (maandelijks, kantoorruimte)",
    "description": "Dagelijks schoonmaken van kantoorruimte tot 500 m²",
    "sku": "SVC-CLEAN-MAAND",
    "cpvCode": "90911200-8",
    "unitPrice": 1250.00,
    "currency": "EUR",
    "unit": "maand",
    "vatRate": 21,
    "minOrderQuantity": 1,
    "leadTimeDays": 14,
    "isActive": true
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "CatalogItem", "slug": "ergonomische-bureaustoel" },
    "name": "Ergonomische Bureaustoel Klasse A",
    "description": "NEN-EN 1335 gecertificeerde bureaustoel met verstelbare rugsteun",
    "sku": "FURN-CHAIR-ERG-A",
    "cpvCode": "39113100-8",
    "unitPrice": 349.00,
    "currency": "EUR",
    "unit": "stuk",
    "vatRate": 21,
    "minOrderQuantity": 1,
    "leadTimeDays": 10,
    "isActive": true
  }
]
```

### ProcurementCategory seed

```json
[
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCategory", "slug": "ict-hardware" },
    "name": "ICT Hardware",
    "code": "302",
    "type": "direct",
    "description": "Computers, randapparatuur en netwerkhardware",
    "approvalRequired": false,
    "budgetCode": "63000"
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCategory", "slug": "kantoorartikelen" },
    "name": "Kantoorartikelen",
    "code": "301",
    "type": "direct",
    "description": "Verbruiksmateriaal, papier, pennen en kantoorbenodigdheden",
    "approvalRequired": false,
    "budgetCode": "61000"
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCategory", "slug": "facilitaire-diensten" },
    "name": "Facilitaire Diensten",
    "code": "909",
    "type": "indirect",
    "description": "Schoonmaak, catering en beveiliging",
    "approvalRequired": false,
    "budgetCode": "64000"
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCategory", "slug": "inhuur-personeel" },
    "name": "Inhuur Personeel",
    "code": "791",
    "type": "indirect",
    "description": "Tijdelijke inhuur van medewerkers en ZZP-professionals",
    "approvalRequired": true,
    "budgetCode": "62000"
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "ProcurementCategory", "slug": "meubilair" },
    "name": "Meubilair en Inventaris",
    "code": "391",
    "type": "direct",
    "description": "Bureaumeubilair, kasten en inrichtingsartikelen",
    "approvalRequired": false,
    "budgetCode": "61500"
  }
]
```

### PurchaseRequisition seed

```json
[
  {
    "@self": { "register": "shillinq-procurement", "schema": "PurchaseRequisition", "slug": "pr-2026-00142" },
    "title": "Laptops voor nieuwe medewerkers Q2 2026",
    "status": "pending_approval",
    "totalAmount": 4395.00,
    "currency": "EUR",
    "requiredByDate": "2026-06-15",
    "justification": "5 nieuwe medewerkers starten 1 juni; laptops zijn arbeidsvoorwaarde",
    "autoApproved": false,
    "isMultiEntity": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "PurchaseRequisition", "slug": "pr-2026-00143" },
    "title": "Kantoorartikelen aanvulling mei 2026",
    "status": "auto_approved",
    "totalAmount": 87.50,
    "currency": "EUR",
    "requiredByDate": "2026-05-28",
    "justification": "Reguliere aanvulling van verbruiksmateriaal",
    "autoApproved": true,
    "isMultiEntity": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "PurchaseRequisition", "slug": "pr-2026-00144" },
    "title": "Inhuur senior Java-developer (3 maanden)",
    "status": "submitted",
    "totalAmount": 27000.00,
    "currency": "EUR",
    "requiredByDate": "2026-06-01",
    "justification": "Versterking ontwikkelteam voor migratie naar nieuwe infrastructuur",
    "autoApproved": false,
    "isMultiEntity": false
  }
]
```

### PurchaseOrder seed

```json
[
  {
    "@self": { "register": "shillinq-procurement", "schema": "PurchaseOrder", "slug": "po-2026-00301" },
    "orderNumber": "PO-2026-00301",
    "status": "sent",
    "totalAmount": 4395.00,
    "currency": "EUR",
    "requiredDeliveryDate": "2026-06-15",
    "paymentTerms": "30 dagen netto",
    "isTemplate": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "PurchaseOrder", "slug": "po-2026-00302" },
    "orderNumber": "PO-2026-00302",
    "status": "delivered",
    "totalAmount": 87.50,
    "currency": "EUR",
    "requiredDeliveryDate": "2026-05-28",
    "paymentTerms": "14 dagen netto",
    "isTemplate": false
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "PurchaseOrder", "slug": "po-template-laptops" },
    "orderNumber": "TPL-LAPTOPS-001",
    "status": "draft",
    "totalAmount": 0,
    "currency": "EUR",
    "paymentTerms": "30 dagen netto",
    "isTemplate": true
  }
]
```

### SavingsOpportunity seed

```json
[
  {
    "@self": { "register": "shillinq-procurement", "schema": "SavingsOpportunity", "slug": "besparing-kantoorartikelen-2026" },
    "title": "Consolidatie kantoorartikelen via raamovereenkomst",
    "baselineAmount": 48000.00,
    "targetAmount": 36000.00,
    "actualAmount": 37200.00,
    "currency": "EUR",
    "savingsType": "volume_consolidation",
    "status": "realised",
    "targetDate": "2026-03-31",
    "achievedDate": "2026-04-15"
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "SavingsOpportunity", "slug": "heronderhandeling-schoonmaak" },
    "title": "Heronderhandeling schoonmaakcontract",
    "baselineAmount": 18000.00,
    "targetAmount": 15000.00,
    "currency": "EUR",
    "savingsType": "price_reduction",
    "status": "in_progress",
    "targetDate": "2026-07-01"
  },
  {
    "@self": { "register": "shillinq-procurement", "schema": "SavingsOpportunity", "slug": "reduceren-inhuur-externe-ict" },
    "title": "Reduceren externe ICT-inhuur via kennisoverdracht",
    "baselineAmount": 120000.00,
    "targetAmount": 90000.00,
    "currency": "EUR",
    "savingsType": "demand_reduction",
    "status": "identified",
    "targetDate": "2026-12-31"
  }
]
```

---

## Frontend Pages & Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/procurement` | `ProcurementDashboard.vue` | KPI dashboard with stats and charts |
| `/procurement/catalogs` | `CnIndexPage` (ProcurementCatalog) | Catalog list |
| `/procurement/catalogs/:id` | `CnDetailPage` (ProcurementCatalog) | Catalog detail + items |
| `/procurement/catalog-items` | `CnIndexPage` (CatalogItem) | All catalog items, faceted search |
| `/procurement/categories` | `CnIndexPage` (ProcurementCategory) | Category hierarchy |
| `/procurement/requisitions` | `CnIndexPage` (PurchaseRequisition) | Requisition list |
| `/procurement/requisitions/:id` | `CnDetailPage` (PurchaseRequisition) | Requisition detail + approval status |
| `/procurement/purchase-orders` | `CnIndexPage` (PurchaseOrder) | PO list |
| `/procurement/purchase-orders/:id` | `CnDetailPage` (PurchaseOrder) | PO detail + revisions + delivery |
| `/procurement/blanket-orders` | `CnIndexPage` (BlanketPurchaseOrder) | Blanket PO list |
| `/procurement/blanket-orders/:id` | `CnDetailPage` (BlanketPurchaseOrder) | Blanket PO detail + call-offs |
| `/procurement/quotes` | `CnIndexPage` (ProcurementQuote) | Quote list with expiry indicators |
| `/procurement/quotes/:id` | `CnDetailPage` (ProcurementQuote) | Quote detail + accept/reject |
| `/procurement/savings` | `CnIndexPage` (SavingsOpportunity) | Savings tracker |
| `/settings` | `ProcurementSettings.vue` | Approval thresholds, automation rules |

---

## Background Jobs

| Job | Trigger | Purpose |
|-----|---------|---------|
| `QuoteExpiryReminderJob` | Daily cron | Sends notifications for quotes expiring within `expiryReminderDays` |
| `BlanketOrderReleaseJob` | Daily cron | Generates call-off POs on `nextReleaseDate` |
| `ProcurementKpiJob` | Weekly cron | Refreshes procurement KPI aggregates |

---

## API Endpoints (custom, beyond OpenRegister CRUD)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/procurement/requisitions/{id}/submit` | Submit a draft requisition for approval |
| `POST` | `/api/procurement/requisitions/{id}/approve` | Approve a pending requisition |
| `POST` | `/api/procurement/requisitions/{id}/reject` | Reject a pending requisition with reason |
| `POST` | `/api/procurement/requisitions/{id}/convert-to-po` | Convert approved requisition to PO |
| `POST` | `/api/procurement/quotes/{id}/accept` | Accept a quote, generate PO |
| `POST` | `/api/procurement/quotes/{id}/reject` | Reject a quote with reason |
| `POST` | `/api/procurement/purchase-orders/{id}/sign` | Digitally sign a PO |
| `POST` | `/api/procurement/purchase-orders/{id}/revise` | Create a PO revision |
| `POST` | `/api/procurement/blanket-orders/{id}/release` | Manually release a call-off PO |
| `GET`  | `/api/procurement/analytics/kpis` | Retrieve KPI aggregates |
| `GET`  | `/api/metrics` | Prometheus metrics (admin) |
| `GET`  | `/api/health` | Health check (public) |

---

## Security Considerations (ADR-005)

- All mutation endpoints enforce `IGroupManager::isAdmin()` or role-based
  `AuthorizationService` checks on the backend.
- Approval authority is validated server-side against the assigned `ApprovalChain`.
- Auto-approval threshold is a server-side configurable setting — not client-controlled.
- PO PDF generation does not expose internal paths or stack traces.
- Spending card virtual card numbers are never stored in logs or error responses.
