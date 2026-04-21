
## Executive Dashboard — Full Documentation

### 1. Short Written Description

The Executive Dashboard provides leadership at Aumet with a consolidated view of stock movement activity across all pharmacy tenants. It is designed to be read in under 60 seconds — surfacing anomalies and network-wide signals without requiring the user to investigate raw data. Everything fits on two pages with no scrolling required.

---

### 2. Dashboard Structure

**Two pages:**

- **Page 1 — Metrics** — six KPIs stacked vertically, each showing YTD, MTD, and Yesterday values with percentage change vs last year same period
- **Page 2 — Tables** — three summary tables: top products by volume, tenant summary, expiry risk

**Left panel — filters** applied across both pages simultaneously:

- Date range
- Tenant
- Product
- Movement direction

---

### 3. KPIs and Views

**Page 1 — Six metrics:**

|Metric|Topic|What it shows|
|---|---|---|
|Total orders|Order activity|Completed transfers across all tenants|
|Total quantity moved|Product movement|Units physically transferred|
|Movement volume|Quantity trends|Total movement events recorded|
|Expired lots|Expiry tracking|Lots past expiration date|
|Active tenants|Tenant visibility|Pharmacies with activity this period|
|% missing expiry|Expiry tracking|Movements with no expiration date recorded|

Each metric shows three columns — YTD, MTD, Yesterday — and one row below each value showing % change vs last year same period.

**Page 2 — Three tables:**

|Table|Columns|
|---|---|
|Top products by movement volume|Product name, orders, qty moved, % of total, volume bar|
|Tenant summary|Tenant, orders, qty moved, last activity, status|
|Expiry risk|Tenant, product, lot number, qty, expiry date, status|

---

### 4. Audience, Purpose, and Drill-down Design

**Audience** Leadership and regional directors at Aumet who oversee the full pharmacy network. They do not work with raw data — they need aggregated signals that tell them where to look, not what to do.

**Purpose** Provide a single consolidated view of stock movement activity across all tenants. The dashboard answers one question at a glance: is the network moving as expected? It surfaces anomalies — inactive tenants, expiry risk, unusual volume — so leadership knows where to focus attention without digging into operational detail.

**Drill-down design** Intentionally shallow. The executive dashboard is not an investigation tool — it is a signal layer. One level of drill-down only:

- Click a tenant → filter the entire dashboard to that tenant's summary
- Click a product → filter to that product across all tenants
- Click an expiry risk tile → see which tenants are affected

Deeper investigation is handed off to the Operational Dashboard. The executive dashboard should never require more than two clicks to get an answer.

**What it does not do:**

- No row-level data
- No lot or batch detail
- No movement-by-movement breakdown
- No interpretation of trends — numbers only, decisions belong to the user

---

### 5. Filters and Dimensions

|Filter|Values|
|---|---|
|Date range|Last 7 days, Last 30 days, Last 90 days, Full year|
|Tenant|All tenants or specific pharmacy client|
|Product|All products or specific product|
|Movement direction|All, Inbound, Outbound, Internal|

All filters apply simultaneously across both pages.

## Operational Dashboard 

### 1. Short Written Description

The Operational Dashboard is the day-to-day working tool for pharmacy operations teams at Aumet. Unlike the Executive Dashboard which shows aggregated signals, this dashboard operates at row level — every individual movement, every specific lot, every expiry risk. It is designed around three workflows: triage alerts first thing in the morning, investigate specific movements when something looks wrong, and plan ahead for expiry risk before it becomes a financial loss. The amber accent color deliberately distinguishes it from the executive dashboard so users always know which context they are in.

---

### 2. Dashboard Structure

**Three pages:**

- **Page 1 — Alerts** — the triage page. Four KPIs at the top showing critical count, warning count, movements today, inactive tenants. Alert queue below ordered by severity with actionable descriptions.
- **Page 2 — Movements** — the investigation page. Row level table of every stock movement, searchable by order ID. Filterable by tenant, product, direction, location.
- **Page 3 — Lot management** — the planning and traceability page. Two sections: expiry risk forecast at the top with 30/60/90 day windows, lot traceability table at the bottom for reactive lookup.

**Left panel — filters** grouped by category and applied across all pages:

- Time — date range
- Tenant — pharmacy client
- Product
- Order — order ID text search
- Movement — direction, location
- Lot / Expiry — lot number text search, expiry status

---

### 3. KPIs and Views

**Page 1 — Four KPIs:**

