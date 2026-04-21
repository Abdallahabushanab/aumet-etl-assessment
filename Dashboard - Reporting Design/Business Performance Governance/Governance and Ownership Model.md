## Aumet Pharmacy Network — Reporting Governance

---

## 1. Purpose

This document defines how the KPI framework is owned, maintained, and kept consistent over time. It establishes the processes that prevent reporting from becoming unreliable, inconsistent, or dependent on individual knowledge. It is the authoritative reference for anyone involved in defining, changing, or approving a reported metric.

---

## 2. Ownership Model

Every KPI has two owners — a business owner and a data owner. They are different people with different responsibilities.

|Role|Responsibility|
|---|---|
|Business owner|Defines what the KPI means in business terms. Approves changes to the definition. Decides if a new KPI is needed. Signs off on retirement of unused KPIs.|
|Data owner|Implements the KPI in the warehouse. Maintains the calculation logic. Alerts the business owner when source data changes affect the KPI. Documents technical changes in the change log.|

Neither owner can change a KPI definition unilaterally. Changes require sign-off from both.

### 2.1 Ownership Matrix

|KPI|Business Owner|Data Owner|
|---|---|---|
|Total orders|Head of Operations|Data Engineering Lead|
|Total quantity moved|Head of Operations|Data Engineering Lead|
|Average order value (USD)|Head of Finance|Data Engineering Lead|
|Movement volume|Head of Operations|Data Engineering Lead|
|Expired lots|Head of Compliance|Data Engineering Lead|
|Active tenants|Head of Product|Data Engineering Lead|
|% missing expiry|Head of Compliance|Data Engineering Lead|
|Cash vs insurance split|Head of Finance|Data Engineering Lead|
|Risk gap|Head of Operations|Data Engineering Lead|
|Currency conversion|Head of Finance|Data Engineering Lead|

---

## 3. How Business and Data Teams Work Together

The relationship follows a clear separation — business defines the question, data defines the answer.

**Business team responsibilities:**

- Define what a KPI should measure in plain language
- Confirm the KPI is being used and is still relevant
- Raise disagreements when a reported number does not match expectations
- Approve any change to a KPI definition before it is implemented

**Data team responsibilities:**

- Translate the business definition into SQL calculation logic
- Flag when source data changes make a KPI unreliable
- Document every change in the change log
- Proactively alert business owners when data quality issues affect a KPI

**Joint responsibilities:**

- Quarterly KPI review — both teams attend
- Disagreement resolution — both owners must agree before a number is changed
- New KPI approval — requires sign-off from both before any development begins

---

## 4. How Disagreements on Numbers Are Handled

Disagreements happen when a business stakeholder reports a different number than what the dashboard shows. The resolution process is:

**Step 1 — Raise a discrepancy ticket** The stakeholder raises a formal discrepancy — stating the expected number, the reported number, the date range, the tenant, and the filter applied.

**Step 2 — Data owner investigates within 48 hours** The data owner traces the number from the dashboard back to the source — checking the SQL, the transformation rules, and the raw data in staging.

**Step 3 — Root cause classification** The discrepancy is classified as one of:

- Data quality issue — source data is wrong or incomplete
- Calculation logic issue — the SQL does not match the agreed definition
- Definition ambiguity — the KPI definition is unclear and open to interpretation
- User error — the filter applied produced a different scope than intended

**Step 4 — Resolution**

- Data quality issue → flagged in validation log, source team notified, number corrected
- Calculation logic issue → data owner fixes, business owner approves, change logged
- Definition ambiguity → business owner and data owner meet to agree on definition, change logged
- User error → user guided to correct filter combination, no change to KPI

**Step 5 — Document outcome** Every disagreement and its resolution is documented in the KPI change log regardless of outcome.

---

## 5. Standardization Across Tenants

Because Aumet operates across multiple pharmacy clients in multiple countries, tenant consistency is one of the hardest governance challenges.

### 5.1 Shared Definitions — Default Behaviour

