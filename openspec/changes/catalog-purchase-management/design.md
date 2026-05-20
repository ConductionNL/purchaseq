# Design: Catalog & Purchase Management — Shillinq

## Architecture overview

This change introduces seven new OpenRegister schemas for Shillinq's procurement catalog and purchase management domain. All domain data is stored as OpenRegister objects. No custom Entity/Mapper classes are introduced — CRUD, search, import/export, audit trails, and file attachments are provided by the platform.

Custom business logic is limited to:
1. **Contract expiry notification jobs** — background jobs that detect catalog items linked to contracts expiring within 60 days
2. **Maverick spend detection** — background job cross-referencing POs without contract references against active contracts by supplier and category
3. **CPV classification service** — suggesting CPV codes for unclassified spend lines
4. **DigiInkoop integration** — outbound electronic ordering adapter
5. **OCI punchout integration** — inbound supplier catalog session handler
6. **Spend cube aggregation** — GraphQL-driven spend breakdown by CPV/supplier/org-unit

## Reuse analysis

| Platform capability | Used by this change | Implementation |
|--------------------|---------------------|---------------|
| `ObjectService.saveObject()` | All schema CRUD | Platform — no custom code |
| `ObjectService.findObjects()` | Catalog search, requisition list | Platform — no custom code |
| `ImportService` + `CnMassImportDialog` | Bulk catalog import (CSV/cXML/CIF) | Platform — configure schema mappings |
| `ExportService` + `CnMassExportDialog` | Catalog export, spend reports | Platform — no custom code |
| `AuditTrailService` + `CnAuditTrailTab` | Procurement action audit trail | Platform — supplement with `ProcurementAuditLog` schema for business-level audit records |
| `NotificationService` | Contract expiry alerts, maverick spend alerts | Platform — dispatch from background job |
| `ActivityService` | Shared procurement workspace feed | Platform — no custom code |
| `CnDashboardPage` + `CnChartWidget` + `CnStatsBlock` | Spend dashboard, contract budget dashboard | Platform — configure widgets |
| `CnIndexPage` + `useListView` | All list views (catalogs, items, requisitions, etc.) | Platform — no custom list components |
| `CnDetailPage` + `CnDetailCard` | All detail views | Platform — no custom detail components |
| `CnFormDialog` | Create/edit forms for all schemas | Platform — schema-driven auto-generation |
| `AuthorizationService` + `PropertyRbacHandler` | User-specific catalog views (feature 7) | Platform — configure roles; no custom RBAC code |
| `GraphQLController` | Spend cube aggregation across schemas | Platform — no custom aggregation service |
| `WorkflowEngineController` | Requisition approval routing | Platform — no custom workflow engine |
| `ScheduledWorkflowController` | Contract expiry check, maverick spend monthly alert | Platform scheduler — custom job payload only |
| `VectorizationService` | CPV code suggestion (semantic similarity) | Platform — custom prompt wrapper only |
| `WebhookService` | DigiInkoop order dispatch event | Platform — register schema/action subscription |

**No overlap found** with existing Shillinq specs or OpenRegister core for the following custom items:
- DigiInkoop adapter (Dutch government-specific; not in OpenRegister core)
- OCI punchout handler (procurement-specific session protocol; not in platform)
- CPV classification prompt wrapper (domain-specific vocabulary; extends VectorizationService)
- Spend cube dimension configuration (domain-specific CPV/supplier/org-unit axes)

## Entity schemas

### ProcurementCatalog (`schema:Catalog`)

Master catalog of products and services available for organizational procurement. Supports internal, cXML, and CIF format catalogs.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `catalogNumber` | string | ✓ | Unique catalog identifier (e.g. `CAT-2025-001`) |
| `catalogName` | string | ✓ | Display name of the catalog |
| `description` | string | | Catalog description and scope |
| `catalogFormat` | string | | Format: `internal`, `cxml`, `cif` |
| `status` | string | ✓ | Lifecycle: `draft`, `active`, `archived` |
| `validFrom` | datetime | | Catalog effective start date |
| `validUntil` | datetime | | Catalog expiration date |

