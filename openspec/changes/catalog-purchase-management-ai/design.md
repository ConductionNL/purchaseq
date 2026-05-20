# Design: Catalog & Purchase Management — Shillinq — Ai

**Change:** catalog-purchase-management-ai
**Date:** 2026-05-20

---

## Architecture Overview

This change adds an AI services layer on top of Shillinq's existing procurement domain model.
All domain data remains in OpenRegister objects. The AI layer consists of:

1. **Parser services** — transform external formats (CIF, cXML, PDF) into OpenRegister objects.
2. **Classification services** — enrich existing objects (SpendTransaction → SpendCategory) using
   vector embeddings and LLM inference via `VectorizationService` and `ChatService`.
3. **Agent services** — orchestrate multi-step procurement workflows triggered by object state changes.
4. **Conversational API** — REST endpoints wrapping `ChatService` with RAG context from OpenRegister.
5. **Background jobs** — scheduled processing for demand anomaly detection and reorder triggers.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Vue Frontend                                                        │
│  SpendAnalystChat ─── SpendAnalystController ─── ChatService        │
│  CatalogValidationResultsDialog ─── AiCatalogController             │
│  OcrPurchaseOrderPreview ─── (file upload via FileService)          │
│  ProcurementAgentPanel ─── ProcurementAgentController               │
│  DemandAnomalyWidget ─── (OpenRegister ObjectService)               │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ REST
┌────────────────────────────────▼────────────────────────────────────┐
│  PHP Controllers (thin)                                              │
│  SpendAnalystController / AiCatalogController / AgentController      │
└─────┬────────────────────────────────────────────────────────────────┘
      │
┌─────▼────────────────────────────────────────────────────────────────┐
│  PHP Services (business logic)                                        │
│  AiCatalogValidationService  ──► ChatService (OpenRegister)          │
│  CifCxmlParserService        ──► ImportService (OpenRegister)        │
│  OcrPurchaseOrderService     ──► TextExtractionService + ChatService │
│  SpendClassificationService  ──► VectorizationService (OpenRegister) │
│  ProcurementAgentService     ──► ChatService + WorkflowEngineRegistry│
│  RequisitionToPurchaseOrder  ──► ObjectService (OpenRegister)        │
│  DemandAnomalyDetectionJob   ──► ObjectService + NotificationService │
└─────┬────────────────────────────────────────────────────────────────┘
      │
┌─────▼────────────────────────────────────────────────────────────────┐
│  OpenRegister Objects                                                 │
│  CatalogItem · ProcurementCatalog · CatalogItem                      │
│  PurchaseOrder · PurchaseRequisition · ProcurementOrder              │
│  SpendTransaction · SpendCategory · ProcurementCategory              │
│  Supplier · SupplierKPI · GoodsReceipt                               │
│  ApprovalChain · ApprovalRequest · ApprovalTask                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Entity Usage

All entities are pre-existing in Shillinq. This change only **reads and writes** them via
`ObjectService` — it does not introduce new schemas.

### Primary entities

| Entity | Usage in this change |
|--------|---------------------|
| `CatalogItem` | Source and target of CIF/cXML import; AI validation; smart recommendations |
| `ProcurementCatalog` | Container for published CatalogItem batches |
| `PurchaseOrder` | Target of OCR extraction; auto-generated from PurchaseRequisition |
| `PurchaseRequisition` | Source for AI Request Creation Agent; trigger for auto-PO generation |
| `SpendTransaction` | Input for spend classification and Spend Analyst queries |
| `SpendCategory` | Enrichment target for SpendTransaction classification |
| `ProcurementCategory` | Baseline for demand anomaly detection |
| `ApprovalChain` | Configuration for PO approval routing |
| `ApprovalRequest` | Trigger for automated PO generation on approval completion |
| `Supplier` | Matched during automated PO generation; sourced during catalog validation |
| `SupplierKPI` | Context for AI procurement performance management |
| `GoodsReceipt` | Matched to PurchaseOrder in spend performance reporting |

### Secondary entities (read-only context)

| Entity | Usage |
|--------|-------|
| `ProcurementOrder` | Historical data for demand detection baseline |
| `SupplierPerformanceReport` | Context for AI procurement strategy recommendations |
| `Budget` | Financial goal alignment for AI performance management |
| `Contract` | Preferred supplier context for auto-PO generation |

---

## Reuse Analysis

Per ADR-012, all available OpenRegister capabilities have been audited before building custom code.