All tenants use the same KPI definitions by default. The calculation logic in ClickHouse is identical for every tenant — the only variable is `tenant_id` in the WHERE clause.

|Shared across all tenants|Detail|
|---|---|
|KPI definitions|Same business definition regardless of tenant|
|Calculation logic|Same SQL — tenant_id is a filter not a variable|
|Currency|All tenants report in USD — local currency preserved separately|
|Reporting frequency|Same cadence for all tenants|
|Data quality thresholds|Same alert thresholds — e.g. % missing expiry > 7% triggers alert for any tenant|

### 5.2 Tenant-Specific Differences — Permitted Exceptions

Some differences between tenants are real and legitimate — not errors. These are handled as documented exceptions rather than separate KPI definitions.

|Situation|How it is handled|
|---|---|
|Tenant does not use lot tracking|% missing expiry will be high — documented as known characteristic, not a data quality alert|
|Tenant operates in a currency not yet supported|Monetary KPIs marked as unavailable for that tenant — not calculated until currency mapping is added|
|Tenant uses a different Odoo version|Table name differences handled at extraction layer — e.g. `stock_lot` vs `stock_production_lot` — same KPI output|
|Tenant has insurance-specific workflows|Payment method field mapped to standard values — custom mapping documented in tenant configuration table|
|Tenant requests a custom metric|Treated as a new supporting metric — must go through full KPI approval process — never added to shared executive layer|

### 5.3 How Exceptions Are Documented

Every tenant exception is recorded in the tenant configuration table:

sql

```sql
CREATE TABLE etl_config.tenant_exceptions (
    tenant_id           VARCHAR,
    exception_type      VARCHAR,  -- 'known_characteristic' / 'currency_gap' / 'custom_mapping'
    kpi_affected        VARCHAR,
    description         TEXT,
    approved_by         VARCHAR,
    approved_at         TIMESTAMP,
    review_date         DATE      -- when this exception should be reviewed
);
```

Exceptions are reviewed quarterly. If an exception has been in place for more than one year it is escalated to the business owner for a decision — either resolve it or formally accept it as a permanent characteristic.

---

## 6. KPI Documentation Standards

Five documents make up the complete KPI documentation set:

|Document|Purpose|Owner|Updated|
|---|---|---|---|
|Document 1 — KPI Framework and Definitions|Business-facing — what KPIs exist and what they mean|Business owner per KPI|When definition changes|
|Document 2 — Governance and Ownership Model|Process-facing — how KPIs are owned and changed|Data Engineering Lead|Quarterly|
|Document 3 — Technical Reference|Engineer-facing — source-to-target mapping, SQL logic, field definitions|Data Engineering Lead|When calculation changes|
|Change log|Record of every KPI change — who, what, when, why|Data Engineering Lead|Every change|
|Tenant exception register|Record of all documented tenant-specific deviations|Data Engineering Lead|Every exception|

### 6.1 Change Log Structure

Every change to a KPI definition or calculation must be logged:

sql

```sql
CREATE TABLE governance.kpi_change_log (
    change_id           VARCHAR   PRIMARY KEY,
    kpi_name            VARCHAR,
    change_type         VARCHAR,  -- 'definition' / 'calculation' / 'threshold' / 'retirement'
    previous_value      TEXT,
    new_value           TEXT,
    reason              TEXT,
    requested_by        VARCHAR,
    approved_by_business VARCHAR,
    approved_by_data    VARCHAR,
    effective_date      DATE,
    created_at          TIMESTAMP
);
```

No KPI change is implemented without a corresponding change log entry. This is enforced by the data owner before any deployment.

---

## 7. Reporting Cadence and Review Model

### 7.1 Monitoring Frequency

|KPI|Operational monitoring|Leadership review|
|---|---|---|
|Total orders|Daily|Weekly|
|Total quantity moved|Daily|Weekly|
|Average order value|Daily|Monthly|
|Movement volume|Daily|Weekly|
|Expired lots|Daily — alert on any new expired lot|Weekly|
|Active tenants|Daily — alert if tenant goes silent 25hr+|Weekly|
|% missing expiry|Daily — alert if > 7%|Weekly|
|Cash vs insurance split|Weekly|Monthly|
|Risk gap|Daily|Weekly|