**Relations:**
- `organization` → Organization (many-to-one) — owning department or entity
- `catalogItems` → CatalogItem (one-to-many) — reverse relation via CatalogItem

---

### CatalogItem (`schema:Product`)

Individual product or service line in a procurement catalog with pricing, availability, lead time, and contract linkage.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `itemCode` | string | ✓ | Unique item code within catalog (e.g. `IT-LAPTOP-001`) |
| `itemName` | string | ✓ | Display name of the item |
| `description` | string | | Detailed item description |
| `basePrice` | number | ✓ | Base unit price (excl. VAT) |
| `unit` | string | ✓ | Pricing unit: `piece`, `kg`, `liter`, `hour`, `license`, etc. |
| `minimumQuantity` | number | | Minimum order quantity |
| `leadTime` | number | | Delivery lead time in calendar days |
| `status` | string | ✓ | `active`, `discontinued` |
| `validFrom` | datetime | | Item availability start |
| `validUntil` | datetime | | Item availability end (linked to contract expiry) |

**Relations:**
- `procurementCatalog` → ProcurementCatalog (many-to-one)
- `product` → Product (many-to-one) — links to master product record
- `pricingRules` → PricingRule (one-to-many) — reverse relation via PricingRule

---

### PricingRule (`schema:PriceSpecification`)

Volume discounts, tiered pricing, bundle discounts, and promotional pricing rules with validity periods and application priorities.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `ruleCode` | string | ✓ | Unique pricing rule identifier |
| `description` | string | | Rule description and application conditions |
| `ruleType` | string | ✓ | `volumeDiscount`, `tierPricing`, `bundleDiscount`, `periodDiscount` |
| `minQuantity` | number | | Minimum quantity for rule activation |
| `maxQuantity` | number | | Maximum quantity covered by rule |
| `discountPercentage` | number | | Percentage discount (0–100) |
| `discountAmount` | number | | Fixed discount amount in base currency |
| `priority` | number | | Priority order for overlapping rule resolution (lower = higher priority) |
| `validFrom` | datetime | | Rule effective start |
| `validUntil` | datetime | | Rule expiration |

**Relations:**
- `catalogItem` → CatalogItem (many-to-one)

---

### PurchaseRequisition (`schema:Order`)

Formal request for goods or services with line items, business justification, delivery requirements, and approval workflow.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `requisitionNumber` | string | ✓ | Unique requisition identifier (auto-generated) |
| `requisitionDate` | datetime | ✓ | Date requisition was created |
| `status` | string | ✓ | `draft`, `submitted`, `approved`, `rejected`, `ordered` |
| `purpose` | string | | Business justification for the purchase |
| `deliveryDate` | datetime | | Requested delivery date |
| `customFields` | object | | Custom fields for procurement-specific data (cost centre, project code, etc.) |
| `totalAmount` | number | | Estimated total value (excl. VAT) |

**Relations:**
- `requester` → Person (many-to-one) — submitting user
- `organization` → Organization (many-to-one) — requesting department
- `approvalRequests` → ApprovalRequest (one-to-many) — approval workflow steps

---

### BlanketPurchaseOrder (`schema:Order`)

Master purchase order with authorized spend limit, scheduled release management, and consumption tracking for blanket purchasing arrangements.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `blanketPoNumber` | string | ✓ | Unique blanket PO identifier |
| `validFrom` | datetime | ✓ | Blanket PO effective start date |
| `validUntil` | datetime | ✓ | Blanket PO expiration date |
| `totalAuthorizedAmount` | number | ✓ | Total authorized spend limit (excl. VAT) |
| `consumedAmount` | number | | Amount spent against blanket PO to date |
| `remainingAmount` | number | | Remaining authorized spend (computed) |
| `releaseSchedule` | array | | Scheduled release objects: `{ releaseDate, amount, description }` |
| `status` | string | ✓ | `active`, `closed`, `cancelled` |

