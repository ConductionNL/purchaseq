# Specs: Catalog & Purchase Management — Shillinq — Other T2

## Requirement Format

Requirements use `REQ-[AREA]-[NNN]` identifiers with GIVEN/WHEN/THEN acceptance
criteria. Areas: `CAT` (Catalog), `REQ` (Requisition), `PO` (Purchase Order),
`BLK` (Blanket Order), `QTE` (Quote), `AUT` (Automation), `ANA` (Analytics),
`CPL` (Compliance).

---

## CAT — Catalog Management

### REQ-CAT-001: Create and manage internal product catalogs

**Feature:** Product & Service Catalog (demand: 15), Custom Internal Product Catalogs (demand: 14), Internal Catalog with Curated Product Selection (demand: 6)

**GIVEN** the user is a Systeembeheerder or Inkoper with catalog management rights  
**WHEN** they navigate to `/procurement/catalogs` and click "Nieuwe catalogus"  
**THEN** a `CnFormDialog` opens with fields: name, description, validFrom, validUntil, isHostedBuyer, supplier (optional)  
**AND** on save, a new `ProcurementCatalog` object is created in OpenRegister with `status: draft`  
**AND** the catalog appears in the catalog list  

**Accessibility:** Form fields have visible labels; keyboard-navigable; error messages announced via aria-live.

---

### REQ-CAT-002: Add categories to procurement systems

**Feature:** Add Categories to Procurement Systems (demand: 69), Product Classification (300+ categories) (demand: 19)

**GIVEN** a `ProcurementCatalog` exists in `status: active`  
**WHEN** an Inkoper navigates to `/procurement/categories` and creates a `ProcurementCategory`  
**THEN** the category is saved with a CPV code, `type` (direct or indirect), and optional `parent` category  
**AND** the category hierarchy is displayed as a tree in the category index page  
**AND** the category is available for selection on `CatalogItem`, `PurchaseRequisition`, and `SavingsOpportunity` forms  

---

### REQ-CAT-003: Add catalog items with pricing

**Feature:** Internal Product Catalog with Configurable Pricing (demand: 11), Volume-Based Pricing with Quantity Discounts (demand: 5), Tiered Pricing with Graduated Rates (demand: 4)

**GIVEN** a `ProcurementCatalog` exists and the user has catalog management rights  
**WHEN** they add a `CatalogItem` with `unitPrice`, `vatRate`, and optional `pricingTiers`  
**THEN** the item is saved and appears in the catalog detail view  
**AND** when an item's `pricingTiers` is non-empty, the displayed price reflects the applicable tier for the entered quantity  
**AND** volume discounts are applied automatically when quantity meets a tier threshold  

---

### REQ-CAT-004: Browse and search catalog with faceted filtering

**Feature:** Add Catalog Items to Order Basket (demand: 15), Hosted Buyer Catalogs with Managed Content (demand: 17)

**GIVEN** a catalog with `status: active` contains `CatalogItem` objects  
**WHEN** an Interne aanvrager navigates to `/procurement/catalog-items`  
**THEN** they see a list/card view of items with `CnFacetSidebar` filtering by category, supplier, price range, and CPV code  
**AND** full-text search via `IndexService` is available  
**AND** clicking "Toevoegen aan mand" on an item initiates a new `PurchaseRequisition` with the item pre-populated  
**AND** hosted buyer catalogs (isHostedBuyer: true) display a supplier badge  

---

### REQ-CAT-005: Free-text and catalog-based ordering with custom form fields

**Feature:** Free-text and catalog-based ordering with custom form fields (demand: 5)

**GIVEN** an Interne aanvrager is creating a `PurchaseRequisition`  
**WHEN** they choose "vrije tekst bestelling" instead of a catalog item  
**THEN** they can enter item description, estimated price, and category manually  
**AND** the `customFields` property on `PurchaseRequisition` stores any app-configured extra fields  
**AND** the requisition is otherwise processed identically to a catalog-based requisition  

---

## REQ — Purchase Requisition

### REQ-REQ-001: Create purchase requisition with configurable forms

**Feature:** Purchase requisition with configurable forms supporting both direct and indirect categories (demand: 56), Purchase Requisitions (demand: 4)