| Capability needed | OpenRegister service used | Custom code? |
|-------------------|--------------------------|--------------|
| CatalogItem CRUD + list | `ObjectService` + `CnIndexPage` | No |
| PurchaseOrder CRUD + list | `ObjectService` + `CnIndexPage` | No |
| File upload for PDF PO | `FileService` + `CnFilesTab` | No |
| Text extraction from PDF | `TextExtractionService` | No (wrapped only) |
| AI inference / conversation | `ChatService` | No (wrapped only) |
| Vector embedding + semantic search | `VectorizationService` | No (wrapped only) |
| CIF/cXML file parsing logic | — | **Yes** — custom `CifCxmlParserService` |
| OCR field extraction orchestration | — | **Yes** — custom `OcrPurchaseOrderService` |
| Spend category classification logic | — | **Yes** — custom `SpendClassificationService` |
| Business rules: requisition → PO | — | **Yes** — custom `RequisitionToPurchaseOrderService` |
| Agent task orchestration | — | **Yes** — custom `ProcurementAgentService` |
| Demand anomaly detection | — | **Yes** — custom `DemandAnomalyDetectionJob` |
| Approval workflow routing | `WorkflowEngineRegistry` + `ApprovalChain` | No |
| Notifications | `NotificationService` | No |
| Bulk import with validation UI | `CnMassImportDialog` | No |
| Dashboard widgets / charts | `CnDashboardPage` + `CnChartWidget` | No |
| Chat UI framework | — | **Yes** — `SpendAnalystChat.vue` (thin wrapper) |

**No overlap found** with `ObjectService`, `RegisterService`, `SchemaService`, or
`ConfigurationService` for the custom-built components. The custom services implement
procurement-domain business logic that has no generic analogue in the platform.

---

## API Design

All endpoints follow ADR-002 (`/index.php/apps/shillinq/api/{resource}`).

### `POST /api/ai-catalog/validate`
Validate a batch of CatalogItems (pre-import).
- Request body: `{ "items": [...] }` (raw parsed CIF/cXML lines)
- Response: `{ "valid": [...], "invalid": [{ "line": N, "errors": [...], "suggestions": [...] }] }`

### `POST /api/ai-catalog/import-cif`
Upload and parse a CIF/cXML file. Internally calls `CifCxmlParserService` then
`AiCatalogValidationService`, then stages items via `ObjectService`.
- Request body: `multipart/form-data` — `file`, `catalogId`, `dryRun` (bool)
- Response: `{ "staged": N, "valid": N, "invalid": N, "jobId": "..." }`

### `POST /api/ai-catalog/ocr`
Trigger OCR extraction on an already-uploaded file attached to a PurchaseOrder.
- Request body: `{ "purchaseOrderId": "...", "fileId": "..." }`
- Response: `{ "extracted": { ...PurchaseOrder fields... }, "confidence": { ...per-field scores... } }`

### `POST /api/spend-analyst/conversation`
Send a message to the Spend Analyst chatbot.
- Request body: `{ "message": "...", "conversationId": "..." }`
- Response: `{ "reply": "...", "conversationId": "...", "sources": [...] }`

### `GET /api/spend-analyst/conversation/{id}`
Retrieve conversation history.

### `POST /api/procurement-agents/{agentSlug}/run`
Trigger a named procurement AI agent.
- Request body: `{ "context": { ...entity references... } }`
- Response: `{ "taskId": "...", "status": "queued" }`

### `GET /api/procurement-agents`
List available agents with status, last-run timestamp, and description.

### `POST /api/spend-classification/classify`
Classify one or more SpendTransaction records.
- Request body: `{ "transactionIds": [...] }`
- Response: `{ "classified": N, "skipped": N }`

---

## Component Design

### `SpendAnalystChat.vue`

Single-panel chat component embedded in the Shillinq dashboard.

```
┌────────────────────────────────────────────────┐
│ Spend Analyst                         [Wissen]  │
├────────────────────────────────────────────────┤
│ [bubble] Wat hebben we dit kwartaal            │
│          besteed aan ICT?                      │
│ [bubble] € 47.320 in 12 orders via 3          │
│          leveranciers. Top leverancier:        │
│          Coolblue Business (€ 22.100).         │
│          [Toon details]                        │
├────────────────────────────────────────────────┤
│ [_________________________] [Versturen]        │
└────────────────────────────────────────────────┘
```

- Uses `createObjectStore` for conversation state.
- Messages rendered with sanitised markdown.
- Source references link to OpenRegister objects.

### `CatalogValidationResultsDialog.vue`

Modal dialog showing import validation results.