**Relations:**
- `organization` → Organization (many-to-one)
- `procurementCatalog` → ProcurementCatalog (many-to-one)
- `purchaseOrders` → PurchaseOrder (one-to-many)
- `approvalRequest` → ApprovalRequest (many-to-one)

---

### StatementOfWork (`schema:CreativeWork`)

Detailed specification of deliverables, milestones, payment terms, and service scope for statement-of-work-based procurement.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `sowNumber` | string | ✓ | Unique SOW identifier |
| `sowDate` | datetime | ✓ | Date SOW was created |
| `title` | string | ✓ | SOW title |
| `description` | string | | Detailed description of the work |
| `scope` | string | | Work scope and exclusions |
| `deliverables` | array | | Deliverable objects: `{ title, description, dueDate, acceptanceCriteria }` |
| `milestones` | array | | Milestone objects: `{ title, completionDate, invoiceAmount, invoiceTrigger }` |
| `totalValue` | number | ✓ | Total SOW value (excl. VAT) |
| `currency` | string | ✓ | ISO 4217 currency code (default: `EUR`) |
| `status` | string | ✓ | `draft`, `active`, `completed`, `cancelled` |

**Relations:**
- `organization` → Organization (many-to-one) — client organization
- `supplier` → Person (many-to-one) — supplier contact
- `contract` → Contract (many-to-one) — parent framework contract
- `purchaseOrders` → PurchaseOrder (one-to-many) — milestone-triggered orders

---

### ProcurementAuditLog (`schema:Action`)

Immutable audit trail recording all procurement actions, approvals, rejections, and changes for transparency, compliance, and decision accountability. Supplements OpenRegister's built-in audit trail with business-level procurement event semantics.

| Property | Type | Required | Description |
|----------|------|:--------:|-------------|
| `auditId` | string | ✓ | Unique audit log entry identifier |
| `entityType` | string | ✓ | `requisition`, `purchaseOrder`, `invoice`, `payment`, `approval`, `catalogItem` |
| `entityId` | string | ✓ | ID of the entity being audited |
| `actionType` | string | ✓ | `created`, `updated`, `approved`, `rejected`, `posted`, `received`, `deactivated` |
| `timestamp` | datetime | ✓ | When the action occurred (server time, immutable) |
| `reason` | string | | Reason or comment for the action |
| `changes` | object | | Changed fields with `{ field, oldValue, newValue }` objects |
| `referenceDocuments` | array | | Related document identifiers (e.g. contract number, PO number) |

**Relations:**
- `actor` → Person (many-to-one) — user who performed the action
- `organization` → Organization (many-to-one) — organizational context

---

## UI pages

| Page | Route | Component pattern |
|------|-------|-------------------|
| Procurement Dashboard | `/` | `CnDashboardPage` with KPI blocks (open requisitions, pending approvals, active catalogs, maverick spend %) and spend trend chart |
| Catalog list | `/catalogs` | `CnIndexPage` + `useListView` |
| Catalog detail | `/catalogs/:id` | `CnDetailPage` + `CnDetailCard` (catalog items list, linked POs) + `CnObjectSidebar` |
| Catalog item list | `/catalog-items` | `CnIndexPage` + `useListView` with filter by catalog, status, price range |
| Catalog item detail | `/catalog-items/:id` | `CnDetailPage` + `CnDetailCard` (pricing rules) + `CnObjectSidebar` |
| Purchase requisition list | `/requisitions` | `CnIndexPage` + `useListView` with filter by status, requester, department |
| Purchase requisition detail | `/requisitions/:id` | `CnDetailPage` + `CnTimelineStages` (draft→submitted→approved→ordered) + approval history |
| Blanket PO list | `/blanket-orders` | `CnIndexPage` + `useListView` |
| Blanket PO detail | `/blanket-orders/:id` | `CnDetailPage` + spend consumption progress bar + release schedule table |
| Statement of work list | `/statements-of-work` | `CnIndexPage` + `useListView` |
| Statement of work detail | `/statements-of-work/:id` | `CnDetailPage` + deliverables and milestones tables |
| Audit log list | `/audit-log` | `CnIndexPage` (read-only) + `useListView` with filter by entityType, actionType, date |
| Settings | `/settings` | `CnVersionInfoCard` → `CnRegisterMapping` → DigiInkoop connection settings |

