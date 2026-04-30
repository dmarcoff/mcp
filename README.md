# DMARKOFF MCP

**DMARC analytics inside your AI assistant.**

DMARKOFF MCP connects your DMARC monitoring data to AI assistants — Claude, ChatGPT, Cursor, Windsurf — through the [Model Context Protocol](https://modelcontextprotocol.io/). Instead of opening a dashboard and building filters, you ask a question and get an answer with real numbers behind it.

**Endpoint:** `https://mcp.dmarkoff.com/mcp`  
**Requires:** an active [DMARKOFF](https://dmarkoff.com) account and an API key from [app.dmarkoff.com/account/mcp](https://app.dmarkoff.com/account/mcp)

---

## What it looks like in practice

> *"Which domains dropped in compliance over the last 14 days?"*

The AI checks health across all your projects and surfaces the ones that changed — ranked by how much.

> *"Why is SPF failing on acme-transact.com? Who's the problem sender?"*

The AI checks return paths, finds the sending source (say, SendGrid using its own bounce domain instead of yours), and tells you what to fix.

> *"Anything unusual in email traffic compared to last week?"*

The AI compares yesterday's stats against a 14-day baseline — compliance rate, unknown sender volume, message counts. The kind of check most teams skip because setting it up manually takes too long.

---

## Quick connect

### Claude.ai

1. **Settings → Connectors → Add custom connector**
2. URL: `https://mcp.dmarkoff.com/mcp`
3. **Advanced settings:**
   - OAuth Client ID: `claude` (any value)
   - OAuth Client Secret: your DMARKOFF API key
4. Click **Add**, then **Connect**

Works on Free (1 custom connector limit), Pro, Max, Team, and Enterprise.

### Claude Desktop

Click **+** in the chat window → **Connectors → Manage Connectors → Add custom connector**, paste the URL, complete the OAuth flow.

Or edit the config file (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

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

### Claude Code

```bash
claude mcp add --transport http dmarkoff https://mcp.dmarkoff.com/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

### Cursor

**Settings → Tools & MCP → Add new MCP server**, select **Streamable HTTP**, enter the URL.

Or edit `~/.cursor/mcp.json`:

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

### ChatGPT (Business / Enterprise / Edu)

Enable Developer Mode, then go to **Settings → Apps → Create App**:

- MCP Server URL: `https://mcp.dmarkoff.com/mcp`
- Auth type: OAuth
- Client ID: `chatgpt`
- Client Secret: your DMARKOFF API key
- Authorization URL: `https://mcp.dmarkoff.com/oauth/authorize`
- Token URL: `https://mcp.dmarkoff.com/oauth/token`
- Scope: `dmarc:read`

### Windsurf

Edit `~/.codeium/windsurf/mcp_config.json`:

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

> Note: Windsurf uses `serverUrl`, not `url`.

→ **[Full connection guide for all clients](docs/connect.md)**

---

## Things to ask

```
Show me an overview of all my DMARC projects
```
```
Which domains have Critical severity right now?
```
```
Why is SPF failing on domain.com? Which senders are the problem?
```
```
Are there anomalies in email traffic over the last 24 hours?
```
```
Show me source details for domain.com grouped by ISP — only rows where SPF is failing
```

Live DNS checks — no account needed:

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

## Tools

**Analytics** (require API key)

| Tool | What it does |
|------|-------------|
| `projects_overview` | Summary across all projects: domain counts, compliance %, severity breakdown |
| `list_domains` | Filter domains by severity, search query, payment status; pagination |
| `get_domain_full_data` | Full diagnostic: DMARC/SPF/DKIM records, stats, timeline, trends — start here |
| `get_domain_stats` | Compliance percentages for a custom date range |
| `get_domain_timeline` | Day-by-day message counts and auth results |
| `get_domain_activity_health` | Per-day SPF/DKIM/DMARC severity vs 14-day baseline — detect when and why things changed |
| `get_domain_senders` | Top sending sources broken down by known ESPs, unknown, forwarded |
| `get_domain_source_details` | Per-record detail grouped by IP, ISP, hostname, or reporter with filters |
| `get_domain_anomaly_report` | Yesterday vs 14-day baseline — flags compliance drops and volume changes |
| `get_geo_sources` | Top 100 sending locations with compliance stats |
| `get_domain_detail` | DMARC record text, policy, health status per dimension, errors, name servers |
| `get_spf_records` | SPF return paths, lookup count, errors, pass/fail volume per source |
| `get_dkim_records` | DKIM selectors, signing domains, pass/fail volume per source |

**Live DNS** (no account needed)

| Tool | What it does |
|------|-------------|
| `dns_check_spf` | Current SPF record with parsed include tree and lookup count |
| `dns_check_dmarc` | Current DMARC policy from DNS |
| `dns_check_dkim` | DKIM public key for a given selector |
| `dns_check_txt` | All TXT records with type classification (spf, dmarc, mta-sts, bimi, other) |

All tools are read-only (`readOnlyHint: true` per MCP spec). No tool can modify or delete data.

→ **[Full tool reference with parameters](docs/tools.md)**

---

## Security

- **API key auth** — access is scoped to your DMARKOFF account
- **OAuth 2.1** (RFC 8414, RFC 9728) — compatible with standard MCP flows used by Claude.ai and ChatGPT
- **Project-level isolation** — the AI only sees what your API key has access to
- **Per-session rate limiting** — protection against accidental overuse
- **Read-only** — all tools are annotated as read-only; no data can be modified

---

## How this differs from other DMARC MCP tools

DNS checks tell you SPF is failing. DMARKOFF MCP tells you which specific sender is causing the failure, how many messages it affected over the last two weeks, and what the fix is. That data comes from your aggregate reports — the actual traffic — not just what's published in DNS.

---

## Troubleshooting

**Tools don't appear in the client**  
→ Confirm your client supports Streamable HTTP (not just SSE).  
→ Some clients need a full restart after config changes.

**401 Unauthorized**  
→ Check your API key at [app.dmarkoff.com/account/mcp](https://app.dmarkoff.com/account/mcp).

**"No projects found"**  
→ The API key needs access to at least one project. Log into [app.dmarkoff.com](https://app.dmarkoff.com) to verify.

**OAuth errors**  
→ The Client Secret field takes your DMARKOFF API key, not your account password.

**Need help?** Email [support@dmarkoff.com](mailto:support@dmarkoff.com) or [open an issue](https://github.com/dmarcoff/mcp/issues).
