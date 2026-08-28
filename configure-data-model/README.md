# FunnelStory Data Model Configuration

## Purpose

Configure FunnelStory data models via MCP tools: pick model type, map columns, add joins, preview, and save. Use when creating or updating a data model, or mapping source data into FunnelStory entities (accounts, users, activities, conversations, and other model types).

**Prerequisite:** Author and validate queries first using [connection-queries](../connection-queries/README.md).

## Prerequisites

- **FunnelStory MCP** connected for the target workspace.
- **`get_data_connections`** — locate source connections and the Semantic DB connection.
- **`execute_query`** — inspect existing models in the semantic DB.
- **`get_data_connection_schema`** and **`preview_data_connection`** — validate queries (via connection-queries workflow).
- **`preview_data_model`** — test the full config without saving.
- **`configure_data_model`** — create or update a saved model.

## Step 1 — Clarify and inspect

1. **Clarify the goal** — which entity type (account, user, activity, conversation, …) and what source system or table backs it.
2. **Inspect existing config** — query semantic DB:

```sql
SELECT id, name, type, draft FROM data_models ORDER BY type, name
```

For the target type:

```sql
SELECT config FROM data_models WHERE type = '<type>' LIMIT 1
```

3. **Read the guide for that type** — when available, call `read_resource("file://data_model/guide.md")` for the index, then the per-type URI (for example `file://data_model/account.md`). If those resources are not registered yet, use the key-column table and pitfalls below.
4. **Write and validate queries** — follow **`connection-queries/`** to author `data_source.query` (and join queries). Confirm with `get_data_connection_schema` and `preview_data_connection`. Never invent column names.

## Step 2 — MCP tool workflow

```
connection-queries/  →  preview_data_connection
        ↓
preview_data_model     ← full config including mappings and joins
        ↓
configure_data_model   ← save (pass id to update)
```

- **`preview_data_connection`** — test a single query against one connection (up to 25 rows).
- **`preview_data_model`** — test the full model payload (primary `data_source`, `mappings`, optional `joins`) without saving.
- **`configure_data_model`** — create or update. Omit `id` to create; pass `id` to update.

Always preview before configure.

## Step 3 — Data source shape

Every model has a **`data_source`**. Every data source includes a **query**. Use exactly one of:

### Connection + query

```json
{
  "data_connection_id": "<uuid>",
  "query": "SELECT ..."
}
```

Query syntax is connection-specific. Read **`connection-queries/`** before writing this field.

### Dataset + query

```json
{
  "dataset": {
    "name": "my_dataset",
    "query": "SELECT account_id, domain FROM my_dataset"
  }
}
```

Use when rows already live in a workspace dataset built from one or more connections.

## Step 4 — Joins (when needed)

Add **`joins`** only when the primary query cannot include the needed columns (extra connection, enrichment table, second SOQL/HS block result joined in SQL).

Each join:

```json
{
  "type": "left join",
  "columns": [
    { "left_column": "account_id", "right_column": "id" }
  ],
  "data_source": {
    "data_connection_id": "<uuid>",
    "query": "SELECT id, health_score FROM ..."
  }
}
```

- **`left_column`** — column from the primary result (or prior joins).
- **`right_column`** — column from this join's query result.
- Join side uses the same data source rules (connection + query, or dataset + query).
- Joins chain as LEFT JOINs in order. Primary columns win on name conflicts.
- Prefer folding columns into the primary query when they share a connection.

## Step 5 — Model config payload

```json
{
  "name": "Accounts",
  "type": "account",
  "draft": true,
  "data_source": { ... },
  "mappings": [
    { "column": "account_id", "property": "account_id" },
    { "column": "name", "property": "name" }
  ],
  "joins": [],
  "refresh": { "interval": "6h" }
}
```

### Key columns (map these properties; null key values drop the row silently)

| type | key properties |
|------|----------------|
| account | account_id |
| user | user_id, account_id |
| contact | email |
| invite | user_id, account_id, invitee_email |
| activity / product_activity / non_product_activity / product_feature_activity | activity_id, account_id |
| support_ticket | account_id, ticket_id |
| conversation | account_id, conversation_id |
| account_metric | account_id, metric_id, timestamp |
| meeting | account_id, meeting_id |
| note | note_id |
| task | task_id |
| strategy | account_id |
| plan | plan_id |
| subscription | subscription_id, account_id |
| usage | account_id, dimension, timestamp |
| product | product_id |

**Mappings:** `column` is the query output name; `property` is the FunnelStory field name. Property names must match exactly (`timestamp`, not `created_at` as the property key).

## Cross-cutting pitfalls

- **Draft models** — missing key mappings, missing query, or incomplete joins keep `draft: true`.
- **Null keys** — rows with null in any key property are skipped at refresh with no error.
- **Activity auto-user creation** — requires both `user_id` and `email` mapped; email alone does nothing.
- **Account properties** — values over 4000 characters become `{"error":"max_length_exceeded"}`.
- **metric_id / activity_id** — use stable generic IDs, not names or timestamps embedded in the ID.
- **product_id** — only when products exist in the workspace product model.
- **Domain joins** — normalize both sides (strip protocol/www; handle country-code TLDs) before joining on domain.
- **Conversation `key`** — enables context-graph mode (needle movers, NLP). Timeline-only models omit it.
- **`<fs_analyze>` / `<fs_sanitize>`** — column prefixes on conversation text trigger NLP pipelines.

## Config checklist

- [ ] Query validated via **`connection-queries/`** and `preview_data_connection`
- [ ] Query returns all key columns non-null for rows you care about
- [ ] Column names match `mappings[].column`
- [ ] Required properties for the model type are mapped
- [ ] `preview_data_model` shows expected rows and `mapped_properties`
- [ ] Join keys tested when joins are present
