# Spec: Catalog Management

Entities: `ProcurementCatalog`, `CatalogItem`, `ProcurementCategory`, `Supplier`

---

## REQ-CAT-001: Internal Product Catalog with Negotiated Pricing

Users can browse and search the internal procurement catalog to find pre-approved products
and services at contracted prices. Each `CatalogItem` displays the negotiated unit price,
supplier, lead time, and applicable category.

#### Scenario 1: Browse catalog items by category

- **GIVEN** a user with procurement access is on the catalog items index page
- **WHEN** they select a `ProcurementCategory` facet in the `CnFacetSidebar`
- **THEN** the `CnDataTable` shows only `CatalogItem` records belonging to that category
- **AND** each row shows name, SKU, unit price, supplier name, and availability status

#### Scenario 2: Search catalog items by name or SKU

- **GIVEN** a user is on the catalog items index page
- **WHEN** they type a search term in the `CnFilterBar` search input
- **THEN** `IndexService` performs full-text search across `CatalogItem.name`, `description`, and `sku`
- **AND** results are returned within 500 ms for catalogs up to 10,000 items

#### Scenario 3: View catalog item detail with negotiated price

- **GIVEN** a user clicks a `CatalogItem` row
- **WHEN** the detail page loads
- **THEN** the page shows unit price, currency, unit of measure, lead time in days,
  linked `Supplier`, linked `ProcurementCatalog`, and the active `PricingRule` if one exists
- **AND** the `CnObjectSidebar` shows Files tab (for item datasheets) and Audit Trail tab

#### Scenario 4: Catalog item not available outside validity window

- **GIVEN** a `ProcurementCatalog` has a `validUntil` date in the past
- **WHEN** a user loads the catalog items index
- **THEN** items from that catalog are shown with an `expired` status badge
- **AND** an expired item cannot be added to a new `PurchaseRequisition`

---

## REQ-CAT-002: Catalog Content Management with Bulk Upload and Enrichment

Catalog administrators can manage the full lifecycle of catalog content: create individual
items, upload batches via CSV/Excel, enrich with descriptions and datasheets, and publish
to procurement users.

#### Scenario 1: Bulk import catalog items from CSV

- **GIVEN** a catalog administrator opens the catalog items index page
- **WHEN** they click "Importeren" and upload a CSV file via `CnMassImportDialog`
- **THEN** `ImportService` validates each row against the `CatalogItem` schema
- **AND** valid rows are created as `CatalogItem` objects; invalid rows are reported
  with the column name and error message
- **AND** the import summary shows total rows, imported count, and failed count

#### Scenario 2: Enrich a catalog item with a datasheet file

- **GIVEN** a catalog administrator is on a `CatalogItem` detail page
- **WHEN** they upload a PDF via the Files tab in `CnObjectSidebar`
- **THEN** `FileService` stores the file linked to the `CatalogItem` object
- **AND** the file is visible to all users with catalog read access

#### Scenario 3: Export catalog items to Excel

- **GIVEN** a catalog administrator is on the catalog items index page
- **WHEN** they click "Exporteren" and select Excel format via `CnMassExportDialog`
- **THEN** `ExportService` generates an Excel file with all visible columns
- **AND** the file downloads in the browser

#### Scenario 4: Create a new procurement catalog

- **GIVEN** a catalog administrator clicks "Catalogus toevoegen" on the catalogs page
- **WHEN** they fill in the `CnFormDialog` with name, description, validFrom, validUntil
- **THEN** a new `ProcurementCatalog` object is saved via `ObjectService.saveObject()`
- **AND** the new catalog appears in the index with status `draft` until published

---

## REQ-CAT-003: Composite Item Management for Kits and Bundles

`CatalogItem` objects can be defined as composite items (kits or bundles) that reference
multiple component `CatalogItem` records. The kit price is derived from component prices
unless a fixed bundle price is set.

#### Scenario 1: Create a composite catalog item (kit)

- **GIVEN** a catalog administrator is creating a new `CatalogItem`
- **WHEN** they enable the "Samengesteld artikel" toggle in the `CnFormDialog`
- **THEN** a components field appears listing selectable `CatalogItem` records
- **AND** on save, the `CatalogItem.components` array is stored as OpenRegister relations
  referencing the component item slugs

#### Scenario 2: Kit price is auto-calculated from components

- **GIVEN** a composite `CatalogItem` has no fixed bundle price set
- **WHEN** a user views the detail page
- **THEN** `CatalogController` sums the `unitPrice` of all component items
- **AND** displays the calculated kit price with a "berekende prijs" indicator

#### Scenario 3: Fixed bundle price overrides component sum

- **GIVEN** a composite `CatalogItem` has `fixedBundlePrice` set
- **WHEN** a user views the detail page
- **THEN** the fixed bundle price is shown instead of the component sum
- **AND** the component items are listed for reference with their individual prices

---

## REQ-CAT-004: Procurement Shop with User-Specific Product Views

Procurement users see a curated catalog view tailored to their role, department, and
entity. Items outside their `ProcurementCategory` access or entity scope are hidden.

#### Scenario 1: User sees only catalogs relevant to their entity

- **GIVEN** a user is associated with organisation `gemeente-westerveld`
- **WHEN** they open the procurement catalog index
- **THEN** only `ProcurementCatalog` objects with `organisation = gemeente-westerveld`
  or organisation `null` (global) are listed
- **AND** catalogs belonging to other organisations are not visible

#### Scenario 2: Catalog items filtered by user's category permissions

- **GIVEN** a user has read access to categories `kantoorbenodigdheden` and `kantoormeubilair`
  but not `ict-apparatuur`
- **WHEN** they load the catalog items index
- **THEN** only items in their permitted categories are shown
- **AND** `AuthorizationService` enforces this at the API level, not only in the UI

#### Scenario 3: Mobile access to catalog

- **GIVEN** a user accesses the catalog on a 375 px wide mobile browser
- **WHEN** the catalog items index page loads
- **THEN** the `CnDataTable` switches to a card layout (`CnCardGrid`) on screens < 768 px
- **AND** all interactive elements (search, filters, add-to-requisition) are keyboard-accessible
  and meet WCAG AA contrast requirements per ADR-010
