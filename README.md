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



## Design Document — Contents

| Part | Topic | Description |
|---|---|---|
| 1 | Source discovery | Odoo table mapping, join strategy, extraction SQL, source validation |
| 2 | Extraction design | Multi-tenant loop, tenant registry, watermark, incremental strategy |
| 3 | Transformation logic | Two-layer design, cleaning rules, business rule decisions |
| 4 | Warehouse modeling | ClickHouse wide table, DDL, partitioning, ordering |
| 5 | Data quality | Three-layer validation, reconciliation checks, validation log |
| 6 | Scalability and reliability | Schema drift, monitoring and alerting, backfill process |
| 7 | Documentation and handover | Glossary, data dictionary, business matrix, runbook |

## Diagrams

All diagrams were built in draw.io.

| File | Covers |
|---|---|
| `odoo_schema.png` | Source tables, columns, primary keys, foreign keys, join relationships |
| `architecture.png` | End-to-end pipeline: pharmacy tenants → ETL engine → ClickHouse |
| `control_tables.png` | Tenant registry and watermark table structure with example data |
| `loop.png` | Step-by-step incremental extraction flow per tenant |
| `reliability.png` | Schema drift detection, monitoring and alerting, backfill process |

## Tech Stack Assumptions

| Layer | Technology |
|---|---|
| Source | Odoo ERP on PostgreSQL (v16+) |
| Orchestration | Apache Airflow |
| Warehouse | ClickHouse |
| Secrets management | AWS Secrets Manager / HashiCorp Vault |
| Language | Python |

## Key Design Decisions

- **Incremental extraction** by `write_date` — not full load — keeps runs fast at scale
- **Tenant registry pattern** — adding a new pharmacy requires one database row, zero code changes
- **Wide denormalised table** in ClickHouse — ETL absorbs JOIN complexity so analysts query one table
- **ReplacingMergeTree** engine — handles duplicate rows automatically on re-insertion
- **Two-layer load** — staging first then fact — protects reporting layer from bad data
- **Watermark per tenant** — each pharmacy tracks its own extraction timestamp independently