```
┌──────────────────────────────────────────────────────┐
│ Catalogus validatieresultaten                   [×]  │
├──────────────────────────────────────────────────────┤
│ ✓ 234 items geldig      ✗ 12 items ongeldig          │
│ ─────────────────────────────────────────────────────│
│ Regel 45: Prijs ontbreekt — [Veld invullen]          │
│ Regel 67: Dubbel SKU "NL-STL-4872" — [Samenvoegen]  │
│ Regel 89: BTW-tarief onbekend "19%" — [Corrigeren]   │
├──────────────────────────────────────────────────────┤
│              [Annuleren]  [Geldige items importeren] │
└──────────────────────────────────────────────────────┘
```

### `OcrPurchaseOrderPreview.vue`

Inline preview panel shown when OCR extraction completes.

- Highlights extracted fields with colour-coded confidence (green ≥ 90%, amber 70-89%, red < 70%).
- Each field is editable inline; changes update the draft PurchaseOrder in real time.
- Low-confidence fields auto-focused on open for user correction.

### `ProcurementAgentPanel.vue`

List of available AI agents within a `CnDetailCard` on the Settings page.

- Each agent row: name, description, last run time, status badge (`CnStatusBadge`), run button.
- Agent run dispatches `POST /api/procurement-agents/{slug}/run` and polls status.

### `DemandAnomalyWidget.vue`

Dashboard widget (fits `CnWidgetWrapper`) showing spend deviation alerts.

- KPI: number of categories with > threshold deviation this period.
- List: top 5 deviating categories with delta % and trend arrow.
- Click → navigate to SpendTransaction list filtered by category.

---

## Seed Data

All seed objects use the `@self` envelope and are loaded via `ConfigurationService::importFromApp`.
Dutch realistic data; municipality context (`Gemeente Westerveen`, fictitious).

### CatalogItem (5 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "cat-item-bureaustoelen-001" },
    "name": "Bureaustoelen ergonomisch NEN 1813",
    "description": "Ergonomische bureaustoel conform NEN 1813, verstelbare rugleuning en armleuningen",
    "sku": "OF-BST-4820",
    "unitPrice": { "amount": 289.00, "currency": "EUR" },
    "unitOfMeasure": "stuk",
    "supplierReference": "BST-ERGO-PRO",
    "procurementCategory": "Kantoormeubilair",
    "vatRate": 21,
    "status": "active",
    "lastValidated": "2026-05-01T09:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "cat-item-printpapier-002" },
    "name": "Printpapier A4 80g/m² wit (500 vel)",
    "description": "Wit kopieerpapier A4 formaat, 80 g/m², 500 vel per pak",
    "sku": "OF-PAP-0080",
    "unitPrice": { "amount": 5.75, "currency": "EUR" },
    "unitOfMeasure": "pak",
    "supplierReference": "RYO-A4-80G",
    "procurementCategory": "Kantoorbenodigdheden",
    "vatRate": 21,
    "status": "active",
    "lastValidated": "2026-05-01T09:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "cat-item-laptop-dell-003" },
    "name": "Laptop Dell Latitude 5540 i5 16GB 512GB",
    "description": "Zakelijke laptop, Intel Core i5-1345U, 16GB DDR5, 512GB NVMe SSD, 15.6\" FHD",
    "sku": "IT-LAP-5540",
    "unitPrice": { "amount": 1149.00, "currency": "EUR" },
    "unitOfMeasure": "stuk",
    "supplierReference": "DELL-LAT5540-I5",
    "procurementCategory": "ICT-apparatuur",
    "vatRate": 21,
    "status": "active",
    "lastValidated": "2026-04-15T14:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "cat-item-koffie-004" },
    "name": "Koffie Arabica 500g bonen (Fairtrade)",
    "description": "100% Arabica koffiebonen, Fairtrade gecertificeerd, medium roast, 500 gram",
    "sku": "CAF-ARB-500G",
    "unitPrice": { "amount": 12.50, "currency": "EUR" },
    "unitOfMeasure": "zak",
    "supplierReference": "FT-ARB-MR-500",
    "procurementCategory": "Facilitaire diensten",
    "vatRate": 9,
    "status": "active",
    "lastValidated": "2026-05-10T11:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "CatalogItem", "slug": "cat-item-balpennen-005" },
    "name": "Balpennen blauw medium 50 stuks",
    "description": "Balpennen blauw, medium punt 1.0mm, kleur blauw, doos van 50 stuks",
    "sku": "OF-BAL-BL50",
    "unitPrice": { "amount": 8.95, "currency": "EUR" },
    "unitOfMeasure": "doos",
    "supplierReference": "BIC-RD-BL-50",
    "procurementCategory": "Kantoorbenodigdheden",
    "vatRate": 21,
    "status": "active",
    "lastValidated": "2026-05-01T09:00:00Z"
  }
]
```

### PurchaseOrder (4 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "PurchaseOrder", "slug": "po-2026-04-001" },
    "orderNumber": "PO-2026-04-001",
    "orderDate": "2026-04-03",
    "supplier": "Staples Nederland B.V.",
    "supplierKvk": "17111283",
    "deliveryAddress": "Raadhuisplein 1, 7811 AP Emmen",
    "status": "delivered",
    "totalAmount": { "amount": 347.25, "currency": "EUR" },
    "reference": "INT/FAC/2026/042",
    "approvedBy": "j.vandenberg@gemeentewesterveen.nl",
    "approvedAt": "2026-04-04T10:15:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseOrder", "slug": "po-2026-04-002" },
    "orderNumber": "PO-2026-04-002",
    "orderDate": "2026-04-08",
    "supplier": "Coolblue Business",
    "supplierKvk": "24449093",
    "deliveryAddress": "Raadhuisplein 1, 7811 AP Emmen",
    "status": "confirmed",
    "totalAmount": { "amount": 4596.00, "currency": "EUR" },
    "reference": "INT/ICT/2026/019",
    "approvedBy": "m.dekker@gemeentewesterveen.nl",
    "approvedAt": "2026-04-09T08:45:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseOrder", "slug": "po-2026-05-001" },
    "orderNumber": "PO-2026-05-001",
    "orderDate": "2026-05-02",
    "supplier": "ISS Facility Services B.V.",
    "supplierKvk": "22038991",
    "deliveryAddress": "Raadhuisplein 1, 7811 AP Emmen",
    "status": "draft",
    "totalAmount": { "amount": 1850.00, "currency": "EUR" },
    "reference": "INT/FAC/2026/053",
    "approvedBy": null,
    "approvedAt": null
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseOrder", "slug": "po-2026-05-002" },
    "orderNumber": "PO-2026-05-002",
    "orderDate": "2026-05-14",
    "supplier": "Lyreco Nederland B.V.",
    "supplierKvk": "62625943",
    "deliveryAddress": "Dorpsstraat 42, 8091 GH Wezep",
    "status": "pending_approval",
    "totalAmount": { "amount": 628.40, "currency": "EUR" },
    "reference": "INT/FAC/2026/061",
    "approvedBy": null,
    "approvedAt": null
  }
]
```