## Navigation (MainMenu)

```
- Dashboard
- Catalogs
  - Catalog items
- Purchase Requisitions
- Blanket Orders
- Statements of Work
- Audit Log
─────────────
⚙ Settings
```

## Background jobs

| Job | Schedule | Logic |
|-----|----------|-------|
| `ContractExpiryNotificationJob` | Daily | Find CatalogItems with `validUntil` within 60 days; send NotificationService alert to category manager; write ProcurementAuditLog entry |
| `MaverickSpendDetectionJob` | Weekly | Find PurchaseOrders without contract reference; cross-reference by supplier + category against active Contracts; flag matches as maverick; write MaverickSpendAlert objects |
| `MaverickSpendMonthlyReportJob` | Monthly (1st) | Aggregate maverick spend by department; send email summary via NotificationService to department managers above threshold |

## DigiInkoop integration

The `DigiInkooopOrderService` (custom) translates confirmed PurchaseRequisition objects into outbound DigiInkoop electronic order messages (UBL 2.1 format) and dispatches via the DigiInkoop API endpoint configured in Settings. Registered as a WebhookService subscriber on the `PurchaseRequisition.approved` event.

## OCI punchout integration

The `OciPunchoutService` (custom) initiates an OCI session to a configured supplier punchout endpoint. On return, the cXML `CXML_TO` hook populates a new PurchaseRequisition with pre-filled CatalogItem data from the supplier's live catalog.

## Seed data

The following seed objects are defined in `lib/Settings/shillinq_register.json` using the `@self` envelope pattern. Slugs are stable identifiers used for idempotent re-import.

### ProcurementCatalog — 3 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "ProcurementCatalog", "slug": "cat-gemeente-ict-2025" },
    "catalogNumber": "CAT-2025-001",
    "catalogName": "ICT Middelen Gemeente Utrecht 2025",
    "description": "Goedgekeurde ICT producten en licenties voor gemeentelijke afdelingen, gebaseerd op raamovereenkomst ICT-2024-007",
    "catalogFormat": "internal",
    "status": "active",
    "validFrom": "2025-01-01T00:00:00+01:00",
    "validUntil": "2026-12-31T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementCatalog", "slug": "cat-kantoorartikelen-2025" },
    "catalogNumber": "CAT-2025-002",
    "catalogName": "Kantoorartikelen en Verbruiksmateriaal",
    "description": "Standaard kantoorbenodigdheden voor alle gemeentelijke diensten — inclusief papier, schrijfmaterialen en kleine hulpmiddelen",
    "catalogFormat": "cif",
    "status": "active",
    "validFrom": "2025-03-01T00:00:00+01:00",
    "validUntil": "2026-02-28T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementCatalog", "slug": "cat-inhuur-2024-archief" },
    "catalogNumber": "CAT-2024-003",
    "catalogName": "Inhuur Tijdelijk Personeel 2024",
    "description": "Tariefkaart voor externe inhuur via raamovereenkomst tijdelijk personeel 2022-2024",
    "catalogFormat": "internal",
    "status": "archived",
    "validFrom": "2024-01-01T00:00:00+01:00",
    "validUntil": "2024-12-31T23:59:59+01:00"
  }
]
```

### CatalogItem — 4 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "item-laptop-dell-latitude" },
    "itemCode": "IT-LAPTOP-001",
    "itemName": "Dell Latitude 5540 — zakelijke laptop",
    "description": "Intel Core i5, 16 GB RAM, 512 GB SSD, Windows 11 Pro, 3 jaar on-site garantie",
    "basePrice": 1249.00,
    "unit": "stuk",
    "minimumQuantity": 1,
    "leadTime": 10,
    "status": "active",
    "validFrom": "2025-01-01T00:00:00+01:00",
    "validUntil": "2026-12-31T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "item-ms365-licentie" },
    "itemCode": "SW-M365-001",
    "itemName": "Microsoft 365 Business Standard licentie (jaarlijks)",
    "description": "Microsoft 365 Business Standard — 1 gebruiker, jaarlijkse licentie inclusief Exchange, Teams en SharePoint",
    "basePrice": 132.00,
    "unit": "licentie/jaar",
    "minimumQuantity": 5,
    "leadTime": 1,
    "status": "active",
    "validFrom": "2025-01-01T00:00:00+01:00",
    "validUntil": "2026-12-31T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "item-bureaustoel-ergonomisch" },
    "itemCode": "MOB-STOEL-003",
    "itemName": "HAG Capisco ergonomische bureaustoel",
    "description": "Ergonomische bureaustoel met verstelbare rugleuning en zithoogte, NEN-EN 1335 gecertificeerd",
    "basePrice": 895.00,
    "unit": "stuk",
    "minimumQuantity": 1,
    "leadTime": 15,
    "status": "active",
    "validFrom": "2025-03-01T00:00:00+01:00",
    "validUntil": "2026-02-28T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "item-a4-papier-doos" },
    "itemCode": "KA-PAPIER-001",
    "itemName": "A4 kopieerpapier 80 g/m² — doos 5 riemen",
    "description": "Wit A4 kopieerpapier, 80 g/m², 500 vel per riem, FSC-gecertificeerd",
    "basePrice": 22.50,
    "unit": "doos",
    "minimumQuantity": 2,
    "leadTime": 2,
    "status": "active",
    "validFrom": "2025-03-01T00:00:00+01:00",
    "validUntil": "2026-02-28T23:59:59+01:00"
  }
]
```

