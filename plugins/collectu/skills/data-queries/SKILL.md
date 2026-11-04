---
name: data-queries
description: "Answer questions about the data of Collectu: values recorded on Collectu Cores (data queries against their published data schemas), the hosted time-series databases of the Collectu Hub, and data objects. Finds the right source and its structure first, then queries with SQL and summarizes the result. Use when the user asks for measured values, trends, averages, counts, the latest value of a machine or sensor, or what data exists."
---

# Querying Collectu data

There are three places data lives. Find out which one holds what the user asks about before
writing any query - and never guess a table, measurement or column name.

| Source | Holds | Discover | Query |
|---|---|---|---|
| A Core's own store | What a pipeline writes with the `outputs.databases.collectu_duckdb_1` module, kept on the Core | `data_schema_search` / `data_schema_get_all_my` - measurements, their columns and the `app_id`s that hold them | `data_query_execute` (DuckDB SQL, run on the Core) |
| A hosted time-series database | What pipelines and other clients write to a database the hub hosts (TimescaleDB) | `tsdb_get_my`, then the database's own catalog with `tsdb_execute` | `tsdb_execute` (PostgreSQL SQL) |
| Data objects | Structured documents shared across pipelines (recipes, limits, master data), versioned | `data_object_get_my_paths`, `data_object_search` | `data_object_get_by_path` |

## A Core's store

Each measurement is one table, named like the measurement, with a `measurement` column, a
`created_at` timestamp and a column per field and tag.

```sql
SELECT AVG(temperature) AS avg_temp FROM machine1
WHERE created_at >= NOW() - INTERVAL 1 HOUR;
```

- Take the measurement and the `app_id` from the data schema, and only those.
- Set `timeout` to at least 10 seconds: the query waits for the Core to pick it up.
- A 504 means the Core did not answer in time. Do not retry at once: check the app's status
  (`app_get_by_app_id`). Offline - tell the user. Online - retry once with a longer timeout.
- A 429 means the owner's plan allows no more queries this minute.
- The data stays on the Core, so a query reaches only Cores that are online.

## A hosted time-series database

```sql
SELECT hypertable_schema, hypertable_name FROM timescaledb_information.hypertables;
SELECT column_name, data_type FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = 'sensors';
SELECT time_bucket('15 minutes', time) AS bucket, avg(value)
FROM sensors WHERE time > now() - interval '1 day' GROUP BY bucket ORDER BY bucket;
```

- Name the instance by its `deployment` (from `tsdb_get_my`).
- **Pass `role: "read"` for every query that only reads.** Without it the highest role the caller
  holds is used, which can write and drop tables.
- Writing - creating a schema or table, inserting, deleting - only when the user asked for exactly
  that, and after showing them the statement.
- Results are text, at most 10000 rows: aggregate in SQL (`time_bucket`, `avg`, `max`,
  `count`) rather than fetching raw rows to summarize them yourself.

## Answering

- Bound every query in time; ask for the period if the question leaves it open and it matters.
- State the period, the source and the unit with the numbers; say when a result was truncated or
  empty, and what an empty result can mean (no data in the period, another measurement name, an
  offline Core).
- For a chart, offer a Grafana panel (the `grafana-dashboards` skill) rather than a table of
  hundreds of rows.
