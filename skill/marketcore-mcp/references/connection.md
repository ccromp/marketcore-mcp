# Connection and Authentication

The MarketCore MCP server is a hosted, remote MCP server. There is nothing to install or run locally — your AI client connects directly to MarketCore's server over HTTPS.

---

## OAuth (recommended for interactive clients)

```
https://mcp.marketcore.ai
```

This is the primary URL. It uses OAuth 2.0 and dynamically negotiates Streamable HTTP or SSE based on what the client supports (defaults to Streamable HTTP, falls back to SSE).

Best for: Claude Desktop, ChatGPT, Cursor, VS Code, Claude.ai web — any client where the user can complete an interactive auth flow in a browser.

### Example client config

```json
{
  "mcpServers": {
    "marketcore": {
      "url": "https://mcp.marketcore.ai"
    }
  }
}
```

The first time the user connects, they'll be redirected to MarketCore to authenticate and grant the MCP client access. After that, the client holds a refresh token and reconnects silently.

---

## API token (non-interactive environments)

For headless / automated environments where the user can't complete an OAuth flow in a browser:

1. Generate an MCP API key at [Integration Settings](https://app.marketcore.ai/integration-settings).
2. Use one of these endpoints (the API key is embedded in the URL path):

| Transport | URL |
|---|---|
| SSE | `https://api.marketcore.ai/x2/mcp/<your-key>/mcp/sse` |
| Streamable HTTP | `https://api.marketcore.ai/x2/mcp/<your-key>/mcp/stream` |

Best for: scripts, CI/CD, server-side agents, headless automation.

### Treat API key URLs as secrets
The full URL is the credential — anyone with the URL can use the user's MarketCore account. Don't log or share these URLs.

---

## Verifying the connection

You do NOT need to call `get_current_user_info` proactively. Every MCP tool call is automatically scoped to the authenticated user's active team, so any successful call confirms auth is working. Only call `get_current_user_info` when the user explicitly asks about their profile, plan, usage, or active team — it returns:

- The user's `name` and `email`.
- The active `active_team_name` and `active_team_role`.
- The `subscription_status` and `plan_name`.
- Current `usage` against limits (AI credits, content count, canvas sessions).

If any other tool call fails with auth errors, the connection isn't established correctly. Direct the user back to their client's MCP config or to the integration settings page.

---

## Switching teams

A user can belong to multiple MarketCore teams. The MCP server operates against the **active team** at the time of the call. To switch teams, the user must change their active team in the MarketCore web app — the change is then reflected on the next MCP tool call.

If the user mentions team-related issues ("I see content I don't recognize"), check `get_current_user_info` for the active team name and confirm with the user that it's the team they expect.

---

## Rate limits and usage

`get_current_user_info` returns a `usage` object with the current period's consumption against plan limits:

- `ai_credits` / `ai_credits_max` — credits consumed this period.
- `deliverables` / `deliverables_max` — content created this period (note: field name is "deliverables" but call this "content" when speaking to the user).
- `canvas_sessions` / `canvas_sessions_max` — freeform AI sessions.
- `ai_updates` / `ai_updates_max` — refinement operations.

A `_max` of `0` means unlimited.

If the user is approaching limits, surface that proactively. If a generation fails with a quota error, suggest checking usage and either waiting for the next period or upgrading.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `get_current_user_info` 401 / unauthorized | Token expired or revoked | Re-auth via OAuth flow, or regenerate API key |
| Tools missing from client | Client connected before tools registered | Restart the MCP client |
| OAuth flow doesn't complete | Browser blocked redirect | Try a different browser; check pop-up blockers |
| API key URL returns 404 | Typo in key, or key revoked | Regenerate at Integration Settings |
| Calls succeed but operate on wrong data | Wrong active team | Switch active team in app, reconnect |

For anything not covered here, see the GitHub repo issues at the link in the main README.