### PurchaseRequisition (3 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "pr-2026-04-017" },
    "requisitionNumber": "PR-2026-04-017",
    "requestedBy": "a.smit@gemeentewesterveen.nl",
    "requestDate": "2026-04-22",
    "description": "Vervanging laptop dienst vergunningen — toestel ouder dan 4 jaar",
    "estimatedAmount": { "amount": 1200.00, "currency": "EUR" },
    "procurementCategory": "ICT-apparatuur",
    "status": "approved",
    "approvedBy": "m.dekker@gemeentewesterveen.nl",
    "approvedAt": "2026-04-23T14:00:00Z",
    "linkedPurchaseOrder": "po-2026-04-002"
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "pr-2026-05-003" },
    "requisitionNumber": "PR-2026-05-003",
    "requestedBy": "b.vanderhoeven@gemeentewesterveen.nl",
    "requestDate": "2026-05-05",
    "description": "Kantoorbenodigdheden Q2 — papier, pennen, ordners",
    "estimatedAmount": { "amount": 350.00, "currency": "EUR" },
    "procurementCategory": "Kantoorbenodigdheden",
    "status": "pending_approval",
    "approvedBy": null,
    "approvedAt": null,
    "linkedPurchaseOrder": null
  },
  {
    "@self": { "register": "shillinq", "schema": "PurchaseRequisition", "slug": "pr-2026-05-009" },
    "requisitionNumber": "PR-2026-05-009",
    "requestedBy": "p.jansen@gemeentewesterveen.nl",
    "requestDate": "2026-05-18",
    "description": "Catering vergadering raad 27 mei — koffie, thee, water voor 40 personen",
    "estimatedAmount": { "amount": 180.00, "currency": "EUR" },
    "procurementCategory": "Facilitaire diensten",
    "status": "draft",
    "approvedBy": null,
    "approvedAt": null,
    "linkedPurchaseOrder": null
  }
]
```

### SpendTransaction (4 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "SpendTransaction", "slug": "st-2026-04-0042" },
    "transactionDate": "2026-04-10",
    "amount": { "amount": 347.25, "currency": "EUR" },
    "supplier": "Staples Nederland B.V.",
    "description": "Kantoorbenodigdheden week 15",
    "purchaseOrderReference": "po-2026-04-001",
    "spendCategory": "Kantoorbenodigdheden",
    "classifiedByAi": false,
    "classificationConfidence": null
  },
  {
    "@self": { "register": "shillinq", "schema": "SpendTransaction", "slug": "st-2026-04-0055" },
    "transactionDate": "2026-04-14",
    "amount": { "amount": 4596.00, "currency": "EUR" },
    "supplier": "Coolblue Business",
    "description": "Hardware inkoop IT afdeling",
    "purchaseOrderReference": "po-2026-04-002",
    "spendCategory": "ICT-apparatuur",
    "classifiedByAi": true,
    "classificationConfidence": 0.97
  },
  {
    "@self": { "register": "shillinq", "schema": "SpendTransaction", "slug": "st-2026-05-0011" },
    "transactionDate": "2026-05-06",
    "amount": { "amount": 212.40, "currency": "EUR" },
    "supplier": "Lyreco Nederland B.V.",
    "description": "Diverse kantoorartikelen",
    "purchaseOrderReference": null,
    "spendCategory": null,
    "classifiedByAi": false,
    "classificationConfidence": null
  },
  {
    "@self": { "register": "shillinq", "schema": "SpendTransaction", "slug": "st-2026-05-0028" },
    "transactionDate": "2026-05-12",
    "amount": { "amount": 890.00, "currency": "EUR" },
    "supplier": "Sodexo Netherlands B.V.",
    "description": "Cateringdiensten mei 2026",
    "purchaseOrderReference": null,
    "spendCategory": "Facilitaire diensten",
    "classifiedByAi": true,
    "classificationConfidence": 0.88
  }
]
```

