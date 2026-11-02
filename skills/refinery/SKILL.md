---
name: refinery
description: >-
  Reference for the TextQL tools: every tool, the rules for
  orgs, sandboxes and governed queries, and how to keep results small. Use
  whenever a task involves the organization's data, connectors, warehouse
  queries, or analysis Python that needs org data.
---

# TextQL Refinery MCP tools

The `refinery_*` tools run against the TextQL deployment this plugin points at,
as the signed-in user. Every call goes through that user's roles, OAuth scopes
and org scoping, so a refusal is a permission answer, not a bug to work around.

Every action is its own tool, taking only that action's fields. Results share
one envelope: `operation`, `label`, `data`, and optionally `source`, `sandbox_id`,
`org`, `warning` and `error`. Read `error` and `warning` before trusting `data`.

| Tool | What it does |
| --- | --- |
| `refinery_workspace_info`, `refinery_workspace_context` | Who you are, scopes and tool availability; the org's standing context (read-only) |
| `refinery_workspace_orgs` | The organizations you belong to (read-only) |
| `refinery_workspace_{chat,agent,dashboard,app,artifact}_{list,show}` | Saved work (read-only) |
| `refinery_connector_db_{list,tables,preview}`, `refinery_connector_api_{list,get}` | Connectors, tables, previews and API connector metadata (read-only) |
| `refinery_query` | SQL or governed TQL into a sandbox dataframe |
| `refinery_sandbox_{new,list,status,rm,python,bash}` | Manage sandboxes and run code in them |
| `refinery_{connector,query,sandbox}_{wait,cancel}` | Follow up on a long call that returned while still running |
| `refinery_ontology_diff`, `refinery_ontology_propose` | Turn a sandbox's `library/` edits into an Ontology patch |
| `refinery_org_set`, `refinery_org_clear` | The organization calls run in when they omit `org` |
| `refinery_rpc_search` | Search the deployment's public RPC methods (read-only) |
| `refinery_rpc_call` | Call one method that `refinery_rpc_search` returned |

## Start here, every session

1. `refinery_workspace_info`: who you are signed in as,
   the org, the scopes the grant carries, and which execution tools the org
   allows. Check it before assuming a capability exists.
2. `refinery_workspace_context`: the organization's
   standing instructions about its own data (definitions, preferred tables,
   rules). The web app applies them on every turn, so read and follow them
   before writing any query. Skipping them is how an answer here ends up
   disagreeing with the app.

## If the tools are missing or not signed in (Codex)

When the `refinery_*` tools are unavailable or every call fails with an
authentication error, and you are running in Codex with a shell, sign in
through Codex's own MCP config. Read the server `url` from `mcp.json` in this
plugin's root (two directories above this skill), then run:

```sh
codex mcp add refinery --url <url>
codex mcp login refinery
```

`login` opens a browser window where the user signs in and approves access.
Tell the user to finish it, then start a new thread so the tools load. Without
a shell, ask the user to run those two commands themselves.

## Find the data before you query it

Use the connector tools before `refinery_query`. They are read-only and cheap:

- `refinery_connector_db_list` lists the org's database and BI connectors.
- `refinery_connector_db_tables` with `connector` (id or exact name) and
  optionally `schema` or `like` (for example `"orders%"`) lists tables.
- `refinery_connector_db_preview` with `connector` and `table` returns a few
  rows. Keep `max_rows` small (the default is 50, the cap is 1000).
- `refinery_connector_api_list` and `refinery_connector_api_get` describe API
  connectors.

Name a table the way `refinery_connector_db_tables` listed it. A name that matches nothing, or
several tables, is refused with the candidates.

Retrieval has a fixed order: look for an existing definition in the org
context, then run a governed `.tql`, then fall back to raw SQL. Python is for
post-processing and charting only, never for fetching data.

## Querying

`refinery_query` with:

