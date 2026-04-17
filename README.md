# aumet-etl-assessment
ETL / ELT pipeline design for Odoo multi-tenant stock movement data warehouse — technical assessment submission
# Aumet ETL Assessment

End-to-end ETL / ELT pipeline design for extracting Odoo stock movement data
from multiple pharmacy tenant databases into a centralised ClickHouse data warehouse.

## Context

Aumet operates a multi-tenant pharmacy ERP platform built on Odoo / PostgreSQL.
This assessment covers the full design of a data pipeline that extracts stock
movement outcomes from each tenant, transforms them into an analytics-ready
structure, and loads them into ClickHouse for reporting.

## Repo Structure
