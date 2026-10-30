# DMARKOFF MCP

**DMARC analytics inside your AI assistant.**

DMARKOFF MCP connects your DMARC monitoring data to AI assistants — Claude, ChatGPT, Cursor, Windsurf — through the [Model Context Protocol](https://modelcontextprotocol.io/). Instead of opening a dashboard and building filters, you ask a question and get an answer with real numbers behind it.

**Endpoint:** `https://mcp.dmarkoff.com/mcp`  
**Requires:** an active [DMARKOFF](https://dmarkoff.com) account and an API key from [app.dmarkoff.com/account/mcp](https://app.dmarkoff.com/account/mcp)

---

## What it looks like in practice

> *"Which domains dropped in compliance over the last 14 days?"*

The AI checks health across all your projects and surfaces the ones that changed — ranked by how much.

> *"Why is SPF failing on example.com? Who's the problem sender?"*

The AI checks return paths, finds the sending source (say, SendGrid using its own bounce domain instead of yours), and tells you what to fix.

> *"Anything unusual in email traffic compared to last week?"*

The AI compares the last fully reported day against the 14 days before it — compliance rate, unknown sender volume, message counts. The kind of check most teams skip because setting it up manually takes too long.

> *"Can example.com go to p=reject yet?"*

The AI reads DMARKOFF's readiness plan for the domain and gives you its verdict. It names the senders that still need DKIM and where to set it up, the unknown sources you have to confirm are not yours, and the exact record to publish once nothing holds the step. Tell it which senders are yours, and it records that for the plan. Before it takes out a sender the plan counts as your own, it asks you.

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
- Scope: leave empty — the server issues unscoped tokens

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
Why is SPF failing on example.com? Which senders are the problem?
```
```
Are there anomalies in email traffic over the last 24 hours?
```
```
Show me source details for example.com grouped by ISP — only rows where SPF is failing
```
```
We publish p=reject on example.com — why did some mail still get delivered?
```
```
Gmail is throttling us on example.com. What are they actually complaining about?
```
```
Our SPF record is at 10 lookups — which includes still carry mail and which can go?
```
```
Compliance on example.com dropped last week. Did the DMARC or SPF record change?
```
```
Yesterday's Google report for example.com is missing. Is it late, or is something wrong?
```
```
Can example.com move to p=reject? What's holding it?
```
```
Klaviyo is ours, and the unknown host sending as example.com is not. Record that for the plan.
```

Live DNS checks — these work for any domain, monitored or not:

```
Check the SPF record for example.com — are there too many DNS lookups?
```
```
What DMARC policy does example.com have right now?
```
```
Look up the DKIM key for selector "google" on example.com
```

Onboarding a new domain — the assistant registers it and hands you the exact TXT record to publish:

```
Add example.com to my project and tell me what to put in DNS
```
```
We already have DMARC on example.com — add our reporting address without touching the rest of the record
```
```
I published the record for example.com. Will its reports reach my project?
```

---

## Tools

**Analytics** (require API key)

| Tool | What it does |
|------|-------------|
| `projects_overview` | Summary across all projects: domain counts, three compliance rates, severity breakdown |
| `list_domains` | Filter domains by severity, search query, payment status; pagination |
| `get_domain_full_data` | Full diagnostic: DMARC/SPF/DKIM records, stats, timeline, trends, and the same numbers over the previous period — start here |
| `get_domain_stats` | Compliance percentages for a custom date range |
| `get_domain_timeline` | Day-by-day message counts and auth results |
| `get_domain_activity_health` | Per-day SPF/DKIM/DMARC severity vs 14-day baseline — detect when and why things changed |
| `get_domain_senders` | Top sending sources broken down by known ESPs, unknown, forwarded — per provider or per sending domain, with each sending domain's place in the readiness plan |
| `get_domain_source_details` | Per-record detail grouped by IP, ISP, hostname, reporter or individual report, with filters — and how long each reporter takes to send its reports |
| `get_domain_anomaly_report` | Last settled day (2 days ago, UTC) vs the 14 days before it — flags compliance drops, volume drops, days without reports, reports that stopped and mail sent as a parked domain; excluded and unpaid domains are not judged on volume |
| `get_geo_sources` | Top 100 sending locations with compliance stats |
| `get_domain_detail` | DMARC record text, policy, health status per dimension, errors, name servers |
| `get_spf_records` | SPF return paths, lookup count, errors, pass/fail volume per source |
| `get_dkim_records` | DKIM selectors, signing domains, pass/fail volume per source |
| `get_record_history` | Past versions of a DMARC, SPF or DKIM record with what changed between checks — when did this break |
| `get_policy_overrides` | Why receivers did not enforce your policy: ARC, forwarding, sampling, local rules |
| `get_smtp_rejections` | Mail throttled or rejected at SMTP time, by reason — Gmail's 421/550 codes explained |
| `get_spf_usage` | Which SPF mechanisms actually carry mail, and which senders your record misses — for lookup-limit clean-ups |
| `get_enforcement_status` | DMARKOFF's readiness plan for a domain: whether it can move to quarantine or reject (or first take `pct` off), what holds it, which sources to confirm, and the record to publish |

**Live DNS** (API key, but no monitored project)

| Tool | What it does |
|------|-------------|
| `dns_check_spf` | Current SPF record, what receivers evaluate it to (`ok`, `permerror` and so on), parsed include tree and lookup count |
| `dns_check_dmarc` | Current DMARC policy from DNS. With a project, whether the record sends reports to it |
| `dns_check_dkim` | DKIM public key for a given selector |
| `dns_check_txt` | All TXT records with type classification — including `spf-ignored`, a record that looks like SPF but receivers do not read |
| `generate_dmarc_record` | The DMARC record a domain should publish, merged on top of what is already in DNS and repaired where the checker rejects it, keeping the policy in force, with a list of what changes. With a project, adds your reporting address to `rua` |

**Onboarding** (requires API key and owner access to the project)

| Tool | What it does |
|------|-------------|
| `add_domain` | Adds up to 20 domains to a project and returns the DMARC record to publish for each. Safe to repeat: it says where a domain already is |

**Source decisions** (requires API key and owner access to the project, or a shared project whose owner lets you manage sources)

| Tool | What it does |
|------|-------------|
| `set_source_decision` | Records your call on a sending source (ours, not ours, or ignore) for the readiness plan, or clears it. Asks before taking out a source the plan counts among your own senders |

Every tool is read-only (`readOnlyHint: true` per MCP spec) except two. `add_domain` creates domains. It never updates or deletes anything, and each non-parked domain it adds consumes a paid domain slot on your plan. `set_source_decision` saves, changes or clears your decision on one sending source of a domain, or confirms that a source you took out stays out. It cannot add or remove domains, change your DNS or touch report data. Taking out one of the plan's own senders is not saved until you confirm, and each change is logged with the API key as its author. The [tool reference](docs/tools.md#set_source_decision) lists which sources that check covers.

→ **[Full tool reference with parameters](docs/tools.md)**

---

## Security

- **API key auth** — access is scoped to your DMARKOFF account
- **OAuth 2.1** (RFC 8414, RFC 9728) — compatible with standard MCP flows used by Claude.ai and ChatGPT
- **Project-level isolation** — the AI only sees what your API key has access to
- **Per-API-key rate limiting** — protection against accidental overuse
- **Two tools write** — `add_domain` creates domains in a project you own. `set_source_decision` records your decision on a sending source in a project you own, or one shared with you to manage sources. Its description tells the assistant to use it only at your request, and the server does not take out a source the plan counts among your own senders until you confirm. Every other tool is read-only. No tool deletes domains or report data, and none changes your DNS

---

## How this differs from other DMARC MCP tools

DNS checks tell you SPF is failing. DMARKOFF MCP tells you which specific sender is causing the failure, how many messages it affected over the last two weeks, and what the fix is. That data comes from your aggregate reports — the actual traffic — not just what's published in DNS.

It also answers the questions that come after: whether the problem is new (every domain report carries the previous period beside the current one), what a record looked like before someone edited it, why a receiver ignored the policy you published, and which parts of an oversized SPF record are still earning their DNS lookups.

When you want to tighten the policy, it reads DMARKOFF's own verdict on whether a domain can go to quarantine or reject. That verdict comes from the plan behind the policy recommendation in the DMARKOFF app and the policy-upgrade email, so the assistant and DMARKOFF give you the same answer.

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

**Can't record a decision on a sender**  
→ Recording decisions needs owner access to the project, or a shared project whose owner lets you manage sources. Reading the readiness plan works in shared projects too.

**Need help?** Email [support@mail.dmarkoff.com](mailto:support@mail.dmarkoff.com) or [open an issue](https://github.com/dmarcoff/mcp/issues).