### PricingRule — 3 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "rule-laptop-volume-10" },
    "ruleCode": "PR-LAPTOP-VOL-10",
    "description": "Volumekorting laptops: 5% korting bij afname van 10 of meer stuks",
    "ruleType": "volumeDiscount",
    "minQuantity": 10,
    "discountPercentage": 5.0,
    "priority": 1,
    "validFrom": "2025-01-01T00:00:00+01:00",
    "validUntil": "2026-12-31T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "rule-m365-volume-50" },
    "ruleCode": "PR-M365-VOL-50",
    "description": "Volumekorting Microsoft 365: 10% korting bij afname van 50 of meer licenties",
    "ruleType": "volumeDiscount",
    "minQuantity": 50,
    "discountPercentage": 10.0,
    "priority": 1,
    "validFrom": "2025-01-01T00:00:00+01:00",
    "validUntil": "2026-12-31T23:59:59+01:00"
  },
  {
    "@self": { "register": "shillinq", "schema": "PricingRule", "slug": "rule-kantoor-q4-promo" },
    "ruleCode": "PR-KA-Q4-2025",
    "description": "Periodieke korting kantoorartikelen Q4 2025: 8% korting op alle kantoorartikelen oktober–december",
    "ruleType": "periodDiscount",
    "discountPercentage": 8.0,
    "priority": 2,
    "validFrom": "2025-10-01T00:00:00+02:00",
    "validUntil": "2025-12-31T23:59:59+01:00"
  }
]
```

### PurchaseRequisition — 3 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "req-2025-0142" },
    "requisitionNumber": "REQ-2025-0142",
    "requisitionDate": "2025-09-15T09:30:00+02:00",
    "status": "approved",
    "purpose": "Vervanging verouderde laptops afdeling Burgerzaken — 8 stuks Dell Latitude 5540 ter vervanging van apparaten ouder dan 5 jaar",
    "deliveryDate": "2025-10-15T00:00:00+02:00",
    "customFields": { "kostenplaats": "BZ-2025", "project": "Digitalisering Balie" },
    "totalAmount": 9992.00
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "req-2025-0189" },
    "requisitionNumber": "REQ-2025-0189",
    "requisitionDate": "2025-10-02T14:00:00+02:00",
    "status": "submitted",
    "purpose": "Aanschaf 25 extra Microsoft 365 Business Standard licenties voor nieuwe medewerkers afdeling Ruimtelijke Ordening",
    "deliveryDate": "2025-10-10T00:00:00+02:00",
    "customFields": { "kostenplaats": "RO-2025", "project": "" },
    "totalAmount": 3300.00
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "req-2025-0201" },
    "requisitionNumber": "REQ-2025-0201",
    "requisitionDate": "2025-10-18T11:15:00+02:00",
    "status": "draft",
    "purpose": "Aanvulling kantoorartikelen depot vierde kwartaal 2025",
    "deliveryDate": "2025-11-01T00:00:00+01:00",
    "customFields": { "kostenplaats": "FACILITAIR-2025" },
    "totalAmount": 450.00
  }
]
```

