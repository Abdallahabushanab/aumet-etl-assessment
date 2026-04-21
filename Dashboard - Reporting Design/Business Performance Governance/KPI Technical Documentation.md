

---

## 1. Purpose

This document is the engineer-facing reference for the Aumet KPI reporting layer. It tells a new data engineer everything they need to understand how every reported number is built — where it comes from, how it is calculated, what the rules are, and what has changed over time. It does not repeat what is in Document 1 or Document 2 — it references them and adds the technical layer on top.

**Read first:**

- Document 1 — KPI Framework and Definitions — for business context
- Document 2 — Governance and Ownership Model — for process context
- Assessment 1 — ETL Design Document — for full pipeline architecture, source table mapping, extraction SQL, transformation logic, and data dictionary

---

## 2. Architecture Reference

The full technical architecture is documented in Assessment 1. Key references:

|Topic|Assessment 1 section|
|---|---|
|Source tables and join relationships|Part 1 — Source Discovery|
|Extraction SQL|Part 1 section 1.4|
|Staging and fact table DDL|Part 3 and Part 4|
|Transformation rules and cleaning logic|Part 3|
|ClickHouse partitioning and ordering|Part 4|
|Validation checks and quality rules|Part 5|
|Schema drift and monitoring|Part 6|
|Data dictionary — all fact table columns|Part 7 section 7.3|
|Source-to-target field mapping|Part 1 + Part 3 combined|

---

## 3. KPI-to-Source Mapping

How each KPI traces from the ClickHouse fact table back to the Odoo PostgreSQL source:

|KPI|ClickHouse field|Source table|Source field|
|---|---|---|---|
|Total orders|`order_id`|`stock_picking`|`name`|
|Total quantity moved|`quantity`|`stock_move_line`|`qty_done`|
|Average order value|`order_value_usd`|`account.move` / `sale.order`|`amount_total` + currency conversion|
|Movement volume|`source_line_id`|`stock_move_line`|`id`|
|Expired lots|`expiration_date`, `lot_number`|`stock_lot`|`expiration_date`, `name`|
|Active tenants|`tenant_id`|Injected at ETL|—|
|% missing expiry|`is_expiration_missing`|Derived at transformation|—|
|Cash vs insurance split|`payment_method`|`account.move` / `sale.order`|`journal_id` / payment type|
|Risk gap|`quantity`, `expiration_date`|`stock_move_line`, `stock_lot`|`qty_done`, `expiration_date`|
|Movement direction|`movement_direction`|Derived at transformation|`stock_location.usage`|

---

## 4. Derived Field Calculations

Fields that do not exist in Odoo and are created during transformation:

**`is_expiration_missing`**

sql

```sql
CASE WHEN expiration_date IS NULL THEN TRUE ELSE FALSE END
```

**`is_quantity_negative`**

sql

```sql
CASE WHEN qty_done < 0 THEN TRUE ELSE FALSE END
```

**`movement_direction`**

sql

```sql
CASE
    WHEN src_location_usage = 'supplier'
     AND dst_location_usage = 'internal'  THEN 'inbound'
    WHEN src_location_usage = 'internal'
     AND dst_location_usage = 'customer'  THEN 'outbound'
    WHEN src_location_usage = 'internal'
     AND dst_location_usage = 'internal'  THEN 'internal'
    ELSE 'other'
END
```

**`order_value_usd`**

sql

```sql
amount_total_local * daily_exchange_rate_to_usd
-- daily_exchange_rate sourced from etl_config.exchange_rates
-- if rate not found: order_value_usd = NULL
--                    currency_conversion_flag = 'failed'
```

**`payment_method`**

sql

```sql
CASE
    WHEN journal_type = 'cash'      THEN 'cash'
    WHEN journal_type = 'insurance' THEN 'insurance'
    WHEN journal_type = 'mixed'     THEN 'mixed'
    ELSE 'unknown'
END
```

**`expected_sales`**

sql

```sql
-- Outbound quantity for same product, same tenant
-- same calendar window, previous year
SELECT SUM(quantity)
FROM   reporting.fact_stock_movements
WHERE  tenant_id          = :tenant_id
AND    product_id          = :product_id
AND    movement_direction  = 'outbound'
AND    create_date BETWEEN :window_start_ly AND :window_end_ly
```

**`risk_gap`**

sql

```sql
quantity_at_risk - expected_sales
-- positive = units likely unsold before expiry
-- negative = on track
-- NULL if expected_sales cannot be calculated
```

---

## 5. Supporting Control Tables

Three control tables support the KPI layer beyond the main fact table. All live in `etl_config` schema:

**Exchange rates table**

sql

