---
kind: code
depends_on: []
chain: []
title: "Catalog & Purchase Management — Shillinq — Ai"
change: catalog-purchase-management-ai
date: 2026-05-20
status: proposed
---

# Proposal: Catalog & Purchase Management — Shillinq — Ai

## Summary

This change introduces AI-powered catalog management, purchase order processing, and procurement
intelligence capabilities into Shillinq. The implementation leverages OpenRegister's `ChatService`,
`VectorizationService`, and `TextExtractionService` to deliver 23 AI-driven procurement features
spanning the full Source-to-Pay (S2P) lifecycle — from catalog search and OCR-based PO ingestion
to spend classification and autonomous procurement agents.

All features operate on existing Shillinq entities (`CatalogItem`, `ProcurementCatalog`,
`PurchaseOrder`, `PurchaseRequisition`, `SpendTransaction`, `SpendCategory`, `ProcurementCategory`,
`ApprovalChain`, `Supplier`, `GoodsReceipt`) — no new schemas are introduced.

## Problem Statement

Procurement teams managing Shillinq currently face:

- **Manual catalog curation** — catalog items require individual validation and updates without
  automated consistency checks or cross-catalog duplicate detection.
- **Paper/PDF purchase orders** — incoming POs from suppliers arrive as unstructured documents;
  staff re-key data into PurchaseOrder records manually.
- **Opaque spend patterns** — SpendTransaction records accumulate without automatic categorisation;
  reporting requires manual triage.
- **Rigid approval routing** — ApprovalChain rules are static; bottlenecks caused by absent
  approvers go undetected until SLAs are breached.
- **No conversational interface** — querying procurement history requires database exports or
  custom reports; non-technical stakeholders cannot self-serve.

## Features

| # | Feature | Demand | Category |
|---|---------|--------|----------|
| 1 | Catalog content management with automated validation and update workflow | 1760 | ai |
| 2 | AI procurement performance management aligned to financial goals | 1709 | ai |
| 3 | Purchase Order Management with AI-Based OCR | 1649 | ai |
| 4 | CIF/cXML catalog upload and management with automated validation and publishing | 1594 | ai |
| 5 | 30+ ready-to-use Procurement AI Agents for task automation across S2P lifecycle | 1043 | ai |
| 6 | Spend visibility across all procurement channels with automated category classification | 556 | ai |
| 7 | 30+ ready-to-use procurement AI agents for workflow automation | 494 | ai |
| 8 | Navi AI Analytics Agent: interactive procurement data visualization and reporting | 432 | ai |
| 9 | Automated PO creation with direct material support including BOM-based procurement | 401 | ai |
| 10 | Automated report generation from historical procurement data | 352 | ai |
| 11 | JAI Copilot AI assistant providing procurement insights and workflow automation | 286 | ai |
| 12 | AI Strategies Agent transforming spend data into actionable procurement strategies | 281 | ai |
| 13 | AI chatbot Spend Analyst for conversational queries on procurement data | 281 | ai |
| 14 | AI Request Creation Agent: unstructured documents to purchase requisitions | 179 | ai |
| 15 | AI-powered catalog search with natural language queries and smart recommendations | 173 | ai |
| 16 | Custom AI Agent Creation for Procurement Tasks | 145 | ai |
| 17 | AI-powered demand detection identifying spending pattern changes and procurement needs | 116 | ai |
| 18 | Smart catalog with AI-driven product recommendations based on purchase history | 110 | ai |
| 19 | Advanced PO approval workflows with automated routing and bottleneck elimination | 49 | ai |
| 20 | Automated PO creation from approved requisitions with configurable business rules | 25 | ai |
| 21 | Centralized PO creation with automated generation from approved purchase requests | 18 | ai |
| 22 | Automated reordering for recurring supplies with scheduled purchase generation | 14 | ai |
| 23 | Automated PO creation from approved requests reducing manual errors | 9 | ai |

**Total demand: 10,661** across 23 features. Average competitor coverage: ~3%, confirming a
significant blue-ocean gap for open-source AI-driven procurement.

## Stakeholders

