# DMARKOFF MCP — Tool Reference

Every tool is **read-only** (`readOnlyHint: true`, `idempotentHint: true`) except `add_domain`, which creates domains in a project you own. Nothing is ever updated or deleted.

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

Returns per-project: domain count, severity breakdown (Critical/Warning/Healthy/Unknown counts), DMARC policy distribution (none/quarantine/reject/missing) and three compliance rates:

- `compliancePct` — compliant messages over all messages. Forwarded mail is excluded from the numerator but stays in the denominator, so this is the conservative floor the DMARKOFF app shows.
- `compliancePctNoForwards` — the same over the mail a DMARC policy can actually act on (forwarded mail dropped from both sides).
- `complianceKnownPct` — the same over known sources only: the health of your own sending infrastructure, unaffected by spoofing attempts. Quote this one when the question is "is our mail set up correctly".

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
- `previousPeriod`: the same headline numbers over the window before this one — the same number of days, ending the day before it starts

Key fields:
- `compliancePct` — compliant messages over all messages, the number the DMARKOFF app shows. Forwarded mail is excluded from the numerator but not from the denominator, so this reads lower than the raw pass rate in `get_domain_stats` whenever a domain has forwarding traffic. Different metrics, not a contradiction.
- `pctEligibleForPolicy` — the same rate over eligible messages only: compliant / (messages − forwarded). E.g. 99.66 means 99.66% of non-forwarded mail passes DMARC.
- `pctFromKnownSources` — % of messages from recognized sending sources
- `previousPeriod.changes.messagesPct` — volume change relative to the previous window. Absent when that window carried no mail; `changes.noPreviousData` says so, because there is no percentage of zero.
- `previousPeriod.changes.compliancePctPoints` — compliance change in percentage points. Every other field of `previousPeriod` is a count to compare directly.
- `errors[].fix` / `errors[].fixDescription` — on every DMARC, SPF and DKIM record error: the fix code and the remediation text the app shows for it.

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

Compare the last settled day against the 14 days before it, for one or more domains. Detects compliance drops, volume drops, days without reports, reports that stopped, reports for a parked domain, spikes in unknown senders and missing return-paths.

