
---
## Aumet Pharmacy Network — Reporting Layer

---

## 1. Purpose

This document defines the KPI framework for the Aumet multi-tenant pharmacy reporting layer. It establishes which metrics exist, what they mean, how they are calculated, and who they are intended for. It is the authoritative reference for any business stakeholder who needs to understand what a reported number represents.

All monetary values in this framework are reported in **USD**. Currency conversion from local pharmacy currencies — SAR, OMR, JOD, EGP — is applied at ETL extraction time using the daily exchange rate at the transaction date. Original local currency values are preserved for audit purposes.

---

## 2. Framework Design Principles

Five principles govern this framework:

- **One definition per KPI** — every metric has exactly one agreed definition. Ambiguity is resolved before a KPI is published, not after.
- **One owner per KPI** — every KPI has a named business owner responsible for its definition. No ownerless metrics.
- **Source traceability** — every KPI must trace back to a defined field in the ClickHouse warehouse. No KPI without a data source.
- **Audience separation** — executive KPIs are signals. Operational KPIs are tools. They are never mixed on the same reporting surface.
- **Overload prevention** — the executive layer is capped at 8 KPIs. Any proposed addition must either replace an existing KPI or move to the operational layer.

---

## 3. KPI Layers

### 3.1 Executive KPIs — Signal Layer

Designed for leadership and regional directors. Aggregated, trend-focused, action-free. A leader reads these and decides where to focus attention — they do not investigate at this level.

|#|KPI|Category|Frequency|
|---|---|---|---|
|1|Total orders|Activity volume|Weekly / Monthly|
|2|Total quantity moved|Quantity movement|Weekly / Monthly|
|3|Average order value (USD)|Financial performance|Weekly / Monthly|
|4|Movement volume|Activity volume|Weekly / Monthly|
|5|Expired lots|Expiration monitoring|Weekly / Monthly|
|6|Active tenants|Tenant visibility|Weekly / Monthly|
|7|% missing expiry|Expiration monitoring|Weekly / Monthly|
|8|Cash vs insurance split|Payment method|Weekly / Monthly|

### 3.2 Operational KPIs — Supporting Metrics Layer

Designed for operations teams and analysts. Row-level, actionable, used for daily monitoring and investigation.

**Activity and volume**

- Orders per tenant per day
- Inbound vs outbound vs internal split
- Average movements per order

**Financial**

- Average order value by tenant (USD)
- Average order value by product category (USD)
- Average order value by payment method — cash vs insurance (USD)

**Expiry and lot**

- Lots expiring within 30 / 60 / 90 days
- Risk gap — quantity at risk minus expected sales
- % movements without lot tracking
- Average days to expiry per product category

**Tenant health**

- Tenants with zero activity in last 25 hours
- % missing expiry per tenant
- Row count accuracy per batch per tenant

**Product**

- Top 10 products by movement volume
- Top 10 products by order value (USD)
- Products with highest expiry risk gap

**Payment method**

- Cash order count vs insurance order count
- Cash vs insurance order value (USD)
- Average order value — cash vs insurance (USD)

---

## 4. KPI Definitions — Core Metrics

### KPI 1 — Total Orders

|Field|Definition|
|---|---|
|KPI name|Total orders|
|Business definition|The count of completed stock transfer records in the selected period across all tenants|
|Calculation logic|`COUNT(DISTINCT order_id) WHERE order_status = 'done'`|
|Audience|Executive leadership|
|Reporting frequency|Weekly and monthly|
|Dimensions|Tenant, date range, movement direction|
|Why it matters|Baseline activity signal — tells leadership whether the pharmacy network is operating at expected volume|
|NULL handling|Orders with NULL order_id are excluded and counted separately in the data quality log|

---

### KPI 2 — Total Quantity Moved

|Field|Definition|
|---|---|
|KPI name|Total quantity moved|
|Business definition|The sum of all units physically transferred across all completed movements in the selected period|
|Calculation logic|`SUM(quantity) WHERE order_status = 'done' AND is_quantity_negative = FALSE`|
|Audience|Executive leadership|
|Reporting frequency|Weekly and monthly|
|Dimensions|Tenant, product, date range, movement direction|
|Why it matters|Measures physical stock activity — distinguishes between order count and actual volume moved|
|NULL handling|NULL quantities defaulted to 0 at transformation. Negative quantities corrected to absolute value and flagged separately|

---

### KPI 3 — Average Order Value (USD)

|Field|Definition|
|---|---|
|KPI name|Average order value (USD)|
|Business definition|The average monetary value per completed order, expressed in USD|
|Calculation logic|`SUM(order_value_usd) / COUNT(DISTINCT order_id) WHERE order_status = 'done'`|
|Audience|Executive leadership|
|Reporting frequency|Weekly and monthly|
|Dimensions|Tenant, payment method, product category, date range|
|Why it matters|Measures revenue density per order — helps identify high-value tenants and product categories|
|Currency note|Converted from local currency at ETL time using daily exchange rate. Original local currency value preserved in `local_currency_amount` field|
|NULL handling|Orders where currency conversion failed are excluded and flagged with `currency_conversion_flag = failed`|
|Source dependency|Requires `account.move` or `sale.order` pipeline — separate from stock movement pipeline|

---

### KPI 4 — Movement Volume

|Field|Definition|
|---|---|
|KPI name|Movement volume|
|Business definition|The count of individual stock movement line records in the selected period — more granular than order count as one order can contain multiple movement lines|
|Calculation logic|`COUNT(source_line_id) WHERE order_status = 'done'`|
|Audience|Executive leadership|
|Reporting frequency|Weekly and monthly|
|Dimensions|Tenant, product, direction, date range|
|Why it matters|Distinguishes between order frequency and operational intensity — a single order can generate many movements|
|NULL handling|NULL source_line_id rows are excluded|