### BlanketPurchaseOrder — 3 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "BlanketPurchaseOrder", "slug": "bpo-2025-ict-raam" },
    "blanketPoNumber": "BPO-2025-001",
    "validFrom": "2025-01-01T00:00:00+01:00",
    "validUntil": "2025-12-31T23:59:59+01:00",
    "totalAuthorizedAmount": 150000.00,
    "consumedAmount": 67340.00,
    "remainingAmount": 82660.00,
    "releaseSchedule": [
      { "releaseDate": "2025-04-01", "amount": 50000.00, "description": "Q2 ICT inkopen" },
      { "releaseDate": "2025-07-01", "amount": 50000.00, "description": "Q3 ICT inkopen" },
      { "releaseDate": "2025-10-01", "amount": 50000.00, "description": "Q4 ICT inkopen" }
    ],
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "BlanketPurchaseOrder", "slug": "bpo-2025-facilitair" },
    "blanketPoNumber": "BPO-2025-002",
    "validFrom": "2025-03-01T00:00:00+01:00",
    "validUntil": "2026-02-28T23:59:59+01:00",
    "totalAuthorizedAmount": 30000.00,
    "consumedAmount": 8920.00,
    "remainingAmount": 21080.00,
    "releaseSchedule": [],
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "BlanketPurchaseOrder", "slug": "bpo-2024-inhuur-gesloten" },
    "blanketPoNumber": "BPO-2024-003",
    "validFrom": "2024-01-01T00:00:00+01:00",
    "validUntil": "2024-12-31T23:59:59+01:00",
    "totalAuthorizedAmount": 200000.00,
    "consumedAmount": 198450.00,
    "remainingAmount": 1550.00,
    "releaseSchedule": [],
    "status": "closed"
  }
]
```

### StatementOfWork — 3 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "StatementOfWork", "slug": "sow-2025-migratie-zaaksysteem" },
    "sowNumber": "SOW-2025-001",
    "sowDate": "2025-06-01T00:00:00+02:00",
    "title": "Migratie zaaksysteem naar OpenZaak — implementatie en datamigratiediensten",
    "description": "Volledige migratie van het huidige zaaksysteem naar OpenZaak inclusief dataconversie, integratie met BRP en BAG, en opleiding van 40 medewerkers",
    "scope": "In scope: analyse, migratie, integratie, opleiding. Buiten scope: aanpassing bestaande formulieren, koppeling met externe portalen",
    "deliverables": [
      { "title": "Migratieplan", "description": "Technisch migratieplan inclusief risico-analyse", "dueDate": "2025-07-15" },
      { "title": "Testrapport", "description": "Resultaten acceptatietesten met testscripts", "dueDate": "2025-09-30" },
      { "title": "Go-live rapport", "description": "Productieomgeving actief en gebruikers opgeleid", "dueDate": "2025-11-01" }
    ],
    "milestones": [
      { "title": "Kick-off en analyse", "completionDate": "2025-07-15", "invoiceAmount": 15000.00, "invoiceTrigger": "Goedkeuring migratieplan" },
      { "title": "Testfase afgerond", "completionDate": "2025-09-30", "invoiceAmount": 20000.00, "invoiceTrigger": "Ondertekend testrapport" },
      { "title": "Go-live", "completionDate": "2025-11-01", "invoiceAmount": 15000.00, "invoiceTrigger": "Productie acceptatie" }
    ],
    "totalValue": 50000.00,
    "currency": "EUR",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "StatementOfWork", "slug": "sow-2025-cybersec-audit" },
    "sowNumber": "SOW-2025-002",
    "sowDate": "2025-08-01T00:00:00+02:00",
    "title": "Cybersecurity audit en penetratietest gemeentelijke systemen",
    "description": "Technische audit van gemeentelijke IT-infrastructuur inclusief externe penetratietest conform BIO (Baseline Informatiebeveiliging Overheid)",
    "scope": "In scope: netwerkscan, webapplicatie pentest, rapportage met bevindingen en aanbevelingen",
    "deliverables": [
      { "title": "Auditrapport", "description": "Volledig auditrapport met risicoklassificatie", "dueDate": "2025-10-15" }
    ],
    "milestones": [
      { "title": "Auditrapport opgeleverd", "completionDate": "2025-10-15", "invoiceAmount": 18500.00, "invoiceTrigger": "Aanvaarding rapport" }
    ],
    "totalValue": 18500.00,
    "currency": "EUR",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "StatementOfWork", "slug": "sow-2024-website-herontwerp" },
    "sowNumber": "SOW-2024-001",
    "sowDate": "2024-02-01T00:00:00+01:00",
    "title": "Herontwerp gemeentelijke website — NL Design System implementatie",
    "description": "Herontwerp en herbouw van de gemeentelijke website conform NL Design System, inclusief migratie van bestaande content",
    "scope": "In scope: ontwerp, bouw, content migratie. Buiten scope: beheer na oplevering",
    "deliverables": [],
    "milestones": [],
    "totalValue": 75000.00,
    "currency": "EUR",
    "status": "completed"
  }
]
```

