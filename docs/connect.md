# Connecting DMARKOFF MCP to AI Assistants

**MCP Endpoint:** `https://mcp.dmarkoff.com/mcp`  
**You will need:** an API key from your DMARKOFF account — [app.dmarkoff.com/account/mcp](https://app.dmarkoff.com/account/mcp)

> **Prerequisite:** An active DMARKOFF account with at least one monitored project. The AI assistant queries live data from your account — domains, DMARC reports, and analytics already collected by DMARKOFF.

---

## Claude.ai (web and mobile)

Available on Free (1 custom connector limit), Pro, Max, Team, and Enterprise plans.

1. Go to **Settings → Connectors**
2. Click **Add custom connector**
3. Enter the URL: `https://mcp.dmarkoff.com/mcp`
4. Click **Advanced settings** and fill in:
   - **OAuth Client ID** — any value (e.g. `claude`)
   - **OAuth Client Secret** — your DMARKOFF API key
5. Click **Add**, then **Connect**

Start a new chat and ask: *"Show me my projects."*

> The server implements OAuth 2.1 (RFC 8414, RFC 9728). Claude.ai discovers the authorization endpoints automatically via `/.well-known/oauth-authorization-server`.

---

## Claude Desktop

### Option 1: Via the Connectors UI (recommended)

1. In the chat window, click **+** → **Connectors → Manage Connectors**
2. Click **Add custom connector** and paste: `https://mcp.dmarkoff.com/mcp`
3. Complete the OAuth flow (same as Claude.ai above)

### Option 2: Via the config file

Open the config file:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add the following entry:

```json
{
  "mcpServers": {
    "dmarkoff": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.dmarkoff.com/mcp",
        "--header",
        "Authorization: Bearer YOUR_API_KEY"
      ]
    }
  }
}
```

Requires Node.js v18+. Quit and restart Claude Desktop completely after saving.

---

## Claude Code (CLI)

```bash
claude mcp add --transport http dmarkoff https://mcp.dmarkoff.com/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

If you already connected DMARKOFF via Claude.ai (Settings → Connectors), the server is automatically available in Claude Code — no additional setup needed.

To verify:
```bash
claude mcp list
```

---

## Cursor

### Option 1: Via Settings UI

1. Open **Cursor Settings → Tools & MCP**
2. Click **Add new MCP server**
3. Select type **Streamable HTTP**, fill in:
   - **Name:** `DMARKOFF`
   - **URL:** `https://mcp.dmarkoff.com/mcp`
4. Click **Save**

### Option 2: Via config file

Edit `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project root):

```json
{
  "mcpServers": {
    "dmarkoff": {
      "url": "https://mcp.dmarkoff.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

Restart Cursor. Tools appear in Agent mode (`Cmd+I` → Agent).

```
@dmarkoff Show all Critical domains in project 12345
```

---

## ChatGPT

Available on Business, Enterprise, and Edu plans with Developer Mode enabled. Plus/Pro via Developer Mode beta — check [OpenAI documentation](https://help.openai.com/en/articles/12584461-developer-mode-apps-and-full-mcp-connectors-in-chatgpt-beta) for current status.

1. Enable **Developer Mode**: Workspace Settings → Permissions & Roles → Developer Mode
2. Go to **Settings → Apps → Create App**
3. Set the connection parameters:
   - **MCP Server URL:** `https://mcp.dmarkoff.com/mcp`
   - **Auth type:** OAuth
   - **Client ID:** `chatgpt`
   - **Client Secret:** your DMARKOFF API key
   - **Authorization URL:** `https://mcp.dmarkoff.com/oauth/authorize`
   - **Token URL:** `https://mcp.dmarkoff.com/oauth/token`
   - **Scope:** `dmarc:read`
4. Publish the App for your workspace

---

## Windsurf (Codeium)

Edit `~/.codeium/windsurf/mcp_config.json` (create if it doesn't exist):

```json
{
  "mcpServers": {
    "dmarkoff": {
      "serverUrl": "https://mcp.dmarkoff.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

> Note: Windsurf uses `serverUrl` (not `url`). Environment variable interpolation is supported: `"Authorization": "Bearer ${env:DMARKOFF_API_KEY}"`.

Restart Windsurf. Tools appear in the Cascade panel.

---

## Continue.dev

MCP only works in **Agent Mode**.

Create `.continue/mcpServers/dmarkoff.yaml`:

```yaml
name: DMARKOFF MCP Server
version: 0.0.1
schema: v1

mcpServers:
  - name: DMARKOFF
    type: streamable-http
    url: https://mcp.dmarkoff.com/mcp
    headers:
      Authorization: Bearer YOUR_API_KEY
```

Or add to your main `config.yaml`:

```yaml
mcpServers:
  - name: DMARKOFF
    type: streamable-http
    url: https://mcp.dmarkoff.com/mcp
    headers:
      Authorization: Bearer YOUR_API_KEY
```

---

## Any MCP-compatible client

| Parameter | Value |
|-----------|-------|
| Transport | Streamable HTTP |
| URL | `https://mcp.dmarkoff.com/mcp` |
| Auth header | `Authorization: Bearer YOUR_API_KEY` |
| OAuth Issuer | `https://mcp.dmarkoff.com` |
| Authorize URL | `https://mcp.dmarkoff.com/oauth/authorize` |
| Token URL | `https://mcp.dmarkoff.com/oauth/token` |
| Scope | `dmarc:read` |

OAuth auto-discovery:
- `GET https://mcp.dmarkoff.com/.well-known/oauth-authorization-server`
- `GET https://mcp.dmarkoff.com/.well-known/oauth-protected-resource`

> **SSE is deprecated.** Most MCP clients have migrated to Streamable HTTP. If your client only supports SSE, use the `mcp-remote` utility as a proxy.

---

## First prompts to try

```
Show me an overview of all my DMARC projects
```
```
Which domains have Critical severity right now?
```
```
Check domain example.com — what's wrong with it?
```
```
Are there any anomalies in email traffic over the last 24 hours?
```
```
Why is SPF failing on domain.com? Which senders are the problem?
```
```
Show me source details for domain.com grouped by ISP — only rows where SPF is failing
```

Live DNS checks (no project needed):

```
Check the SPF record for example.com — are there too many DNS lookups?
```
```
What DMARC policy does example.com have right now?
```
```
Look up the DKIM key for selector "google" on example.com
```

---

## Troubleshooting

**Tools don't appear in the client**  
→ Confirm your client supports Streamable HTTP (not just SSE).  
→ Make sure `Authorization: Bearer ...` is being sent.  
→ Some clients require a full restart after config changes.

**401 Unauthorized**  
→ Check your API key at [app.dmarkoff.com/account/mcp](https://app.dmarkoff.com/account/mcp) — make sure it's active.

**"No projects found"**  
→ The API key needs access to at least one project. Log into [app.dmarkoff.com](https://app.dmarkoff.com) to verify.

**OAuth errors**  
→ The Client Secret field takes your DMARKOFF API key, not your account password.  
→ Try disconnecting and reconnecting the integration.

**Need help?** Email [support@dmarkoff.com](mailto:support@dmarkoff.com) or [open an issue](https://github.com/dmarcoff/mcp/issues).
