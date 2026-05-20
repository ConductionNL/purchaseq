# Spec: Catalog Browsing & Punch-out

**Change:** catalog-purchase-management-other-t3
**Capability:** catalog-browsing, catalog-punchout

---

## ADDED Requirements

### REQ-CAT-001: Guided Catalog Browsing

Users must be able to browse `ProcurementCatalog` and `CatalogItem` objects through a guided buying experience with enriched attributes, UNSPSC classification codes, pricing display, and faceted search navigation.

#### Scenario: Browse catalog items with faceted filters

- **GIVEN** a user navigates to the Catalogus section in Shillinq
- **WHEN** the catalog index page loads
- **THEN** all active `CatalogItem` objects are displayed in a paginated list with name, SKU, unit price (EUR), unit, stock status, and category
- **AND** a facet sidebar shows filterable facets for: category, supplier, classification code, stock status, and price range
- **AND** full-text search is available via the search bar

#### Scenario: View enriched item detail

- **GIVEN** a user clicks on a `CatalogItem` in the catalog list
- **WHEN** the item detail page loads
- **THEN** the page shows all enriched attributes: name, SKU, description, unit price, unit, minimum order quantity, lead time (days), UNSPSC classification code and scheme, stock status, and supplier catalog
- **AND** a quantity input and "Toevoegen aan aanvraag" button are displayed
- **AND** WCAG AA keyboard navigation and focus management are satisfied

#### Scenario: Add item to basket

- **GIVEN** a user is on a `CatalogItem` detail page
- **WHEN** the user enters a quantity ≥ minimumOrderQuantity and clicks "Toevoegen aan aanvraag"
- **THEN** the item and quantity are added to the transient basket store
- **AND** the basket item count in the navigation header updates
- **AND** a success notification confirms the addition (via `NotificationService`)

#### Scenario: Assign cost center on catalog order

- **GIVEN** a user is reviewing the basket before checkout
- **WHEN** the user selects a `CostCenter` from the cost center picker
- **THEN** the selected cost center is associated with all basket items
- **AND** the cost center is pre-populated on the resulting `PurchaseRequisition`

### REQ-CAT-002: Catalog Content Enrichment

`CatalogItem` objects must support standardized attributes and classification codes for interoperability with procurement standards.

#### Scenario: Item has UNSPSC classification

- **GIVEN** a `CatalogItem` exists with `classificationCode` and `classificationScheme` fields set
- **WHEN** a user views the item detail
- **THEN** the classification code and scheme label (e.g. "UNSPSC 44101500") are displayed
- **AND** searching by classification code in the search bar returns matching items

#### Scenario: Import catalog with enriched attributes

- **GIVEN** an admin uploads a catalog CSV with columns: name, sku, unitPrice, category, classificationCode, classificationScheme, stockStatus, minimumOrderQuantity
- **WHEN** the import completes via `CnMassImportDialog`
- **THEN** all items are created as `CatalogItem` OpenRegister objects with all provided attributes populated

### REQ-CAT-003: Punch-out Catalog Integration (Catalogusbeheer)

Shillinq must support cXML/OCI punch-out catalog integration so users can browse supplier catalogs externally and return basket items directly into a purchase requisition.

#### Scenario: Launch punch-out session

- **GIVEN** a `ProcurementCatalog` has a `punchoutUrl` configured
- **WHEN** a user clicks "Punch-out openen" on the catalog detail page
- **THEN** `CatalogPunchoutService::generateSetupRequest()` generates a valid cXML PunchOutSetupRequest with the Shillinq return URL
- **AND** the user is redirected to the supplier's punch-out portal in a new window

#### Scenario: Receive punch-out basket

- **GIVEN** the user has selected items in the external punch-out portal
- **WHEN** the supplier redirects back to the Shillinq cXML return endpoint with a PunchOutOrderMessage
- **THEN** `CatalogPunchoutService::receiveOrderMessage()` parses the XML payload
- **AND** the returned items are displayed as basket items for requisition creation
- **AND** no items are persisted as `CatalogItem` objects without explicit user confirmation

#### Scenario: Catalog without punch-out URL

- **GIVEN** a `ProcurementCatalog` has no `punchoutUrl`
- **WHEN** a user views the catalog detail page
- **THEN** the "Punch-out openen" button is not displayed
- **AND** only locally stored `CatalogItem` objects from that catalog are shown

### REQ-CAT-004: Catalog-Based Ordering with Pricing Display

Users must be able to create a `PurchaseOrder` or `PurchaseRequisition` directly from catalog items with live pricing applied.

#### Scenario: Checkout basket as purchase requisition

- **GIVEN** a user has one or more items in the basket with quantities set
- **WHEN** the user clicks "Aanvraag indienen"
- **THEN** a `PurchaseRequisition` object is created with basket items as line items
- **AND** pricing rules are evaluated via `PricingRuleService::applyRules()` and applied discounts shown
- **AND** the requisition form pre-populates title, cost center, requester, and line items
- **AND** the user can edit the justification field before submitting

#### Scenario: Catalog item price display with discount indicator

- **GIVEN** active `PricingRule` objects match a `CatalogItem` for the current user's organization
- **WHEN** the item is displayed in the catalog list or detail page
- **THEN** the discounted price is shown alongside the original price (struck through)
- **AND** a discount badge indicates the percentage or fixed amount saved
