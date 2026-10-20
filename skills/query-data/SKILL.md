---
name: query-data
description: >-
  Answer a question from your company's data with TextQL: run SQL or a
  governed TQL query into a Python sandbox, then aggregate, analyze or chart
  the result there. Use for numbers, trends, comparisons and charts.
---

# Query and analyze data

Follow the `refinery` skill for tool details and limits.

1. `refinery_workspace_context`. Use an existing definition or governed
   model before writing raw SQL.
2. If the table is unknown, find it first with the `explore-data` steps.
3. `refinery_query` with `connector`, `sql` (one SELECT, in the
   connector's dialect) or `tql`, and a `dataframe` name. Do the heavy
   aggregation in SQL. `LIMIT` (sql) and `max_rows` (tql) bound the loaded
   dataframe, not just the inline preview — use them for samples and
   previews, and keep every row the analysis needs. Writes are refused
   unless the connector allows write operations; send only SELECT statements unless
   the user asked for a change.
4. Keep the returned `sandbox_id` and pass it to every later call.
5. `refinery_sandbox_python` with `code` and that `sandbox_id` for anything
   SQL cannot do cleanly: joins across queries, statistics, charts. Print
   only the summary, and set `max_output_chars` when output could be large.

Report the answer first, then the query you ran and any caveat (filters,
date range, rows excluded). If a query fails or a table is refused, show the
error. Never fill gaps with invented numbers.
