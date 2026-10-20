---
name: switch-org
description: >-
  See which TextQL organizations you can use, set the one to work in, and
  switch without signing in again. Use when someone names another
  organization, workspace or customer, asks to set or change their org, or
  asks which orgs they have.
---

# Switch organization

1. `refinery_workspace_orgs` lists the organizations, marks the current
   one and the default, and says why any is unavailable.
2. To set the organization to work in, call `refinery_org_set` with `org`
   (its id or exact name). Calls that omit `org` run there from then on, in
   this conversation and later ones, until it is set again or cleared with
   `refinery_org_clear`.
3. After setting it, remember the organization's id and pass it as `org` on
   every tool call, sandbox calls included. Do not rely on the default alone:
   passing `org` keeps each call pinned even if the default changes elsewhere.
4. A `sandbox_id` belongs to the organization that created it, so start a new
   sandbox after switching.
5. Results from another organization carry `org` in the envelope. Name the
   organization in your answer so the user knows where the numbers came from.

If `refinery_workspace_orgs` says the app must be reconnected, tell the user to disconnect and
reconnect it from a browser. The current connection only covers its original
organization until then.