### ProcurementAuditLog — 4 seed objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "ProcurementAuditLog", "slug": "aud-2025-0001" },
    "auditId": "AUD-2025-0001",
    "entityType": "requisition",
    "entityId": "req-2025-0142",
    "actionType": "approved",
    "timestamp": "2025-09-18T10:45:00+02:00",
    "reason": "Aanvraag goedgekeurd na verificatie budgetruimte kostenplaats BZ-2025. Conform mandaat afdeling Burgerzaken.",
    "changes": {},
    "referenceDocuments": ["REQ-2025-0142", "BPO-2025-001"]
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementAuditLog", "slug": "aud-2025-0002" },
    "auditId": "AUD-2025-0002",
    "entityType": "catalogItem",
    "entityId": "item-ms365-licentie",
    "actionType": "updated",
    "timestamp": "2025-10-01T09:00:00+02:00",
    "reason": "Prijsindexatie per 1 oktober 2025 conform contractbepaling artikel 7.3",
    "changes": { "basePrice": { "old": 125.00, "new": 132.00 } },
    "referenceDocuments": ["CAT-2025-001", "CONTRACT-2024-ICT-007"]
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementAuditLog", "slug": "aud-2025-0003" },
    "auditId": "AUD-2025-0003",
    "entityType": "requisition",
    "entityId": "req-2025-0189",
    "actionType": "created",
    "timestamp": "2025-10-02T14:00:00+02:00",
    "reason": "",
    "changes": {},
    "referenceDocuments": ["REQ-2025-0189"]
  },
  {
    "@self": { "register": "shillinq", "schema": "ProcurementAuditLog", "slug": "aud-2025-0004" },
    "auditId": "AUD-2025-0004",
    "entityType": "catalogItem",
    "entityId": "item-bureaustoel-ergonomisch",
    "actionType": "deactivated",
    "timestamp": "2025-10-10T16:30:00+02:00",
    "reason": "Artikel gedeactiveerd wegens ophanden zijnde verloopdatum raamovereenkomst Meubilair (28-02-2026). Categoriemanager geïnformeerd via automatisch signaal.",
    "changes": { "status": { "old": "active", "new": "discontinued" } },
    "referenceDocuments": ["CAT-2025-002", "CONTRACT-2025-MOB-001"]
  }
]
```