- `connector`: id or exact name.
- `sql` or `tql` (one of them). Raw SQL runs verbatim in the connector's own
  dialect, one statement per call. DDL/DML is refused unless the connector's
  `allow_sql_write_operations` is enabled — with it on, a write runs under
  the connector's credentials, so send only SELECT statements unless the user asked
  for a change and confirmed it. `tql` runs the approved Ontology copy of a
  governed model, with typed `params`.
- `dataframe`: the name the result gets in the sandbox's Python session.
- `max_rows` (TQL only): caps the rows loaded into the dataframe, not only the
  inline preview. Keep it low for samples; keep every row the analysis needs.
- `sandbox_id`: see below.

## Organizations: `org`

One sign-in connects every organization the user belongs to. A call that
omits `org` (an organization id or exact name) runs in the default set with
`refinery_org_set`, or else the organization the connection was approved in.
When the user picks an organization, set it with `refinery_org_set`, remember
its id, and pass it as `org` on every later call. `refinery_workspace_orgs`
lists them, marks the current one and the default, and
says why any is unavailable. Results
from another organization carry `org` in the envelope. Keep the same `org` for
every call of one task, sandbox calls included: a `sandbox_id` belongs to the
organization that created it. If `refinery_workspace_orgs` says the app must be reconnected, the
connection only covers its original organization until the user reconnects it.

## Sandboxes: reuse `sandbox_id`

Results that touch a sandbox return a `sandbox_id`. Pass that same
`sandbox_id` to every later `refinery_query` and `refinery_sandbox_*` call in
the task, so the dataframe you loaded is still there when you run Python on it.
`refinery_sandbox_python` and `refinery_sandbox_bash` require it, and each
result's `hint` repeats the id to pass. Omitting it on `refinery_query` starts
a fresh, isolated sandbox with none of your earlier state.

- `refinery_sandbox_python` with `code` runs in a persistent kernel.
  Variables and dataframes carry across calls in the same sandbox.
- `refinery_sandbox_bash` with `code` runs a fresh process and may be
  disabled for the org (check `refinery_workspace_info`).
- `refinery_sandbox_new`, `_list`, `_status` and `_rm` manage sandboxes.
  Keep the sandbox for follow-up questions in the same conversation; idle
  sandboxes are reaped after about an hour. `env` sets environment variables for a run.
- A call that takes longer than about 25 seconds returns `operation_id` and a
  warning instead of its result; the work keeps running. Call the matching
  `_wait` tool (`refinery_query_wait`, `refinery_sandbox_wait` or
  `refinery_connector_wait`) with that `operation_id` and the same `org` until
  the result arrives. The matching `_cancel` tool stops it. Do not re-run the
  call: it would run twice.
- Kernel state is lost when a sandbox restarts or is reaped after about an
  hour idle. Files under `/sandbox/files` survive. Re-run the data load if a
  warning says the sandbox restarted.
- `pip install` is blocked. Common libraries (pandas, polars, numpy, scipy,
  sklearn, matplotlib, seaborn and more) are preinstalled.

## Changing the Ontology: `refinery_ontology_*`

Every sandbox holds a working copy of the Ontology at `/sandbox/files/library`.
To change it:

1. Edit files there with `refinery_sandbox_python` or `refinery_sandbox_bash`, in one sandbox.
2. `refinery_ontology_diff` with that `sandbox_id` shows the pending changes.
3. `refinery_ontology_propose` with the same `sandbox_id`, a `title` and a
   `description` authors a patch as the user. Pass `patch_number` to revise a
   patch the user already authored, or `draft` to file it for the user to
   review first.

The user's Ontology permissions decide the outcome. If the changes touch a
path they cannot read, both tools refuse and name the paths. Where they own every touched path, the patch merges (`outcome`
`merged`). Otherwise it waits for the owners (`awaiting_approval`, with the
`reason`). Report the `outcome` and `url` as returned. Do not describe an
open patch as applied. Get the user's go-ahead before proposing.

## Keep results small

Everything a tool returns lands in your context. Aggregate in the sandbox and
print only the summary you need. For previews and samples keep `max_rows`
(TQL) and `LIMIT` (raw SQL) low — both bound the loaded dataframe, so keep
every row the analysis needs — and set `max_output_chars` on
`refinery_sandbox_python` and `refinery_sandbox_bash` runs when output could be large. A `truncated` flag means your code
printed too much: aggregate first rather than raising the cap.

