---
name: find-saved-work
description: >-
  Find existing TextQL work before redoing it: saved chats, dashboards,
  agents, apps and artifacts. Use when someone asks for a dashboard, a past
  analysis, or "the report on" something.
---

# Find saved work

Read-only. Follow the `refinery` skill for tool details.

1. Pick the kind the user means: `refinery_workspace_dashboard_list`,
   `refinery_workspace_chat_list`, `refinery_workspace_agent_list`,
   `refinery_workspace_app_list` or `refinery_workspace_artifact_list`.
   When unsure, try the dashboard and chat lists first.
2. Open the best match with the matching `_show` tool (for example
   `refinery_workspace_dashboard_show`) and `ref` (an id or unique name from
   the listing).
3. Summarize what it contains and when it was made. Give its link when the
   result includes one.

Prefer an existing dashboard or analysis over recomputing it. If nothing
matches, say so and offer the `query-data` skill instead.