The day evaluated is **2 days ago, in UTC**. DMARC aggregate reports arrive a day or two after the traffic they describe, so yesterday is still filling in and is not judged — reading its shortfall as a drop would flag almost every domain. The baseline is the 14 days before the evaluated one, all of them settled.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domains` | string | yes | Comma-separated domain names (e.g. `example.com` or `a.com,b.com`) |

Returns, per domain:
- `evaluatedDate`, `baselineStart`, `baselineEnd` — the day judged and the 14-day baseline before it
- The evaluated day: `messages`, `forwardCount`, `unknownCount`, `dmarcPct`, `dkimPct`, `spfPct`
- Baseline quantiles, over the baseline days that had reports: `messagesQ5`, `messagesQ95`, `unknownQ95`, `dmarcQ5`, `dmarcQ95`, `spfQ95`, `dkimQ95`
- `baselineReportDays` — how many of the 14 baseline days had reports
- `sameWeekdayReportDays` — how many of the 2 baseline days with the evaluated day's weekday had reports
- `baselineMaxGapDays` — the longest gap in days between baseline days with reports, the first counted from the day before `baselineStart` (0 with none)
- `lastReportDate` — the newest report date, settled or not
- `missingReturnPaths` — ESPs sending with their own Return-Path on the evaluated day
- `paid`, `parked`, `userOverride`, `dmarcStatus` — the domain's state in the project, as `list_domains` shows it. Absent for a domain the project does not hold, which is judged on its reports alone. Every domain the project holds has a row, also without any report (`lastReportDate` 1970-01-01).

The domain's state comes first, and the summary names it in the domain's header (`parked`, `excluded from monitoring`, `unpaid`, `not monitored`):
- **Parked** — the domain is not meant to send mail, so it is never judged on volume or silence: no gap, stop, sparse sender or "no baseline yet". DMARKOFF processes one report a day for a parked domain, so `messages` is a floor ("at least N messages"). Mail on the evaluated day is judged by its DMARC result, which replaces the compliance, unknown-sender and return-path checks:
  - some of it passes DMARC — a warning: a forgotten or compromised sender is sending as a domain that should send none; check its sources with `get_domain_senders` and `get_domain_source_details`;
  - all of it fails — spoofing: a note while the domain's DMARC policy quarantines or rejects it, a warning when the policy in effect is `none` (including `quarantine` with `t=y`).

  Reports only after the evaluated day are a note: those days are not settled, the row holds no DMARC result for them, and each is judged once it is the evaluated day. Without any, it is a note — no mail expected — that names the baseline days that had reports, if any.
- **Excluded** (`userOverride` = `Excluded`) — DMARKOFF no longer processes the domain's reports. A single note, and nothing is judged, on a day with reports too.
- **Unpaid** (`paid` = false, not parked or excluded) — DMARKOFF processes at most one report a day for it, and none once it has been unpaid for more than 3 days. A note says so, and volume and silence are not judged; the compliance, unknown-sender and return-path checks of the day apply.
- **Not monitored** (`dmarcStatus` = `NotMonitored`) — DMARKOFF's reporting address was not in the domain's DMARC `rua` at the last check, so its reports may stop. It is judged as usual, and reports that stopped are a real warning.

What it flags:
- **Volume drop** — the evaluated day is more than 20% under the baseline's 5th percentile.
- **No reports** — the evaluated day has no reports at all. That is not counted as a drop: the mail may have stopped, or the reports never came. It is a warning (a possible reporting gap) only for a domain with reports on at least 7 of the 14 baseline days and on both baseline days with the same weekday (`sameWeekdayReportDays` = 2). Otherwise it is a note: a domain that sends on weekdays only is quiet every Saturday. `lastReportDate` shows whether reports have arrived since.
- **Reports stopped** — no reports for more than 7 days before the evaluated day, and for longer than any gap in the baseline (`baselineMaxGapDays`). A warning whether or not the domain is a sparse sender, so a daily sender that stops stays flagged rather than turning into a sparse sender a week later; also for a domain with no reports in the baseline at all.
- **Sparse sender** — reports on fewer than 7 of the 14 baseline days, such as weekly batches. Days without mail are normal for such a domain, so its volume is not judged and the report says so instead of flagging every quiet day, with the days since the last report.
- **No baseline yet** — no reports in the 14 baseline days, but reports on the evaluated day or after it: a new domain. Its volume is not judged, and it is not called a sparse sender.
- **Compliance drop** — `dmarcPct` under the baseline's 5th percentile (`dmarcQ5`).
- **Unknown senders** — `unknownCount` over the baseline's 95th percentile.

---

### `get_domain_senders`

Top sending sources broken down by source type (known/unknown/forward), with compliance stats per source. The top 20 of each type. Use before `get_domain_source_details` to identify which senders to investigate.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `source_type` | string | no | Filter: `known`, `unknown`, or `forward` (default: all) |
| `group_by` | string | no | Row identity: `provider` (default, one row per ESP/ISP) or `domain` (one row per sending domain) |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

`group_by=domain` splits a provider that hosts several senders into the hostnames actually sending — a cloud whose customers each get their own hostname, or unknown traffic spread over many hosts of the same provider. Row identity is in `groupKey`; `isp`, `brandDomain`, `sender` and `ipDomainName` describe it, and `domainsCount` / `providersCount` / `ipCount` say how many distinct sending domains, providers and IPs it covers. A high `domainsCount` in provider mode is the signal to switch.

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

Each row carries `sourceReason` and `sourceReasonText` — which evidence put the source into known, unknown or forward. The type alone does not say what to do; the reason does, and the actions are opposite:

| Reason | Type | What it means |
|--------|------|---------------|
| `dkim_spf_pass` | known | Fully authenticated — SPF and DKIM both aligned |
| `spf_aligned` | known | Authenticated by SPF |
| `rp_owned_spf_authorized` | known | Your own sending channel |
| `known_by_no_custom_rp` | known | Your ESP sending without a custom Return-Path — configure a custom MAIL FROM to align SPF |
| `rp_owned_esp_spf_unauthorized` | known | Your ESP on your custom Return-Path, but that Return-Path's SPF is missing the ESP — add the ESP's include |
| `fwd_dkim_dmarc_pass` | forward | Forwarded; your DKIM survived, so DMARC still passes |
| `fwd_gateway`, `fwd_gateway_spf_ok` | forward | Relayed by a security gateway |
| `fwd_mailbox`, `fwd_mailbox_spf_ok` | forward | Forwarded by a mailbox provider |
| `fwd_forwarder`, `fwd_forwarder_spf_ok` | forward | Relayed by a forwarding service |
| `fwd_po_hint` | forward | The receiver reported this as forwarded |
| `fwd_rp_foreign` | forward | Forwarded — the Return-Path belongs to someone else |
| `fwd_third_party_dkim` | forward | Forwarded — the relay added its own DKIM signature |
| `fwd_arc_pass` / `fwd_arc_fail` | forward | The receiver validated an ARC chain / an ARC chain was present but failed |
| `no_auth_pass` | unknown | Nothing authenticated — identify the sender, then authorise it or treat it as spoofing |
| `dmarc_not_pass` | unknown | Almost aligned: something authenticated, but not for your domain. Usually a Return-Path or selector fix |
| `dmarc_pass_but_not_ours` | unknown | DMARC passed, but this could not be attributed to your infrastructure |
| `relay_broke_dkim` | unknown | Probably your mail, modified by a relay — the signature no longer verifies |

Empty on reports parsed before the classifier started recording the reason.

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

### `get_policy_overrides`

Why messages were **not** handled the way your published DMARC policy says — the receiver reported a policy override in its aggregate report. Grouped by ISP, sender host, disposition and SPF/DKIM results, largest first (top 30).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

`override_type` values: `arc_pass` (the receiver validated an ARC chain and trusted the original authentication — typical for mail forwarded through Google or Microsoft), `arc_fail`, `forwarded`, `trusted_forwarder`, `sampled_out` (the `pct=` tag excluded the message from enforcement), `local_policy`, `other`. `arc_sealer` names the domain that sealed the chain when the report says so.

Use when the policy is `quarantine` or `reject` but mail still reached inboxes, or when forwarded traffic needs explaining. An empty result means no receiver reported an override in the period.

---

### `get_smtp_rejections`

Mail that receivers throttled or rejected at SMTP time, from the `Sender requirement failed: <code>` comments Google and a few others add to aggregate reports. Grouped by source type, ISP, SMTP code, disposition and auth results, largest first (top 20).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

`smtpCategory` names the reason behind the code: `rate_limit_spf_failure` (421-4.7.27), `rate_limit_dkim_failure` (421-4.7.30), `rate_limit_no_alignment` (421-4.7.32), `rate_limit_no_tls` (421-4.7.29), `blocked_ptr_or_rdns_mismatch` (550-5.7.25), `blocked_spf_failure` (550-5.7.27), `blocked_dkim_failure` (550-5.7.30), `blocked_other` (550-5.7.1), plus the general 421/550 buckets. Each row also carries the receiver's own wording, the reporters that saw it and the message count.

Use when you hear about Gmail bounces or "temporarily rate limited" errors. An empty result means no aggregate report carried such a comment — not that nothing bounced.

---

### `get_record_history`

Stored snapshots of a record, newest first: what it looked like at each check, with its health and errors at the time. Each snapshot lists `changedFromPrevious` — the fields that differ from the next older one, so a re-check that found nothing new is visibly empty.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `record_type` | string | yes | `dmarc`, `spf` or `dkim` |
| `return_path` | string | no | `spf` only: the return path to trace (default: the domain itself) — take it from `get_spf_records` |
| `dkim_domain` | string | no | `dkim` only: the signing domain (default: the domain itself) |
| `selector` | string | for dkim | `dkim` only: the selector — list them with `get_dkim_records` |
| `limit` | int | no | Snapshots to return, default 10, max 50 |

Use after `get_domain_activity_health` or `get_domain_timeline` showed *when* compliance dropped, to check whether a DNS change explains it. No snapshots means the record has not been checked yet or the return path / selector does not match a monitored record — not that it is absent from DNS.

---

### `get_spf_usage`

Which mechanisms of an SPF record actually carried mail — the tool for cleaning up a record that is over the 10-lookup limit or full of legacy includes. The record is resolved live, every sending IP that used the return path is matched against its `ip4`/`ip6` networks, and the traffic is laid onto the include tree.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `return_path` | string | no | The return path whose SPF record to evaluate (default: the domain itself) — take it from `get_spf_records` |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |
| `include_unused` | bool | no | Also list `ip4`/`ip6` entries inside includes that carried no mail (default: false) |

Returns:
- `tree` — the record with stats per mechanism; an include's stats are the sum of its parts. A mechanism without stats carried nothing in the period: a removal candidate, together with the DNS lookups it costs.
- `uncovered` / `uncoveredSources` — mail sent with this return path from IPs no `ip4`/`ip6` network of the record covers: senders the record does not authorise, or senders only a dynamic mechanism could match (`dynamicMechanisms` lists those — `%{...}` macros, `exists:`, `ptr`).
- `totals` — all mail seen with this return path in the period.

A mechanism whose `spfPass` is far below its `total` is being used by a sender with a different HELO or MAIL FROM, or the mail is forwarded.

---

## Live DNS tools (no project needed)

These tools query DNS directly in real time and work for any domain — including ones not in your DMARKOFF account. They still need your API key: every request to the server is authenticated. What they do not need is a monitored project or collected report data.

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

### `generate_dmarc_record`

Builds the DMARC TXT record a domain should publish. Reads what is currently in DNS and merges the requested tags into it — existing tags are preserved, nothing is silently dropped.

Pass `project_id` for a domain in a DMARKOFF project: the project's reporting address is added to `rua`, which is what makes DMARKOFF receive aggregate reports for it.

Returns the host to publish at (`_dmarc.<domain>`), the current record with its status (`published`, `inherited`, `none`, `lookupFailed`), and the recommended record. Nothing is published — you add the TXT record in your DNS zone, then verify with `dns_check_dmarc`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |
| `project_id` | integer | no | Adds the project's reporting address to `rua` |
| `p` / `sp` / `np` | string | no | Policy, subdomain policy, non-existent subdomain policy: `none`, `quarantine`, `reject` |
| `adkim` / `aspf` | string | no | Alignment: `s` (strict) or `r` (relaxed) |
| `fo` | string | no | Failure reporting options: `0`, `1`, `d`, `s` joined by `:` |
| `t` | string | no | RFC 9989 test mode: `y` or `n` |
| `psd` | string | no | Public suffix domain flag: `u`, `n`, `y` |
| `rua_emails` | string | no | Comma-separated addresses replacing the ones in the current record |
| `transfer_ruf` | string | no | Keep the `ruf` tag of the current record: `true` or `false` |

---

## Onboarding tools (require API key)

### `add_domain`

Adds one or more domains to a project so their DMARC reports start being collected. Up to 20 per call.

Requires **owner** access to the project — domains cannot be added to a project shared with you. Each non-parked domain consumes a paid domain slot on the account's plan; `parked` domains (zones that send no mail) do not.

Domains that already exist are reported as skipped, invalid ones as failed, and the rest are still added.

Registering the domain is only half the job: reports start flowing once the DMARC record is published. For a single domain the response carries the exact record; for several, a template plus the list of domains that already publish their own record — call `generate_dmarc_record` for each of those, since the template would drop the tags they already have.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | integer | yes | Project to add the domains to |
| `domains` | string[] | yes | 1 to 20 domain names |
| `parked` | boolean | no | Mark as parked: no mail expected, at most one report a day processed (so mail sent as the domain still shows), no paid slot |

---

## Field glossary

| Field | Meaning |
|-------|---------|
| `dmarcStatus` | Whether the domain is monitored in DMARKOFF — NOT whether a DMARC DNS record exists. Values: `Monitored`, `Pending`, `NotMonitored`. Use `dns_check_dmarc` for the actual DNS record. |
| `reportStatus` | Whether DMARC aggregate (rua) reports are arriving. `Receiving` = reports coming in; `NotReceiving` = no reports yet. |
| `reportProcessingStatus` | Whether the system is actively processing reports. `Processing` / `NotProcessing`. |
| `userOverride` | Manual override set in the UI: `Monitored`, `Excluded`, or `Invalid`. Domains with `Excluded` are hidden in `list_domains`. |
| `parked` | The domain is not meant to send mail: health metrics do not apply, and no reports is its normal. Mail reported for it is sent as a domain that should send none — spoofing when it fails DMARC, a forgotten or compromised sender when it passes — and is worth a look at its sources. DMARKOFF processes one report a day for a parked domain, so its volumes are a floor. |
| `policy_override` | Override applied by the receiving mail server (from the DMARC XML report). Not the same as the user's manual override. |
| `compliancePct` | Compliant messages over all messages — the number the DMARKOFF app shows. Forwarded mail is excluded from the numerator but stays in the denominator, so it reads lower than the raw pass rate in `get_domain_stats` when a domain has forwarding traffic. Different metrics, not a contradiction. |
| `pctEligibleForPolicy` | The same rate over eligible messages only: compliant / (messages − forwarded). |
| `dmarcFail` | Two definitions, both matching the app: in a stats summary it is `messages − compliant`, so forwarded mail counts as a failure and it stays consistent with `compliancePct`; in a timeline row it is `messages − compliant − forwarded`. The summary is therefore larger than the sum of the timeline column. For messages that actually failed the policy check, use `get_domain_stats`. |
| `fix` / `failedCheck` / `fixDescription` | On every record error: the fix code, the check that produced it, and the remediation text the app shows for that code. |
| `sourceReason` / `sourceReasonText` | Why a source was classified known / unknown / forward — see the table under `get_domain_source_details`. |
| `excludeFromHealth` | On SPF/DKIM records: the record was suppressed in the app. It is still listed with its own health and errors, but it does not feed the domain's severity — so a Critical record under a Healthy domain is expected, not a contradiction. |
| `missingReturnPaths` | ESPs sending with their own Return-Path instead of your domain — causes SPF alignment failure. Fix: configure a Custom Return-Path at the ESP. |

Which SPF/DKIM records are listed at all: those the domain's health is computed from — contributing and seen in reports within the last 14 days (for SPF, records flagged as always contributing are listed regardless of activity). A return path or selector missing from the list is inactive or non-contributing; that is not evidence it is absent from DNS — check with `dns_check_spf` / `dns_check_dkim`.

## Recommended workflow

```
projects_overview
  → list_domains (project_id, optional severity filter)
    → get_domain_full_data (domain, project_id)   ← read previousPeriod first: is this new?
      → get_domain_anomaly_report (for multi-domain anomaly check)
      → get_domain_activity_health (to understand when severity changed)
      → get_domain_senders (the sending landscape; group_by=domain to split a busy provider)
      → get_domain_source_details (to drill into specific senders)
      → get_geo_sources (geographic breakdown)
```

Then, by symptom:

```
policy published but not enforced   → get_policy_overrides
Gmail bounces or rate limiting      → get_smtp_rejections
"when did this change?"             → get_record_history (dmarc / spf / dkim)
SPF over the 10-lookup limit        → get_spf_usage
```

For quick DNS verification without a project: use `dns_check_spf`, `dns_check_dmarc`, `dns_check_dkim`, `dns_check_txt` directly.