---

### KPI 5 — Expired Lots

|Field|Definition|
|---|---|
|KPI name|Expired lots|
|Business definition|The count of distinct lot numbers where the expiration date has passed as of the reporting date|
|Calculation logic|`COUNT(DISTINCT lot_number) WHERE expiration_date < today() AND expiration_date IS NOT NULL`|
|Audience|Executive leadership — alert signal. Operational teams — action|
|Reporting frequency|Daily for operational teams. Weekly for leadership|
|Dimensions|Tenant, product, location|
|Why it matters|Critical compliance metric for pharmacy — expired medicine on shelves is a regulatory and patient safety risk|
|NULL handling|Lots with NULL expiration_date are excluded from this count and reported separately under % missing expiry|

---

### KPI 6 — Active Tenants

|Field|Definition|
|---|---|
|KPI name|Active tenants|
|Business definition|The count of distinct pharmacy clients with at least one completed stock movement in the selected period|
|Calculation logic|`COUNT(DISTINCT tenant_id) WHERE order_status = 'done' AND create_date >= period_start`|
|Audience|Executive leadership|
|Reporting frequency|Weekly and monthly|
|Dimensions|Date range|
|Why it matters|Measures network participation — a declining active tenant count is an early signal of client disengagement or pipeline failure|
|NULL handling|tenant_id is never NULL — enforced at extraction. Rows without tenant_id are rejected|

---

### KPI 7 — % Missing Expiry

|Field|Definition|
|---|---|
|KPI name|% missing expiry|
|Business definition|The percentage of stock movement lines where no expiration date could be associated — either because the product has no lot tracking or because the lot has no expiry date recorded in Odoo|
|Calculation logic|`SUM(is_expiration_missing) / COUNT(*) * 100`|
|Audience|Executive leadership — trend signal. Operational teams — investigation|
|Reporting frequency|Daily for operational teams. Weekly for leadership|
|Dimensions|Tenant, product|
|Why it matters|High missing expiry rate indicates a data quality problem in the source system — products are being moved without proper lot tracking which creates compliance risk|
|Threshold|Alert triggered when % exceeds 7% for any tenant|
|NULL handling|is_expiration_missing is a derived boolean — always populated at transformation|

---

### KPI 8 — Cash vs Insurance Split

|Field|Definition|
|---|---|
|KPI name|Cash vs insurance order split|
|Business definition|The percentage breakdown of completed orders by payment method — cash, insurance, mixed, or unknown|
|Calculation logic|`COUNT(order_id) GROUP BY payment_method / COUNT(DISTINCT order_id) * 100`|
|Audience|Executive leadership|
|Reporting frequency|Weekly and monthly|
|Dimensions|Tenant, product category, date range|
|Why it matters|Payment method mix directly affects order value, revenue predictability, and operational complexity. Insurance-heavy pharmacies have different risk profiles from cash-heavy ones|
|Source dependency|Requires `account.move` or `sale.order` pipeline. If unavailable payment_method defaults to `unknown`|
|NULL handling|Unknown payment method is reported as a separate category — never excluded or defaulted to cash or insurance|

---

### KPI 9 — Risk Gap

|Field|Definition|
|---|---|
|KPI name|Risk gap|
|Business definition|The difference between the quantity of stock at risk of expiring within a defined window and the expected sales volume during that same window based on last year same period performance|
|Calculation logic|`SUM(quantity_at_risk) - SUM(expected_sales)` per product per tenant per expiry window|
|Audience|Operational teams|
|Reporting frequency|Daily|
|Dimensions|Tenant, product, lot, expiry window — 30 / 60 / 90 days|
|Why it matters|Identifies which specific lots are likely to expire unsold — enables proactive action such as promotions, transfers, or write-offs before the stock becomes a loss|
|Positive value|Units likely unsold before expiry — action required|
|Negative value|Expected sales exceed quantity at risk — on track|
|Source dependency|Expected sales derived from historical outbound quantity same period last year|

---

## 5. Currency Framework

|Item|Definition|
|---|---|
|Reporting currency|USD — all monetary KPIs|
|Conversion timing|At ETL extraction time — daily rate at transaction date|
|Rate source|To be defined by data owner — suggested: ECB daily reference rate or equivalent|
|Local currency preservation|`local_currency_amount` and `local_currency_code` fields retained on every row|
|Conversion failure handling|Row flagged `currency_conversion_flag = failed` — excluded from monetary aggregations — reported in data quality log|
|Supported currencies|SAR (Saudi Arabia), OMR (Oman), JOD (Jordan), EGP (Egypt)|

---

## 6. Payment Method Framework

|Value|Definition|
|---|---|
|`cash`|Order paid directly by the pharmacy or customer without insurance involvement|
|`insurance`|Order fully covered by an insurance provider — payment confirmed by claim|
|`mixed`|Order partially covered by insurance with a cash co-payment|
|`unknown`|Payment method not available from source system — reported as a distinct category|

Source: Odoo `account.move` or `sale.order` — requires a separate ETL pipeline from stock movements. This pipeline is flagged as a dependency for KPIs 3 and 8.

---

## 7. KPI Overload Prevention

|Rule|Detail|
|---|---|
|Executive cap|Maximum 8 KPIs on executive dashboard — enforced by governance owner|
|Addition rule|Any new KPI must either replace an existing one or be assigned to the operational layer|
|Retirement rule|KPIs not accessed in 90 days are flagged for review and potential retirement|
|Dimension rule|Payment method, currency, and tenant are dimensions — not separate KPIs — so they add analytical depth without adding KPI count|
|New tenant rule|New tenants inherit the standard KPI set — no new KPIs are created per tenant|

---

