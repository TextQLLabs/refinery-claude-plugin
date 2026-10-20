---
name: explore-data
description: >-
  Explore your company's data in TextQL: list the connectors you can use,
  browse their tables, and preview rows. Use when someone asks what data
  exists, where a metric lives, or what a table looks like.
---

# Explore data

Read-only. Follow the `refinery` skill for tool details.

1. `refinery_workspace_info`, then `refinery_workspace_context`. Note any definitions or
   preferred tables the org's context names.
2. `refinery_connector_db_list` to list connectors. If the user named a
   source, match it here.
3. `refinery_connector_db_tables` with `connector`, narrowed with `schema`
   or `like` (for example `"%order%"`) when the list is long.
4. `refinery_connector_db_preview` with `connector`, `table` and a small
   `max_rows` (10 is plenty) for the one or two tables that matter.

Answer with a short map: which connector, which tables, what each holds, and
the columns that answer the user's question. Suggest the `query-data` skill
for the next step instead of running a query yourself.