|KPI|Color|Meaning|
|---|---|---|
|Critical alerts|Red|Issues requiring immediate action|
|Warnings|Amber|Issues to review today|
|Movements today|White|Total completed movements this period|
|Inactive tenants|Amber|Tenants with no activity in 25+ hours|

**Alert queue — severity levels:**

|Severity|Color|Examples|
|---|---|---|
|Critical|Red|Expired lots, pipeline failures, row count mismatches|
|Warning|Amber|Lots expiring soon, missing expiry above threshold, negative quantities|
|Resolved|Green|Successful batch loads, cleared issues|

**Page 2 — Movement detail table columns:**

Order ID, Tenant, Product, Direction, Quantity, From location, To location, Date, Status

**Page 3 — Expiry risk forecast KPIs:**

|KPI|Meaning|
|---|---|
|Lots at risk|Number of lots expiring within selected window|
|Total qty at risk|Total units across all at-risk lots|
|Sales same period LY|Historical sales baseline — same period last year|
|Risk gap|Qty at risk minus expected sales — positive means likely unsold|

**Page 3 — Expiry risk table columns:**

Product, Tenant, Lot, Qty at risk, Sales LY, Expected sales, Risk gap, Window, Action

**Page 3 — Lot traceability table columns:**

Lot number, Tenant, Product, Qty, Location, Expiry date, Days left, Status

---

### 4. Audience, Purpose, and Drill-down Design

**Audience** Operations teams, warehouse managers, and pharmacy data analysts at Aumet who work with the data daily. They need to act on specific records — not read summaries.

**Purpose** Surface what needs action today and enable fast investigation. The dashboard answers three questions across its three pages:

- What broke or needs urgent attention right now? — Page 1
- What exactly moved, where, and when? — Page 2
- Which lots will expire before we can sell them, and where are they now? — Page 3

**Drill-down design** Three levels across the three pages — each page goes deeper than the last:

- Page 1 — Alert fires → read the description and know what to do or where to look
- Page 2 → filter by tenant + order ID to find the specific movement that triggered the alert
- Page 3 → filter by lot number to find the exact batch, its location, and its expiry status

The sidebar lot number and order ID text inputs are the primary drill-down mechanism — type any value and all three pages filter simultaneously.

**What it does not do:**

- No YTD or MTD aggregations — that belongs to the executive dashboard
- No trend charts — the ops team needs rows not lines
- No cross-dashboard navigation — each dashboard stands alone by design

---

## Metrics Glossary — Both Dashboards Combined

|Metric|Dashboard|Definition|
|---|---|---|
|Total orders|Executive|Count of completed stock transfer records (`stock_picking` where `state = done`) in the selected period|
|Total quantity moved|Executive|Sum of `qty_done` across all completed `stock_move_line` records|
|Movement volume|Executive|Count of individual `stock_move_line` rows — more granular than order count|
|Expired lots|Executive + Ops|Count of `stock_lot` records where `expiration_date` is in the past|
|Active tenants|Executive|Count of distinct `tenant_id` values with at least one completed movement in the period|
|% missing expiry|Executive + Ops|Percentage of `stock_move_line` rows where `lot_id` is NULL or `expiration_date` is NULL|
|YTD|Executive|Year to date — from January 1st of current year to today|
|MTD|Executive|Month to date — from the 1st of current month to today|
|% vs LY|Executive|Percentage change vs same period last year|
|Critical alerts|Ops|Count of validation failures with `severity = critical` in `etl_config.validation_log`|
|Warnings|Ops|Count of validation results with `severity = warn`|
|Movements today|Ops|Count of `stock_move_line` rows with `write_date` = today|
|Inactive tenants|Ops|Count of active tenants with no movements in the last 25 hours|
|Lots at risk|Ops|Count of `stock_lot` records with `expiration_date` falling within the selected window|
|Total qty at risk|Ops|Sum of quantity across all lots in the expiry risk window|
|Sales same period LY|Ops|Historical outbound quantity for the same product in the same date window last year — used as expected sales baseline|
|Expected sales|Ops|Projected units to be sold before expiry based on last year same period velocity|
|Risk gap|Ops|Qty at risk minus expected sales. Positive value = units likely unsold before expiry. Negative = on track.|
|Days left|Ops|Calendar days remaining until `expiration_date`. Negative = already expired.|
|Movement direction|Both|Derived field — inbound (supplier to pharmacy), outbound (pharmacy to customer), internal (location to location)|