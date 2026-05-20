# Spec: Spend Analytics

**Change:** catalog-purchase-management-analytics
**Capability:** spend-analytics

---

## ADDED Requirements

### REQ-SPA-001: Real-Time Expense Tracking Dashboard

Procurement managers must be able to view all procurement spend in real time through a
centralised dashboard. The dashboard consumes `SpendTransaction` objects from OpenRegister
and displays totals, trends, and breakdowns without requiring manual exports.

#### Scenario 1.1: Dashboard loads spend summary on open

- **GIVEN** a procurement manager opens the Shillinq dashboard
- **WHEN** the dashboard initialises
- **THEN** total spend for the current fiscal year is displayed in a `CnStatsBlock`
- **AND** spend is broken down by `ProcurementCategory` in a `CnChartWidget` (bar chart)
- **AND** the data reflects all `SpendTransaction` objects with `status` = "Goedgekeurd" or "Betaald"

#### Scenario 1.2: Dashboard filters by date range

- **GIVEN** a procurement manager is viewing the spend dashboard
- **WHEN** they select a custom date range using the date filter
- **THEN** all spend widgets update to show only `SpendTransaction` records within that range
- **AND** the applied filter is displayed visually (e.g. "1 jan 2026 – 31 mrt 2026")

#### Scenario 1.3: Drill-down into a category

- **GIVEN** a spend breakdown chart is displayed by `ProcurementCategory`
- **WHEN** the manager clicks on a category segment
- **THEN** a filtered list of `SpendTransaction` records for that category is shown
- **AND** the list is rendered using `CnDataTable` with sortable columns (date, amount, supplier, status)

#### Scenario 1.4: Export spend data

- **GIVEN** a procurement manager is viewing spend data
- **WHEN** they click the export button
- **THEN** `CnMassExportDialog` opens with format options (CSV, Excel, JSON)
- **AND** the exported file contains all `SpendTransaction` fields visible in the current filter

---

### REQ-SPA-002: Estimated vs. Actual Spend Tracking

Finance and procurement managers must be able to compare budgeted (estimated) spend against
actual spend per `ProcurementCategory`. Variance is computed server-side by
`SpendAggregationService` and exposed via `GET /api/analytics/spend-variance`.

#### Scenario 2.1: Spend variance widget displays estimated vs. actual

- **GIVEN** a manager opens the procurement analytics dashboard
- **WHEN** the `SpendVarianceWidget` loads
- **THEN** a bar chart shows estimated spend (from `Budget`/`BudgetAllocation`) and actual spend
  (from `SpendTransaction`) side by side per `ProcurementCategory`
- **AND** categories where actual exceeds estimated are highlighted in a distinct colour
  (using NL Design System token, not a hardcoded hex value)

#### Scenario 2.2: Variance percentage is shown per category

- **GIVEN** the spend variance chart is displayed
- **WHEN** the manager hovers over a category bar
- **THEN** a tooltip shows: estimated amount, actual amount, variance amount, and variance %
- **AND** values are formatted according to the user's locale (Dutch: "€ 1.240,50")

#### Scenario 2.3: No spend data for a category

- **GIVEN** a `ProcurementCategory` exists with a budget allocation but zero spend transactions
- **WHEN** the spend variance widget loads
- **THEN** the category is displayed with estimated spend and 0 actual spend
- **AND** no error or empty-state hides the category from the chart

#### Scenario 2.4: Spend variance endpoint returns structured response

- **GIVEN** the frontend calls `GET /api/analytics/spend-variance`
- **WHEN** the request is authenticated and includes optional `?from=` and `?to=` query params
- **THEN** the response is HTTP 200 with JSON array: `[{ category, estimated, actual, variance, variancePct }]`
- **AND** no stack traces, SQL, or internal paths appear in the response (ADR-005)

---

### REQ-SPA-003: Service Procurement with SOW-Based PO Creation and Milestone Tracking

Procurement officers must be able to create a `PurchaseOrder` linked to a
`StatementOfWork`, and track milestone completion through the procurement dashboard.

#### Scenario 3.1: Create PO linked to an existing SOW

- **GIVEN** a procurement officer opens a `StatementOfWork` detail page
- **WHEN** they click "Inkooporder aanmaken"
- **THEN** a `CnFormDialog` opens pre-populated with SOW reference, supplier, value, and date range
- **AND** on save, a `PurchaseOrder` object is created in OpenRegister with a relation to the `StatementOfWork`

#### Scenario 3.2: Milestone progress displayed on dashboard

- **GIVEN** a `PurchaseOrder` is linked to a `StatementOfWork` with `ContractMilestone` objects
- **WHEN** the procurement dashboard loads the milestone widget
- **THEN** `CnTimelineStages` renders each `ContractMilestone` with its status (Gepland / In uitvoering / Afgerond)
- **AND** overdue milestones (due date < today, status ≠ Afgerond) are highlighted

#### Scenario 3.3: Milestone status update

- **GIVEN** a milestone is displayed in `CnTimelineStages`
- **WHEN** a procurement officer changes its status to "Afgerond"
- **THEN** the `ContractMilestone` object is updated in OpenRegister via `ObjectService.saveObject()`
- **AND** the dashboard milestone widget reflects the updated status immediately

---

### REQ-SPA-004: Strategic Procurement Analytics

Procurement managers must be able to access strategic insights — top suppliers by spend,
category concentration, savings opportunities — derived from existing `SpendTransaction`,
`SupplierKPI`, and `SavingsOpportunity` objects.

#### Scenario 4.1: Top suppliers by spend

- **GIVEN** a manager opens the strategic analytics view
- **WHEN** the top-suppliers widget loads
- **THEN** a ranked list of suppliers is displayed (supplier name, total spend, % of total)
  using `CnTableWidget`
- **AND** the list is limited to the top 10 by spend for the selected period

#### Scenario 4.2: Category concentration warning

- **GIVEN** a single `ProcurementCategory` accounts for more than 60% of total spend
- **WHEN** the strategic analytics dashboard loads
- **THEN** a `CnStatsBlock` highlights the concentration risk with a warning indicator
- **AND** the warning is accessible (not colour-only — includes an icon and text label, ADR-010)

#### Scenario 4.3: Savings opportunities surfaced

- **GIVEN** `SavingsOpportunity` objects exist in OpenRegister
- **WHEN** the strategic analytics widget loads
- **THEN** open savings opportunities are listed with estimated savings amount and category
- **AND** clicking an opportunity navigates to its detail page