### 7.2 Leadership Reviews vs Operational Reviews

| Severity   | Definition                         | Response time                           |
| ---------- | ---------------------------------- | --------------------------------------- |
| Frequency  | Monthly                            | Daily / Weekly<br>                      |
| Format     | Executive dashboard — 8 KPIs       | Operational dashboard — full detail<br> |
| Attendees  | C-level, regional directors        | Ops team, data team<br>                 |
| Purpose    | Strategic signals — where to focus | Tactical action — what to fix<br>       |
| Output     | Decisions on priorities<br>        | Actions on specific issues<br>          |
| Data level | Aggregated totals<br>              | Row level detail<br>                    |

### 7.3 How Reporting Issues Are Escalated

| Severity | Definition                                             | Response time         | Escalation path                                      |
| -------- | ------------------------------------------------------ | --------------------- | ---------------------------------------------------- |
| Critical | Number is wrong or missing for a leadership review     | 4 hours               | Data owner → Data Engineering Lead → Head of Product |
| High     | KPI is unavailable for a tenant for more than 24 hours | 24 hours              | Data owner → Data Engineering Lead                   |
| Medium   | KPI definition is disputed                             | 48 hours              | Data owner + Business owner joint resolution         |
| Low      | KPI not accessed — candidate for retirement            | Next quarterly review | Business owner decision                              |

### 7.4 How Reporting Requirements Changes Are Handled

When a new reporting requirement is raised:

- Business owner submits a request describing the business question the new metric answers
- Data owner assesses feasibility — is the data available in the warehouse?
- Both owners agree on definition before any development begins
- New metric is assigned to executive or operational layer — never both
- If executive layer — one existing KPI must be retired or moved to operational
- Change is logged and documentation updated before the metric goes live

---

## 8. Trust and Sustainability

### 8.1 How the Reporting Layer Stays Trusted

- Every number traces back to a defined source — no magic numbers
- Discrepancies are resolved through a documented process — not gut feel
- Validation checks run after every ETL batch — stakeholders are alerted before they notice a problem themselves
- Data quality issues are visible in the dashboard — not hidden
- The change log is public to all stakeholders — anyone can see what changed and why

### 8.2 How Consistency Is Maintained Over Time

- KPI definitions live in Document 1 — not in someone's head
- Calculation logic lives in documented SQL — not in a personal notebook
- Every change requires two sign-offs — preventing unilateral drift
- Quarterly reviews catch unused or outdated KPIs before they cause confusion
- Tenant exceptions are documented and reviewed — not silently tolerated

### 8.3 How the Framework Avoids Depending on One Person

- Every KPI has two named owners — if one leaves the other remains
- All documentation is stored in the shared repository — not on a personal drive
- The change log ensures institutional memory survives staff turnover
- New engineers onboard using Document 3 — the technical reference — without needing to ask anyone
- The quarterly review involves both business and data teams — knowledge is shared not siloed

### 8.4 How the Framework Scales

|Growth scenario|How it is handled|
|---|---|
|New pharmacy tenant onboarded|Inherits standard KPI set — tenant_id added to registry — no new KPI definitions|
|New product category added|Existing KPIs apply — product is a dimension not a KPI — no framework change|
|New reporting requirement|Goes through full KPI approval process — added to operational layer first|
|New country and currency|Currency added to conversion table — monetary KPIs automatically include new tenant|
|New payment method|Added to payment method mapping table — cash vs insurance split updates automatically|
|Team member leaves|Documentation and ownership matrix ensure continuity — named successor assigned within 30 days|

---

Document 2 complete. Document 3 is a short technical reference pointing to Assessment 1 with a change log and ownership matrix added. Want me to write that now or go straight to compiling the full assessment 3 document?