---
name: grafana-dashboards
description: "Build or change dashboards in the Grafana organization the Collectu Hub hosts for the user or their organization: find the data source and the tables, write the panel queries, add, update or remove panels and dashboards. Use when the user wants to see their machine or sensor data as charts, gauges or tables in Grafana, or asks to change a dashboard."
---

# Grafana dashboards on the Collectu Hub

The hub hosts a Grafana organization per owner. Its tools act as the signed-in user, with the role
their permissions give them in Grafana - a viewer can read dashboards but not save them.

## Building a panel

1. **Data source.** `grafana_get_datasources` names the data sources with their `type` and `uid`.
   A hosted time-series database is a PostgreSQL data source (`grafana-postgresql-datasource`).
2. **Tables and columns.** Look them up before writing a query - `tsdb_execute` with
   `role: "read"` on the matching instance (`tsdb_get_my`), see the `data-queries` skill. Never
   guess a table or column.
3. **Query.** Let Grafana bound the time with its macros, so the panel follows the time picker:

   ```sql
   SELECT $__timeGroupAlias(time, $__interval), avg(value) AS temperature
   FROM sensors
   WHERE $__timeFilter(time) AND machine = 'press_1'
   GROUP BY 1 ORDER BY 1
   ```

   Test the same query with `tsdb_execute` (with the macros replaced by a fixed period) before
   saving it, so a panel never shows "no data" because of a typo.
4. **Panel.** `timeseries` for values over time, `stat` or `gauge` for the latest value,
   `table` for lists, `bargauge`, `barchart` and `piechart` for comparisons. Set the unit
   (`fieldConfig.defaults.unit`: `celsius`, `percent`, `kwatt`, `bar`, ...) and, for a stat or
   gauge, `min`, `max` and thresholds when the user names limits.
5. **Save.** `grafana_create_panel` places it beside the last panel (the dashboard is 24 columns
   wide). Give every change a `message` - it is the version history of the dashboard.

## Changing a dashboard

- Find it with `grafana_search_dashboards` and read it with `grafana_get_dashboard` first: panels
  are named by their `id`, and a collapsed row keeps its panels inside it.
- Change one panel with `grafana_update_panel`, sending only the keys that change (`targets`
  replaces every query of the panel). Resend the whole dashboard with `grafana_update_dashboard`
  only for what concerns the dashboard itself - variables, time range, refresh, layout.
- A 409 means somebody saved the dashboard at the same moment, or that it uses Grafana's new
  layouts (tabs, auto grids), which the tools cannot save without losing them. Read it again, or
  tell the user to make that change in Grafana.
- Deleting a panel or dashboard only on the user's explicit request, naming what will be deleted.

## A new dashboard

`grafana_create_dashboard` with a title (and a folder from `grafana_get_folders` if the user keeps
dashboards in folders), then its panels one by one. Lay out what belongs together in one row:
key figures as stats at the top, their trends below. Tell the user its URL when done.
