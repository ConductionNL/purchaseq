# Spec: Dashboards & KPIs

**Change:** catalog-purchase-management-analytics
**Capability:** dashboards, procurement-kpis, sustainability-analytics

---

## ADDED Requirements

### REQ-DSH-001: Custom Dashboard Creation with Drill-Down

Procurement managers must be able to configure a personal dashboard by selecting, arranging,
and resizing widgets that drill down into procurement data. The dashboard is built on
`CnDashboardPage` with GridStack drag-drop layout.

#### Scenario 1.1: Manager arranges dashboard widgets

- **GIVEN** a procurement manager opens the dashboard in edit mode
- **WHEN** they drag a widget (e.g. "Spend per categorie") to a new position
- **THEN** the layout is saved to the user's `UserPreference` object in OpenRegister
- **AND** on next login the dashboard is restored to the saved layout

#### Scenario 1.2: Manager adds a widget from the widget gallery

- **GIVEN** a manager is in dashboard edit mode
- **WHEN** they click "Widget toevoegen" and select "Top leveranciers"
- **THEN** the `CnTableWidget` for top suppliers is added to the dashboard grid
- **AND** the widget fetches data from `GET /api/analytics/spend-variance` or the
  relevant endpoint without a full page reload

#### Scenario 1.3: Drill-down from dashboard widget to detail

- **GIVEN** the "Spend per categorie" chart widget is on the dashboard
- **WHEN** the manager clicks a category segment in the chart
- **THEN** they are navigated to the `SpendAnalytics` index page pre-filtered on that category
- **AND** the URL reflects the applied filter (e.g. `?category=ICT-hardware`)

#### Scenario 1.4: Dashboard is accessible at 768px viewport (ADR-010)

- **GIVEN** a manager opens the dashboard on a tablet (768px width)
- **WHEN** the page renders
- **THEN** all widgets stack vertically and remain fully readable
- **AND** no content is clipped or horizontally scrollable

---

### REQ-DSH-002: Procurement KPI Tracking with Configurable Metrics

Procurement teams must be able to define, track, and display KPIs using `SupplierKPI` and
`SupplierPerformanceScore` objects. KPI cards are rendered via `CnKpiGrid`.

#### Scenario 2.1: KPI scorecard displays current scores

- **GIVEN** `SupplierKPI` objects exist for one or more suppliers
- **WHEN** the KPI scorecard page loads
- **THEN** `CnKpiGrid` renders each KPI with its name, current score, target, and trend indicator
- **AND** KPIs below target threshold are highlighted (using NL Design System token)

#### Scenario 2.2: Configuring a new KPI metric

- **GIVEN** a procurement manager opens the KPI settings
- **WHEN** they create a new `SupplierKPI` via `CnFormDialog`
- **THEN** the KPI is saved to OpenRegister with name, unit, target value, and associated supplier
- **AND** it appears immediately in the KPI scorecard

#### Scenario 2.3: KPI benchmarking view

- **GIVEN** multiple `SupplierPerformanceScore` objects exist for the same KPI across suppliers
- **WHEN** the benchmarking widget loads
- **THEN** a bar chart (`CnChartWidget`) compares supplier scores side by side for that KPI
- **AND** the best-performing supplier is visually identified

#### Scenario 2.4: KPI trend over time

- **GIVEN** a `SupplierPerformanceScore` has scores recorded for multiple periods
- **WHEN** a manager selects a KPI and views its trend
- **THEN** a line chart (`CnChartWidget`) shows the score over time
- **AND** the target value is rendered as a horizontal reference line on the chart

---

### REQ-DSH-003: Executive Procurement Dashboards with KPI Scorecards

Executives (directie, college van B&W) must be able to view a read-only, high-level
procurement dashboard with top-line KPIs, spend totals, and `SupplierPerformanceReport`
summaries. Access is restricted via `AuthorizationService` to the executive role.