**GIVEN** an authenticated user  
**WHEN** they navigate to `/procurement/requisitions/new`  
**THEN** a `PurchaseRequisition` form is displayed, schema-driven via `CnFormDialog`  
**AND** the form shows fields: title, category (direct/indirect), lineItems, requiredByDate, justification, and any `customFields`  
**AND** the user can attach supporting documents via the `CnFilesTab` in the sidebar  
**AND** on submit, `status` transitions from `draft` to `submitted`  

---

### REQ-REQ-002: Attach supporting documents to requisition

**Feature:** Attach supporting documents to requisition (demand: 6)

**GIVEN** a `PurchaseRequisition` in any status  
**WHEN** the user opens the Files tab in `CnObjectSidebar`  
**THEN** they can upload one or more files via `FileService`  
**AND** uploaded files appear in the requisition's file list  
**AND** approvers can view attachments during the approval step  

---

### REQ-REQ-003: View requisition approval status

**Feature:** View requisition approval status (demand: 47)

**GIVEN** a `PurchaseRequisition` with `status: pending_approval`  
**WHEN** the Interne aanvrager opens the requisition detail page  
**THEN** a `CnTimelineStages` component displays the approval workflow stages (Ingediend → In behandeling → Goedgekeurd / Afgewezen)  
**AND** the current approver's name and expected decision date are visible  
**AND** status updates are reflected in real time (on page reload)  

---

### REQ-REQ-004: Auto-approval for low-value requisitions

**Feature:** Auto-approval for low-value requisitions under configurable spend thresholds (demand: 42)

**GIVEN** a system administrator has configured an auto-approval threshold (e.g. €250) in Settings  
**WHEN** a `PurchaseRequisition` is submitted with `totalAmount` ≤ the threshold  
**THEN** `autoApproved` is set to `true` and `status` transitions directly to `approved`  
**AND** no manual approval action is required  
**AND** a Nextcloud notification is sent to the requester confirming auto-approval  
**AND** the `ProcurementAuditLog` records the auto-approval with the threshold value that applied  

---

### REQ-REQ-005: Multi-entity requisition

**Feature:** Multi-entity requisition (demand: 11)

**GIVEN** the user belongs to multiple organizational units  
**WHEN** they create a `PurchaseRequisition`  
**THEN** they can set `isMultiEntity: true` and select multiple organizational units as `entity`  
**AND** the total budget impact is split across the selected entities  
**AND** each entity's approver receives an approval task  

---

## PO — Purchase Order

### REQ-PO-001: Create purchase order

**Feature:** Create purchase order (demand: 53), Purchase order processing (demand: 62)

**GIVEN** an Inkoper has the required permissions  
**WHEN** they click "Nieuwe inkooporder" in `/procurement/purchase-orders`  
**THEN** a `CnFormDialog` opens with supplier, lineItems, requiredDeliveryDate, and paymentTerms  
**AND** on save, a `PurchaseOrder` is created with `status: draft` and an auto-generated `orderNumber`  
**AND** the order number format is `PO-{YYYY}-{sequential five-digit number}` (e.g. `PO-2026-00301`)  

---

### REQ-PO-002: Generate purchase order from requisition

**Feature:** Generate purchase order (demand: 53)

**GIVEN** a `PurchaseRequisition` in `status: approved`  
**WHEN** the Inkoper clicks "Omzetten naar inkooporder" on the requisition detail page  
**THEN** a new `PurchaseOrder` is created pre-populated with the requisition's line items, supplier, and delivery date  
**AND** the `PurchaseRequisition.purchaseOrder` relation is set to the new PO  
**AND** `PurchaseRequisition.status` transitions to `converted_to_po`  

---

### REQ-PO-003: Purchase order templates

**Feature:** Purchase Order Templates (demand: 55), PO templates for standardized ordering with pre-populated fields (demand: 5)

**GIVEN** a `PurchaseOrder` exists with `isTemplate: true`  
**WHEN** an Inkoper creates a new PO and selects "Gebruik sjabloon"  
**THEN** the form pre-populates with the template's supplier, lineItems, and paymentTerms  
**AND** the new PO's `template` relation is set to the source template  
**AND** the resulting PO is a distinct object (not a copy of the template)  

---

### REQ-PO-004: Revise a purchase order

**Feature:** Purchase Order Revision (demand: 53)

