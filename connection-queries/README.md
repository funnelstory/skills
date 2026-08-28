# FunnelStory Connection Queries

## Purpose

Write and validate **`data_source.query`** (and join queries) for FunnelStory data models. Covers SQL warehouses, Salesforce SOQL blocks, HubSpot blocks, Attio blocks, and synced integration tables.

Use this sub-skill **before** [configure-data-model](../configure-data-model/README.md). After the query works, continue there for mappings, joins, preview, and save.

## Prerequisites

- **FunnelStory MCP** connected for the target workspace.
- **`get_data_connections`** — identify the source connection and its type.
- **`get_data_connection_schema`** — list tables and columns before writing SQL (never invent column names).
- **`preview_data_connection`** — validate the query returns expected columns and sample rows (up to 25) before using it in `preview_data_model` or `configure_data_model`.

## Workflow

Every data model `data_source.query` must match the connection type. Block-based connectors embed fetch steps; SQL below runs over the materialized tables.

1. Identify the connection type from workspace config.
2. Write the query using the correct dialect or block syntax (sections below).
3. Run **`preview_data_connection`** and confirm output column names match planned `mappings[].column` values.
4. Hand off to **`configure-data-model/`** for the full model payload.

## SQL warehouses (Postgres, MySQL, Snowflake, BigQuery, Databricks, Redshift)

Write SQL directly. No block wrappers.

- Postgres: `ILIKE`, `->>` JSON, `DATE_TRUNC`, window functions.
- MySQL: `SUBSTRING_INDEX` for splits; `JSON_EXTRACT`; no `ILIKE` — use `LOWER(...) LIKE`.
- BigQuery: backtick tables `` `project.dataset.table` ``, `JSON_VALUE`, `UNNEST`.
- Snowflake / Databricks: dialect-specific functions; `catalog.schema.table` naming.

## Salesforce (SOQL blocks)

**Block syntax:** `-- SOQL_START: query_name` … `-- SOQL_END`

The block name becomes a table for SQL below. Multiple blocks can appear in one query; join them with SQL.

```sql
-- SOQL_START: accounts
SELECT Id, Name, Website FROM Account WHERE IsDeleted = false
-- SOQL_END

SELECT
  a.Id AS account_id,
  a.Name AS name,
  a.Website AS domain
FROM accounts a
```

Notes:

- Custom fields end with `__c`.
- `StageName` closed-won values are customer-specific — confirm with the user.
- `WHERE field != NULL` is valid SOQL (not SQL `IS NOT NULL`).
- Date fields may arrive as strings; use SQLite date functions in downstream SQL when FunnelStory processes locally.
- Resolve owners via a separate `User` SOQL block joined on `OwnerId` or custom lookup fields.

## HubSpot (HS blocks)

**Block syntax:** `-- HS_START: query_name` … `-- HS_END`

JSON config after the start line can specify `object_type`, `properties`, `filter_groups`.

Blocks populate internal tables such as `deals`, `companies`, `deal_companies`, `engagements`. Query them with SQL/JSON extracts:

```sql
-- HS_START: companies
{"object_type": "companies", "properties": ["name", "domain"]}
-- HS_END

SELECT
  id AS account_id,
  domain,
  properties->>'name' AS name
FROM companies
```

## Attio (AT blocks)

**Block syntax:** `-- AT_START: query_name` … `-- AT_END`

Field values are JSON arrays. Use `json_extract(field, '$[0].value')`. Record IDs: `json_extract(id, '$.record_id')`. Deduplicate association joins.

## Synced integration tables

Connectors that sync without user fetch blocks (for example Zendesk, Intercom, Slack) expose pre-built internal tables. When a query is used, it is SQL over those synced tables — inspect schema first, do not invent columns.

## Checklist

- [ ] Connection type identified from workspace config
- [ ] Correct dialect or block syntax for that type
- [ ] `preview_data_connection` returns expected columns and sample rows
- [ ] Output column names match planned model `mappings[].column`
