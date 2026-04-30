# DMARKOFF MCP — Tool Reference

All tools are **read-only** (`readOnlyHint: true`, `idempotentHint: true`). No tool modifies or deletes data.

Default date range for analytics tools: last 14 days (`today − 14 days → yesterday`). Pass `stats_period_start` / `stats_period_end` to override.

---

## Analytics tools (require API key)

### `projects_overview`

**Start here.** Overview of all projects your API key has access to — severity breakdown, domain counts, compliance %, DMARC policy distribution.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days ago) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` (default: yesterday) |
| `page` | int | no | Page number, default 1 |
| `limit` | int | no | Items per page, default 20, max 100 |

Returns per-project: domain count, severity breakdown (Critical/Warning/Healthy/Unknown counts), compliance %, DMARC policy distribution (none/quarantine/reject/missing).

---

### `list_domains`

List domains in a project with filtering and pagination. Returns per-domain health status, DMARC policy, and user override.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `severity` | string | no | Filter by severity: `Unknown`, `Healthy`, `Info`, `Warning`, `Critical` |
| `query` | string | no | Search by domain name |
| `paid` | bool | no | `true` = paid only, `false` = unpaid only, omit = all |
| `page` | int | no | Page number, default 1 |

---

### `get_domain_full_data`

**Primary diagnostic tool.** Everything about a domain in one call. Use this instead of calling narrow tools individually.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name (e.g. `example.com`) |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days ago) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` (default: yesterday) |

Returns:
- Base info: severity, DMARC status, policy, report status, user override
- Health status per dimension: DMARC record, SPF record, DKIM record, SPF alignment, DKIM alignment, DMARC compliance, message volume
- Stats summary: compliance %, source counts, fail counts
- SPF records: return paths, record text, lookup count, errors, pass/fail volume per source
- DKIM records: selectors, signing domains, record text, errors, pass/fail volume
- Daily timeline: per-day volume and compliance breakdown
- Statistical quantiles (5th, 90th, 95th percentile) for volume and failure rates — use as anomaly baselines

Key fields:
- `pctEligibleForPolicy` — % of messages subject to DMARC enforcement (excludes forwarded mail)
- `pctFromKnownSources` — % of messages from recognized sending sources

---

### `get_domain_detail`

Lightweight domain info: DMARC record text, policy, health status per dimension, DNS errors, name servers, monitoring status. Does NOT include SPF/DKIM records, timeline, or stats.

Use when you only need to check DMARC configuration without full analytics.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |

---

### `get_domain_activity_health`

Daily activity health for the last N days — per-day breakdown of SPF alignment, DKIM alignment, DMARC compliance, and message volume severity. Each metric includes a quantile (0–1 scale vs 14-day rolling baseline) and actual value.

Use after `get_domain_full_data` to understand *when* and *why* severity changed.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `days` | int | no | Number of days (default: 7, max: 30) |

---

### `get_domain_stats`

DMARC compliance statistics for a period: totals, SPF/DKIM/DMARC pass/fail counts, compliance %.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days ago) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` (default: yesterday) |

---

### `get_domain_timeline`

Day-by-day timeline: message volume, compliant/failed/forwarded/unknown counts, statistical quantiles.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days ago) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` (default: yesterday) |

---

### `get_domain_anomaly_report`

Compare yesterday's metrics against 14-day statistical baselines for one or more domains. Detects compliance drops, volume anomalies, and missing return-paths.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domains` | string | yes | Comma-separated domain names (e.g. `example.com` or `a.com,b.com`) |

---

### `get_domain_senders`

Top sending sources grouped by ISP/provider, broken down by source type (known/unknown/forward). Returns compliance stats per source. Use before `get_domain_source_details` to identify which senders to investigate.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `source_type` | string | no | Filter: `known`, `unknown`, or `forward` (default: all) |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

---

### `get_domain_source_details`