**GIVEN** a `PurchaseOrder` in `status: sent` or `acknowledged`  
**WHEN** the Inkoper clicks "Revisie aanmaken" and fills in changed fields plus a reason  
**THEN** a `PurchaseOrderRevision` is created recording the before/after state  
**AND** the PO's `revisions` relation gains the new revision  
**AND** if `totalAmount` changed, the PO's `totalAmount` is updated  
**AND** the supplier is notified of the revision via `NotificationService`  

---

### REQ-PO-005: Digital signature on purchase order

**Feature:** Purchase Order Signatures (demand: 53)

**GIVEN** a `PurchaseOrder` in `status: draft` ready for issuance  
**WHEN** the authorised signatory clicks "Ondertekenen"  
**THEN** `signedBy` is set to the current user's Person record and `signedAt` to the current timestamp  
**AND** `status` transitions from `draft` to `sent`  
**AND** the signature is recorded in `ProcurementAuditLog`  
**AND** the signed PO PDF is generated and attached via `FileService`  

---

### REQ-PO-006: Track catalog order delivery status

**Feature:** Track catalog order delivery status (demand: 30), Confirm receipt of catalog order (demand: 12)

**GIVEN** a `PurchaseOrder` in `status: sent` or `acknowledged`  
**WHEN** the Inkoper or warehouse staff opens the PO detail page  
**THEN** a delivery status timeline is displayed showing expected vs. actual delivery  
**AND** clicking "Ontvangst bevestigen" transitions `status` to `partially_delivered` or `delivered`  
**AND** a `GoodsReceipt` or `ProofOfDelivery` relation is created on confirmation  

---

## BLK — Blanket Purchase Orders

### REQ-BLK-001: Create blanket purchase order with scheduled releases

**Feature:** Blanket purchase orders with scheduled release dates and quantity limits (demand: 29), Blanket Orders (demand: 9)

**GIVEN** an Inkoper  
**WHEN** they create a `BlanketPurchaseOrder` with `validFrom`, `validUntil`, `maxValue`, `maxQuantity`, and `releaseSchedule`  
**THEN** the blanket order is saved in OpenRegister  
**AND** the background job `BlanketOrderReleaseJob` schedules the first `nextReleaseDate` based on `releaseSchedule`  

---

### REQ-BLK-002: Automatic call-off order generation

**Feature:** Blanket purchase orders with scheduled release dates and quantity limits (demand: 29)

**GIVEN** a `BlanketPurchaseOrder` in `status: active` with `releaseSchedule: monthly`  
**WHEN** `nextReleaseDate` is reached  
**THEN** a `PurchaseOrder` (call-off) is automatically created and added to `callOffOrders`  
**AND** `consumedValue` and `consumedQuantity` are updated  
**AND** if `consumedValue` ≥ `maxValue` or `consumedQuantity` ≥ `maxQuantity`, `status` transitions to `exhausted`  
**AND** when `validUntil` passes, `status` transitions to `expired`  

---

### REQ-BLK-003: Manual release of call-off order

**Feature:** Blanket purchase orders with scheduled release dates and quantity limits (demand: 29)

**GIVEN** a `BlanketPurchaseOrder` in `status: active` with `releaseSchedule: manual`  
**WHEN** the Inkoper clicks "Afroep aanmaken"  
**THEN** a `PurchaseOrder` (call-off) is created and linked to the blanket order  
**AND** consumed totals are updated  

---

## QTE — Procurement Quotes

### REQ-QTE-001: Set quote expiry date and send reminders

**Feature:** Set quote expiry date (demand: 69), Online quote/estimate creation with one-click acceptance (demand: 31), Estimate Creation (demand: 5), Commercial Proposals (demand: 5)

**GIVEN** an Inkoper creates a `ProcurementQuote`  
**WHEN** they set `expiryDate` and optionally `expiryReminderDays` (default: 7)  
**THEN** the quote is saved with those values  
**AND** the `QuoteExpiryReminderJob` sends a Nextcloud notification `expiryReminderDays` before `expiryDate`  
**AND** on `expiryDate`, `status` automatically transitions to `expired` if still `received` or `under_review`  

---

### REQ-QTE-002: One-click quote acceptance → generate PO

**Feature:** Online quote/estimate creation with one-click acceptance (demand: 31)