## Saved work

`refinery_workspace_chat_list`, `_agent_list`, `_dashboard_list`, `_app_list`
and `_artifact_list` find existing work, and the matching `_show` tool with `ref`
(an id or unique name from a listing) opens one. Prefer an existing dashboard
or agent answer over recomputing it.

## Data apps: you CAN create and edit them

The `_app_` tools only read, but building and changing data apps is a core use
of these tools: every write goes through `AppService` with
`refinery_rpc_call`, by design. Never tell the user editing an app is
unsupported, needs engineering, or has to be handed off to Ana. Search
`query: "app"` for this deployment's methods; the usual ones are
`GetAppFiles` (current code and files), `UpdateApp`, `CreateApp`,
`RefreshApp`, `InvokeAppComputeFunction`, `QueryAppData`, and
`ListAppVersions`/`RestoreAppVersion` to undo.

Read before you write: `GetAppFiles`, change what the user asked for, then
`UpdateApp`. `code` replaces the whole entry file. `files` and
`computeFunctions` are ignored unless `replaceFiles` or
`replaceComputeFunctions` is true, and then they replace the full list, so
send everything you want to keep. Edits go live immediately; confirm with the
user before changing an app other people use. After a write, call
`VerifyAppRender` and fix the app until it returns `ready: true` before giving
the user its URL.

## The escape hatch: `refinery_rpc_search` and `refinery_rpc_call`

Use these only when no tool above does the job. The ChatGPT connector does not
list them; there, anything below that needs them, data app writes included, is
done in the TextQL app instead, so tell the user that rather than attempting it. Always search first with a
few keywords (`query: "ontology upsert"`), then call the exact `target` it
returned with a proto-JSON `body` (camelCase field names). Writes through
`refinery_rpc_call` are real and immediate: confirm destructive or org-visible
changes with the user first, and never put login credentials in a body.

Connector groups have no dedicated tool. Use `refinery_rpc_call` with these
`ConnectorService` targets: `ListConnectorGroups` (body `{}`),
`CreateConnectorGroup` (`name`, `description`, `connectorIds`),
`UpdateConnectorGroup` (`groupId`, `name`, `description`),
`DeleteConnectorGroup` (`groupId`), `UpdateConnectorGroupMembers` (`groupId`,
`addConnectorIds`, `removeConnectorIds`), `ShareConnectorGroup` (`groupId`,
`grants` of `{memberId|roleId|groupId, accessType: viewer|editor}`) and
`UpdateConnectorGroupSettings` (`groupId`, `config` with `connectorType:
"AZURE_SYNAPSE"` and an `azureSynapse` object). List first and act only on a
group whose `canManage` is true. Only groups the caller manages or was given (directly, by role, or by user group)
are listed; a connector reached only through its own sharing is standalone. A connector can belong to several groups; adding it preserves its other
memberships. Direct invitations and every group's viewer/editor grants combine at the highest level. Adding a
connector to a shared group gives everyone it is shared with access, so list the group's grants first, tell the
user who gains access, and confirm; do the same for who loses access before removing one.
Use `ListConnectorGroupGrants` (`groupId`), `UpdateConnectorGroupGrant` (`groupId`, `grantId`, `accessType`),
and `RevokeConnectorGroupGrant` (`groupId`, `grantId`) to manage group access. All three need the `connector:write` scope
and group management rights; updating also needs edit access to every connector in the group. Revocation or group removal
only removes that group's contribution, along with any per-connector copies older versions wrote when sharing the group. Settings changes affect the connector in all its groups; show each
setting from its current value to the new one and confirm before changing live connections. For the full rules, fetch the deployment skill
with `refinery_rpc_call` on `OntologyManagementService/GetSkill`, body
`{"trigger":"connector-groups"}`.

## Never fabricate

If a query fails, a table is missing, or a permission is refused, say so and
show the error. Do not invent numbers, tables or columns.