Per-record source view with flexible grouping and filtering. Use to drill into specific senders, isolate authentication failures, or investigate by provider/IP/reporter.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `group_by` | string | no | `isp` (default) / `ip` / `host` / `reporter` |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |
| `page` | int | no | Page number, default 1 |
| `limit` | int | no | Results per page, default 25, max 100 |
| `source_ip` | string | no | Filter by sending IP |
| `isp` | string | no | Filter by ISP/provider name |
| `ip_domain_name` | string | no | Filter by hostname |
| `eval_spf` | string | no | Filter by SPF evaluation: `pass` or `fail` |
| `eval_dkim` | string | no | Filter by DKIM evaluation: `pass` or `fail` |
| `eval_dmarc` | string | no | Filter by DMARC evaluation: `pass` or `fail` |
| `source_type` | string | no | Filter by source type: `known` / `unknown` / `forward` |
| `disposition` | string | no | Filter by disposition: `none` / `quarantine` / `reject` |
| `dkim_domain` | string | no | Filter by DKIM signing domain |
| `dkim_selector` | string | no | Filter by DKIM selector |
| `spf_domain` | string | no | Filter by SPF auth domain (return-path domain) |
| `problems_only` | bool | no | ISP grouping only: show only rows with auth failures |

Note: with `group_by=isp`, the same provider may appear multiple times with different countries (one row per provider+country).

---

### `get_geo_sources`

Top 100 sending locations grouped by country, with compliance stats per location.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `source_type` | string | yes | `known`, `unknown`, or `forward` |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

---

### `get_spf_records`

SPF records from your DMARC data — return paths with recent activity, record text, DNS-lookup count, errors, pass/fail volume per source. Included in `get_domain_full_data`; call this separately only when you need this slice specifically.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |

---

### `get_dkim_records`

DKIM records from your DMARC data — selectors, signing domains, record text, errors, pass/fail volume per source. Included in `get_domain_full_data`; call this separately only when you need this slice specifically.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |

---

## Live DNS tools (no account needed)

These tools query DNS directly in real time. They work for any domain — including ones not in your DMARKOFF account.

---

### `dns_check_spf`

Live SPF lookup: record text, parsed include tree, DNS-lookup count (flags if over the 10-lookup limit), errors.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name (e.g. `example.com`) |

---

### `dns_check_dmarc`

Live DMARC lookup: record text, parsed policy (none/quarantine/reject), errors.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |

---

### `dns_check_dkim`

Live DKIM key lookup for a specific selector: queries `<selector>._domainkey.<domain>`, returns key record, key length, errors.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |
| `selector` | string | yes | DKIM selector (e.g. `google`, `selector1`) |

---

### `dns_check_txt`

All TXT records for a domain with TTL and type classification: `spf`, `dmarc`, `mta-sts`, `tlsrpt`, `bimi`, `other`.

Useful for detecting duplicate SPF records (multiple `v=spf1` entries cause SPF to fail) or verifying a newly published record is visible.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |

---

## Field glossary

| Field | Meaning |
|-------|---------|
| `dmarcStatus` | Whether the domain is monitored in DMARKOFF — NOT whether a DMARC DNS record exists. Values: `Monitored`, `Pending`, `NotMonitored`. Use `dns_check_dmarc` for the actual DNS record. |
| `reportStatus` | Whether DMARC aggregate (rua) reports are arriving. `Receiving` = reports coming in; `NotReceiving` = no reports yet. |
| `reportProcessingStatus` | Whether the system is actively processing reports. `Processing` / `NotProcessing`. |
| `userOverride` | Manual override set in the UI: `Monitored`, `Excluded`, or `Invalid`. Domains with `Excluded` are hidden in `list_domains`. |
| `parked` | Domain is inactive. High unknown sender % and low volume are expected — not actionable. |
| `policy_override` | Override applied by the receiving mail server (from the DMARC XML report). Not the same as the user's manual override. |
| `pctEligibleForPolicy` | Compliance % excluding forwarded messages not subject to DMARC enforcement. |
| `activityState` | On SPF/DKIM records: `active` / `stale` / `dormant` based on recent report activity. |
| `missingReturnPaths` | ESPs sending with their own Return-Path instead of your domain — causes SPF alignment failure. Fix: configure a Custom Return-Path at the ESP. |

## Recommended workflow

```
projects_overview
  → list_domains (project_id, optional severity filter)
    → get_domain_full_data (domain, project_id)
      → get_domain_anomaly_report (for multi-domain anomaly check)
      → get_domain_activity_health (to understand when severity changed)
      → get_domain_source_details (to drill into specific senders)
      → get_geo_sources (geographic breakdown)
```

For quick DNS verification without a project: use `dns_check_spf`, `dns_check_dmarc`, `dns_check_dkim`, `dns_check_txt` directly.