| Role | Concern | Impact |
|------|---------|--------|
| **Inkoopmedewerker** (Procurement Officer) | Automate repetitive PO entry and catalog curation | High — daily workflow |
| **Financieel Controller** (Financial Controller) | Spend visibility, budget alignment, automated reporting | High — monthly close |
| **Systeembeheerder** (System Administrator) | Configure AI agents, register catalogs, manage integrations | Medium — initial setup |
| **Afdelingshoofd** (Department Head) | Self-service spend queries, performance dashboards | Medium — periodic review |
| **Leveranciersmanager** (Supplier Manager) | Supplier KPI tracking, catalog quality oversight | Medium — supplier reviews |
| **IT-beheerder** (IT Manager) | Nextcloud deployment, OpenRegister connectivity, API keys | Low — infrastructure |

## User Stories

**US-01 — Automated Catalog Validation**
As an Inkoopmedewerker, I want the system to automatically validate CatalogItem records on
import so that I can identify pricing anomalies and missing mandatory fields without manual review.

**US-02 — OCR-Based PO Ingestion**
As an Inkoopmedewerker, I want to upload a scanned or PDF purchase order and have the system
extract and pre-populate a PurchaseOrder record so that I can avoid re-keying data from paper.

**US-03 — CIF/cXML Catalog Import**
As a Systeembeheerder, I want to upload a CIF or cXML catalog file and have items validated and
published to ProcurementCatalog so that supplier catalogs are onboarded consistently.

**US-04 — Spend Category Classification**
As a Financieel Controller, I want SpendTransaction records to be automatically assigned a
SpendCategory so that I can generate spend-by-category reports without manual tagging.