**GIVEN** a `ProcurementQuote` in `status: received` or `under_review`  
**WHEN** the Inkoper clicks "Accepteren"  
**THEN** `status` transitions to `accepted`  
**AND** a `PurchaseOrder` is created from the quote's line items and supplier  
**AND** `ProcurementQuote.purchaseOrder` is set to the new PO  
**AND** the action is recorded in `ProcurementAuditLog`  

---

### REQ-QTE-003: Reject a quote with reason

**Feature:** Online quote/estimate creation with one-click acceptance (demand: 31)

**GIVEN** a `ProcurementQuote` in `status: received` or `under_review`  
**WHEN** the Inkoper clicks "Afwijzen" and enters a reason  
**THEN** `status` transitions to `rejected`  
**AND** the reason is stored in `notes`  
**AND** the supplier (if a `SupplierPortalUser` exists) receives a notification  

---

## AUT — Procurement Automation

### REQ-AUT-001: No-code autopilot for procurement automation

**Feature:** No-code Autopilot for custom procurement automation without developer involvement (demand: 65), Auto Procurement Rules (demand: 10), Procurement automation (demand: 26)

**GIVEN** a Systeembeheerder navigates to Settings → Inkoopautomatisering  
**WHEN** they define a `PolicyRule` (condition + action) via the no-code rule builder  
**THEN** the rule is saved and evaluated by `WorkflowEngineController` on the configured trigger  
**AND** example rules include: "Als categorie = Kantoorartikelen EN bedrag < €500 → automatisch goedkeuren" and "Als leveranciersrisico = hoog → escaleer naar inkoopdirecteur"  
**AND** rules can be enabled/disabled without developer involvement  

---

### REQ-AUT-002: Auto-approval threshold configuration

**Feature:** Auto-approval for low-value requisitions under configurable spend thresholds (demand: 42)

**GIVEN** a Systeembeheerder  
**WHEN** they navigate to Settings → Drempelwaarden and set `autoApprovalThreshold` (e.g. €250)  
**THEN** the threshold is saved via `IAppConfig`  
**AND** all future requisitions with `totalAmount` ≤ threshold are auto-approved (REQ-REQ-004)  
**AND** the threshold can be set per `ProcurementCategory`  

---

## ANA — Procurement Analytics

### REQ-ANA-001: Track procurement savings

**Feature:** Track procurement savings (demand: 21)

**GIVEN** an Inkoper or Financieel medewerker  
**WHEN** they create a `SavingsOpportunity` with `baselineAmount`, `targetAmount`, and `savingsType`  
**THEN** the opportunity is listed in `/procurement/savings`  
**AND** when `actualAmount` is entered and `achievedDate` is set, `status` transitions to `realised`  
**AND** the dashboard KPI card shows total realised savings year-to-date  

---

### REQ-ANA-002: Track procurement KPIs vs targets

**Feature:** Track procurement KPIs vs targets (demand: 21)

**GIVEN** the procurement dashboard is open  
**WHEN** the user views the KPI section  
**THEN** they see four `CnStatsBlock` cards: Openstaande bestellingen, Openstaande aanvragen, Totale bestelwaarde (YTD), Gerealiseerde besparingen (YTD)  
**AND** a `CnChartWidget` shows requisition volume and PO value by month  
**AND** KPIs are refreshed by `ProcurementKpiJob` (weekly) and on-demand via a refresh button  

---

### REQ-ANA-003: Maintain and check procurement dossier completeness

**Feature:** Maintain procurement dossier (demand: 24), Check procurement dossier completeness (demand: 15)

**GIVEN** a `PurchaseOrder` with `totalAmount` above the Dutch public procurement threshold  
**WHEN** the Inkoper views the PO detail page  
**THEN** a completeness indicator shows which required dossier documents are present and which are missing  
**AND** required documents are: aanvraag, goedkeuring, offertes (≥ 3 for amounts > €50.000), gunningsbeslissing  
**AND** the system prevents `status` from transitioning to `sent` if mandatory documents are missing  

---

## CPL — Dutch Compliance & Risk

### REQ-CPL-001: Dutch procurement compliance

**Feature:** Dutch Procurement (demand: 33), E-Procurement (demand: 17), Procurement Module (demand: 18)

**GIVEN** a Dutch government entity user  
**WHEN** a `PurchaseOrder` exceeds the DigiInkoop mandatory electronic procurement threshold  
**THEN** the system flags the PO for DigiInkoop submission  
**AND** the `ProcurementComplianceReport` records the applicable threshold, procedure type, and publication requirement  
**AND** the Inkoper can export the dossier in the format required for DigiInkoop  