```sql
CREATE TABLE etl_config.exchange_rates (
    rate_date        DATE,
    from_currency    VARCHAR,   -- SAR / OMR / JOD / EGP
    to_currency      VARCHAR,   -- always USD
    exchange_rate    DECIMAL,
    source           VARCHAR,   -- rate provider name
    created_at       TIMESTAMP,
    PRIMARY KEY (rate_date, from_currency)
);
```

**Tenant exceptions table**

sql

```sql
CREATE TABLE etl_config.tenant_exceptions (
    tenant_id        VARCHAR,
    exception_type   VARCHAR,   -- 'known_characteristic' / 'currency_gap' / 'custom_mapping'
    kpi_affected     VARCHAR,
    description      TEXT,
    approved_by      VARCHAR,
    approved_at      TIMESTAMP,
    review_date      DATE,
    PRIMARY KEY (tenant_id, kpi_affected)
);
```

**Payment method mapping table**

sql

```sql
CREATE TABLE etl_config.payment_method_mapping (
    tenant_id        VARCHAR,
    source_value     VARCHAR,   -- raw value from Odoo journal
    mapped_value     VARCHAR,   -- cash / insurance / mixed / unknown
    approved_by      VARCHAR,
    created_at       TIMESTAMP,
    PRIMARY KEY (tenant_id, source_value)
);
```

---

## 6. KPI Change Log

Every change to a KPI definition, calculation, threshold, or retirement must be logged here before deployment:

sql

```sql
CREATE TABLE governance.kpi_change_log (
    change_id             VARCHAR     PRIMARY KEY,
    kpi_name              VARCHAR,
    change_type           VARCHAR,    -- 'definition' / 'calculation' / 'threshold' / 'retirement'
    previous_value        TEXT,
    new_value             TEXT,
    reason                TEXT,
    requested_by          VARCHAR,
    approved_by_business  VARCHAR,
    approved_by_data      VARCHAR,
    effective_date        DATE,
    created_at            TIMESTAMP
);
```

**Example entries:**

|change_id|kpi_name|change_type|previous_value|new_value|reason|effective_date|
|---|---|---|---|---|---|---|
|CHG-001|% missing expiry|threshold|10%|7%|Business owner requested tighter threshold after compliance review|2024-04-01|
|CHG-002|Average order value|calculation|SAR only|USD via exchange rate|Multi-country expansion — standardise to USD|2024-04-15|
|CHG-003|Cash vs insurance split|definition|Not defined|Added payment method dimension|New requirement from finance team|2024-05-01|

---

## 7. KPI Ownership Matrix

Full ownership table — business owner responsible for definition, data owner responsible for implementation:

|KPI|Business owner|Data owner|Last reviewed|
|---|---|---|---|
|Total orders|Head of Operations|Data Engineering Lead|2024-04-01|
|Total quantity moved|Head of Operations|Data Engineering Lead|2024-04-01|
|Average order value (USD)|Head of Finance|Data Engineering Lead|2024-04-01|
|Movement volume|Head of Operations|Data Engineering Lead|2024-04-01|
|Expired lots|Head of Compliance|Data Engineering Lead|2024-04-01|
|Active tenants|Head of Product|Data Engineering Lead|2024-04-01|
|% missing expiry|Head of Compliance|Data Engineering Lead|2024-04-01|
|Cash vs insurance split|Head of Finance|Data Engineering Lead|2024-04-01|
|Risk gap|Head of Operations|Data Engineering Lead|2024-04-01|
|Currency conversion|Head of Finance|Data Engineering Lead|2024-04-01|
|Payment method mapping|Head of Finance|Data Engineering Lead|2024-04-01|

---

## 8. New Engineer Onboarding Checklist

A new data engineer joining the team should complete these steps in order before touching any KPI:

- Read Assessment 1 ETL Design Document — understand the full pipeline
- Read Document 1 — understand what each KPI means to the business
- Read Document 2 — understand the governance process and who to contact
- Read this document — understand how KPIs are calculated technically
- Review the change log — understand what has changed and why
- Review the tenant exception register — understand which tenants behave differently
- Shadow a quarterly KPI review before making any definition changes

---

## 9. Quarterly Review Checklist

Checklist used at every quarterly review attended by both business and data owners:

- Are all KPIs still being accessed — check dashboard usage logs
- Are any KPIs flagged for retirement — not accessed in 90 days
- Are any tenant exceptions overdue for review
- Are any KPI definitions disputed or unclear
- Are any new reporting requirements pending approval
- Is the exchange rate table up to date for all active currencies
- Is the payment method mapping table complete for all tenants
- Is the change log up to date with all changes since last review
- Are all business owner and data owner assignments current — no vacant roles

---