**US-05 — Conversational Procurement Query**
As an Afdelingshoofd, I want to ask procurement questions in plain Dutch ("Wat hebben we dit
kwartaal besteed aan kantoorartikelen?") and receive a data-backed answer so that I can
self-serve without IT involvement.

**US-06 — AI Procurement Strategy Recommendations**
As a Financieel Controller, I want the system to analyse aggregated SpendTransaction data and
suggest consolidation or substitution strategies so that I can reduce procurement costs.

**US-07 — Demand Anomaly Alerts**
As an Inkoopmedewerker, I want to receive a Nextcloud notification when spending on a
ProcurementCategory deviates significantly from its historical baseline so that I can investigate
and act before budget limits are breached.

**US-08 — Automated PO Generation from Requisitions**
As an Inkoopmedewerker, I want approved PurchaseRequisition records to automatically generate a
draft PurchaseOrder for review so that the S2P cycle shortens and errors are reduced.

**US-09 — Smart Catalog Recommendations**
As an Inkoopmedewerker, I want the catalog search to suggest CatalogItem alternatives based on
purchase history and item similarity so that I select optimal items faster.

**US-10 — AI Request Creation from Unstructured Docs**
As an Inkoopmedewerker, I want to upload an email, memo, or scanned form and have the system
create a draft PurchaseRequisition with extracted line items so that ad-hoc requests are
processed quickly.

## Customer Journeys

### Journey 1 — Catalog Onboarding (CIF/cXML → CatalogItem)

**Trigger:** Supplier sends a new CIF or cXML catalog file via email or supplier portal.

**Pain point:** Manual item-by-item entry into ProcurementCatalog is time-consuming and
error-prone; inconsistencies across suppliers create downstream pricing issues.

**Steps:**
1. Systeembeheerder uploads CIF/cXML via `CnMassImportDialog`.
2. AI validation service parses structure, flags mandatory field gaps and price outliers.
3. Valid items are staged as draft CatalogItems; invalid rows shown with correction hints.
4. Inkoopmedewerker reviews flagged items and approves batch publication.
5. CatalogItems published to ProcurementCatalog; old versions archived.

### Journey 2 — PO Ingestion via OCR

**Trigger:** Paper PO or PDF arrives from an external customer/supplier.

**Pain point:** Re-keying unstructured PO data causes transcription errors and delays.

**Steps:**
1. Inkoopmedewerker uploads PDF to PurchaseOrder record via file tab.
2. AI OCR service extracts header fields (supplier, date, reference, currency) and line items.
3. Draft PurchaseOrder pre-populated with extracted data for review.
4. Inkoopmedewerker corrects low-confidence fields highlighted by the UI.
5. PurchaseOrder confirmed and routed to ApprovalChain.

### Journey 3 — Conversational Spend Analysis (Spend Analyst Chat)

**Trigger:** Department head wants a quick answer on quarterly spend without opening a report.

**Pain point:** Finance staff required to run manual exports; non-technical stakeholders
blocked from self-service.

**Steps:**
1. User opens Spend Analyst chat panel in Shillinq dashboard.
2. User types natural-language query in Dutch.
3. ChatService retrieves relevant SpendTransaction + SpendCategory context via RAG.
4. AI generates a concise, data-backed summary with optional chart.
5. User can drill down by asking follow-up questions.

### Journey 4 — Automated PO Generation

**Trigger:** PurchaseRequisition status transitions to `approved`.

**Pain point:** Manual PO creation from approved requisitions is duplicative effort.

**Steps:**
1. WorkflowEngineRegistry fires event on ApprovalRequest completion.
2. AI business rules service evaluates requisition lines against preferred suppliers.
3. Draft PurchaseOrder created from matching CatalogItems and Supplier records.
4. Inkoopmedewerker receives Nextcloud notification to review and submit PO.
5. On submit, PurchaseOrder dispatched to supplier (Peppol/email).

## Technical Approach

This change builds **custom business logic services and Vue UI components only**. All CRUD,
search, audit, file management, notifications, and workflow primitives are consumed from
OpenRegister and `@conduction/nextcloud-vue`.

### Custom components to build

| Component | Type | Purpose |
|-----------|------|---------|
| `AiCatalogValidationService` | PHP Service | Validates CatalogItem batches; calls ChatService for anomaly detection |
| `CifCxmlParserService` | PHP Service | Parses CIF 3.0 and cXML 1.2 catalog files into CatalogItem arrays |
| `OcrPurchaseOrderService` | PHP Service | Wraps TextExtractionService + ChatService to extract PO fields from uploaded files |
| `SpendClassificationService` | PHP Service | Classifies SpendTransaction records into SpendCategory using VectorizationService embeddings |
| `ProcurementAgentService` | PHP Service | Orchestrates named AI agent tasks (catalog update, demand detection, strategy recommendations) |
| `RequisitionToPurchaseOrderService` | PHP Service | Business rules: creates PurchaseOrder from approved PurchaseRequisition |
| `DemandAnomalyDetectionJob` | PHP Background Job | Scheduled job: detects spend deviations and dispatches notifications |
| `SpendAnalystController` | PHP Controller | Thin REST controller exposing ChatService conversation for Spend Analyst chat |
| `AiCatalogController` | PHP Controller | Thin REST controller for catalog validation and CIF/cXML import trigger |
| `ProcurementAgentController` | PHP Controller | Thin REST controller for agent task dispatch and status |
| `SpendAnalystChat.vue` | Vue Component | Chat UI panel consuming SpendAnalystController conversation endpoint |
| `CatalogValidationResultsDialog.vue` | Vue Component | Displays validation results from CIF/cXML import with correction workflow |
| `OcrPurchaseOrderPreview.vue` | Vue Component | Highlights extracted PO fields with confidence scores for review |
| `ProcurementAgentPanel.vue` | Vue Component | Lists available AI agents with status, last-run, and trigger controls |
| `DemandAnomalyWidget.vue` | Vue Component | Dashboard widget: spend deviation alerts per ProcurementCategory |

### Deliberately NOT building

- CRUD pages for CatalogItem, PurchaseOrder, PurchaseRequisition → use `CnIndexPage` + `CnDetailPage`
- Approval chain routing → use existing `ApprovalChain` + `WorkflowEngineRegistry`
- Search/facet for catalog → use `VectorizationService` + `CnFacetSidebar` (already provided)
- Spend reports → use `CnDashboardPage` + `CnChartWidget`
- File upload → use `FileService` + `CnFilesTab`

## Compliance and Constraints

- All AI inference calls MUST go through OpenRegister's `ChatService` — no direct external LLM calls.
- No PII (BSN, NAW data) in AI prompts; use anonymised identifiers and aggregated amounts.
- CIF/cXML import MUST be idempotent: re-importing the same catalog version skips existing items.
- Dutch locale: all user-visible labels via `t(appName, 'key')`, currency formatting respects locale.
- Adheres to ADR-003 (Controller → Service → Mapper), ADR-004 (Vue 2 + Pinia), ADR-005 (security),
  ADR-007 (nl/en translations), ADR-008 (PHPUnit + Playwright tests).