---

## Security Considerations (ADR-005)

- **No PII in AI prompts**: SpendTransaction queries pass anonymised supplier codes and amounts;
  names resolved only for display after AI response is received.
- **Admin check backend**: agent trigger endpoints verify `IGroupManager::isAdmin()` or
  dedicated `shillinq:procurement-admin` group on the **server side**.
- **File validation**: CIF/cXML and PDF uploads validated for MIME type + max size (25 MB)
  before `TextExtractionService` or `CifCxmlParserService` processes them.
- **No stack traces**: all controllers return structured `{ "message": "..." }` errors only.
- **API rate limiting**: procurement agent endpoints carry a per-user cooldown (60 s) to
  prevent runaway LLM usage; enforced in `ProcurementAgentController`.

---

## Internationalisation (ADR-007)

All user-visible strings use `t(appName, 'key')`. Translation keys are added to:
- `l10n/nl.js` and `l10n/en.js`

Key translation categories:
- `ai.catalog.validation.*` — catalog validation messages and field labels
- `ai.po.ocr.*` — OCR preview labels and confidence indicators
- `ai.spend.analyst.*` — chat interface strings
- `ai.agent.*` — agent panel labels and status messages
- `ai.anomaly.*` — demand anomaly widget labels

Currency and date formatting defers to the user's Nextcloud locale setting.

---

## Testing Strategy (ADR-008)

| Layer | Approach |
|-------|----------|
| `CifCxmlParserService` | PHPUnit: valid CIF 3.0, valid cXML 1.2, malformed input, oversized file |
| `OcrPurchaseOrderService` | PHPUnit: mocked `TextExtractionService`, assert field mapping + confidence scoring |
| `SpendClassificationService` | PHPUnit: mocked `VectorizationService`, assert category assignment |
| `RequisitionToPurchaseOrderService` | PHPUnit: mock `ObjectService`, assert PO fields derived correctly |
| `DemandAnomalyDetectionJob` | PHPUnit: mock `ObjectService` + `NotificationService`, assert threshold logic |
| `SpendAnalystController` | Newman: conversation flow, empty message rejection, conversation ID persistence |
| `AiCatalogController` | Newman: dry-run import, full import with invalid rows, re-import idempotency |
| `SpendAnalystChat.vue` | Playwright: type query, assert reply rendered, assert source links present |
| `OcrPurchaseOrderPreview.vue` | Playwright: upload PDF, assert confidence colours, assert editable fields |
| `CatalogValidationResultsDialog.vue` | Playwright: import CIF file, assert invalid row count, assert approve flow |