#### Scenario 3.1: Executive dashboard shows top-line KPIs

- **GIVEN** an executive logs in and opens the Shillinq dashboard
- **WHEN** the executive dashboard loads
- **THEN** four `CnStatsBlock` cards display: total spend YTD, number of active POs,
  number of open delivery notes, and average supplier performance score
- **AND** each card shows the change vs. prior period (e.g. "+12% t.o.v. vorige maand")

#### Scenario 3.2: Executive cannot edit dashboard or KPI data

- **GIVEN** a user has the executive role (read-only)
- **WHEN** they view the dashboard
- **THEN** no edit controls, "Widget toevoegen", or "Bewerken" buttons are visible
- **AND** attempting to call `POST /api/analytics/*` as this user returns HTTP 403

#### Scenario 3.3: SupplierPerformanceReport linked from executive dashboard

- **GIVEN** a `SupplierPerformanceReport` object exists
- **WHEN** the executive clicks "Volledig leveranciersrapport"
- **THEN** they are navigated to the `ProcurementKpi` detail page showing the full report
- **AND** the report is rendered using `CnDetailPage` with `CnDetailCard` sections

---

### REQ-DSH-004: Sustainability Spend Tracking with CO2 Emission Data

Sustainability managers and procurement officers must be able to view CO2 emission totals
linked to procurement spend per `ProcurementCategory`. CO2 data is computed by
`SpendAggregationService` from the `co2EmissionFactor` on each category.

#### Scenario 4.1: Sustainability widget shows CO2 by category

- **GIVEN** `ProcurementCategory` objects have a `co2EmissionFactor` value set
- **WHEN** the sustainability widget on the dashboard loads
- **THEN** a donut chart (`CnChartWidget`) shows estimated CO2 (kg) per category
- **AND** the chart legend labels are in Dutch ("ICT-hardware", "Facilitaire diensten", etc.)

#### Scenario 4.2: CO2 total displayed as a KPI card

- **GIVEN** the sustainability dashboard is open
- **WHEN** spend transactions exist for the current year
- **THEN** a `CnStatsBlock` shows the total estimated CO2 for the year in kg
- **AND** a secondary line shows the CO2 target (if configured via `UserPreference`)

#### Scenario 4.3: CO2 endpoint returns structured data

- **GIVEN** the frontend calls `GET /api/analytics/co2`
- **WHEN** the request is authenticated and includes optional `?from=` and `?to=` params
- **THEN** the response is HTTP 200 with JSON: `[{ category, spend, co2Kg, co2Factor }]`
- **AND** the response contains no stack traces or internal paths (ADR-005)

#### Scenario 4.4: Categories without CO2 factor excluded from calculation

- **GIVEN** a `ProcurementCategory` has no `co2EmissionFactor` set (null or zero)
- **WHEN** the CO2 calculation runs
- **THEN** that category is excluded from the CO2 total
- **AND** a footnote on the widget indicates "N categorieën zonder CO2-factor"

---

### REQ-DSH-005: Procurement KPI Dashboards with Configurable Visualizations

Procurement analysts must be able to switch chart types (bar, line, pie, donut) for any
KPI widget on the dashboard. Visualization preferences are persisted per user.

#### Scenario 5.1: Manager switches chart type for a widget

- **GIVEN** a KPI widget is displayed on the dashboard
- **WHEN** the manager opens the widget settings and selects "Lijngrafiek"
- **THEN** the `CnChartWidget` re-renders with the line chart type
- **AND** the chosen chart type is saved to `UserPreference` and persists on next load

#### Scenario 5.2: Configurable metric selection per widget

- **GIVEN** a KPI widget supports multiple metric sources
- **WHEN** the manager opens widget configuration
- **THEN** a dropdown shows available metrics (e.g. Leveranciersscore, Levertijdnaleving,
  Factuurnauwkeurigheid)
- **AND** selecting a metric updates the widget data without a full page reload