---

### REQ-CPL-002: Risk-aware procurement

**Feature:** Risk-aware procurement (demand: 62)

**GIVEN** a `PurchaseRequisition` or `PurchaseOrder` referencing a `Supplier`  
**WHEN** the object is created or updated  
**THEN** the system checks `SupplierRiskProfile` for the linked supplier  
**AND** if `SupplierRiskProfile.riskLevel` is `high`, a warning banner is shown on the detail page  
**AND** a `ComplianceRisk` record is created and linked to the procurement object  
**AND** high-risk supplier orders require a second-level approval even if below the auto-approval threshold  

---

### REQ-CPL-003: Define procurement team and roles

**Feature:** Define procurement team and roles (demand: 33)

**GIVEN** a Systeembeheerder  
**WHEN** they navigate to Settings → Inkoopteam  
**THEN** they can create a `Team` with named `Role` assignments (Inkoper, Budgethouder, Interne aanvrager)  
**AND** each role maps to a set of `Permission` objects in `AuthorizationService`  
**AND** role assignments are reflected in approval chain routing  

---

### REQ-CPL-004: Procurement audit log

**Feature:** Procurement Module (demand: 18), Dutch Procurement (demand: 33)

**GIVEN** any procurement action is performed (create, approve, revise, sign, reject, auto-approve)  
**WHEN** the action completes  
**THEN** a `ProcurementAuditLog` record is created with: action type, actor, timestamp, object reference, and before/after state  
**AND** the audit log is accessible from the object's `CnObjectSidebar` → Audit tab  
**AND** the audit log can be exported as CSV for compliance reporting  

---

## MOB — Mobile & Accessibility

### REQ-MOB-001: Mobile procurement management

**Feature:** Mobile Procurement Management (demand: 40)

**GIVEN** a user accesses Shillinq on a device with viewport width 375px (mobile)  
**WHEN** they navigate to the procurement section  
**THEN** all procurement pages render correctly at 375px–768px viewport widths  
**AND** the catalog browser, requisition form, and PO approval flow are fully functional on mobile  
**AND** touch targets are ≥ 44px (WCAG 2.5.5)  

---

### REQ-MOB-002: WCAG AA compliance across procurement UI

**Feature:** (cross-cutting, ADR-010)

**GIVEN** any procurement page in the application  
**WHEN** it is rendered  
**THEN** all interactive elements have accessible labels  
**AND** color is not the sole conveyor of status information (status badges include text)  
**AND** focus order is logical  
**AND** all forms are keyboard-navigable  

---

## SPD — Spending Cards

### REQ-SPD-001: Virtual and physical spending card management

**Feature:** Virtual and physical purchasing cards (Spending Cards) for on-the-go procurement (demand: 65)

**GIVEN** a Systeembeheerder has configured the spending card integration  
**WHEN** an Inkoper issues a virtual spending card  
**THEN** a `SpendTransaction` is created for each card transaction  
**AND** transactions are categorised via `SpendCategory`  
**AND** card spend is visible in the procurement dashboard  
**AND** card numbers and sensitive data are never stored in application logs or API responses  

---

## DMD — Demand-Based Procurement

### REQ-DMD-001: Demand-based procurement planning

**Feature:** Demand-Based Procurement (demand: 30)

**GIVEN** a Financieel medewerker views the procurement analytics  
**WHEN** they open the "Vraagplanning" section  
**THEN** they can see historical spend by `ProcurementCategory` over the past 12 months  
**AND** the system suggests re-order points based on average monthly consumption  
**AND** they can initiate a `PurchaseRequisition` directly from a demand suggestion  

---

## PCL — Procurement Cloud

### REQ-PCL-001: Cloud-hosted procurement catalog (Procurement Cloud)

**Feature:** Procurement cloud (demand: 57)

**GIVEN** a `ProcurementCatalog` with `isHostedBuyer: true` and an external supplier API configured  
**WHEN** an Inkoper browses the catalog  
**THEN** catalog items are fetched from the supplier's hosted catalog API in real time  
**AND** pricing reflects the live supplier pricing  
**AND** fallback to cached items is used when the external API is unavailable  
