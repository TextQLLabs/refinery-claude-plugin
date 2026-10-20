# TextQL Refinery plugin for Claude Code

Bundles the TextQL Refinery MCP server (`/refinery-mcp`) with a skill that
teaches Claude how to use its tools: load the org context first, look up
connectors and tables before querying, reuse `sandbox_id` across calls, and keep
results small.

## Layout

- `.claude-plugin/plugin.json`: plugin manifest
- `.mcp.json`: registers the remote Refinery MCP server over HTTP. Claude Code
  runs the OAuth flow on first use.
- `skills/`: generated, not committed. `build.sh` emits the skills marked
  `plugin-mcp` from `integrations/agent-skills/skills` (the canonical home of
  every shipped skill) into the zip — `skills/refinery/SKILL.md` teaches the
  `refinery_*` tools. Generate the same set locally before `--plugin-dir`
  testing; a full-directory copy would also load `refinery-cli`, which
  declares the same `name: refinery` with CLI instructions instead.

## Pointing it at your deployment

Every deployment has its own host, so `.mcp.json` reads it from an environment
variable with the TextQL cloud host as the default:

```json
"url": "https://${REFINERY_HOST:-app.textql.com}/refinery-mcp"
```

TextQL cloud users set nothing. Everyone else exports the host name of their
deployment (no scheme, no path) before launching Claude Code:

```sh
export REFINERY_HOST=textql.example.com
claude
```

The variable is read when Claude Code starts, so restart it after changing the
value.

## Branding

This is a TextQL-branded plugin. White-label deployments get no plugin until a
per-brand build exists. Their users can still add the server directly with
`claude mcp add --transport http refinery-mcp https://<host>/refinery-mcp`.

## Testing locally

Generate the plugin's skills first, then point Claude at the directory:

```sh
python3 integrations/agent-skills/select.py copy plugin-mcp integrations/refinery-mcp/claude-plugin/skills
claude plugin validate integrations/refinery-mcp/claude-plugin
claude --plugin-dir integrations/refinery-mcp/claude-plugin
```
