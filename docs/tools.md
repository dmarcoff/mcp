# DMARKOFF MCP — Tool Reference

Every tool is **read-only** (`readOnlyHint: true`, `idempotentHint: true`) except two, annotated `readOnlyHint: false` and `destructiveHint: false`. `add_domain` creates domains in a project you own. `set_source_decision` saves, changes or clears your decision on one sending source of a domain, or confirms that a source you took out stays out. No tool deletes a domain or report data, and none changes your DNS.

Default period for analytics tools (`projects_overview` has its own, below): the 15 days ending today (`today − 14 days → today`, UTC, both ends inclusive). Pass `stats_period_start` / `stats_period_end` to override: a start equal to the end reads that one day, a missing start is 14 days before the end, and an end after today reads up to today. A date that does not parse, or a start after the end, is an error. Every answer of a tool that takes a period carries `period` (`start`, `end`, `days`, and `endClamped` when the end asked for was after today). A report is filed under the UTC day its window begins, and reports arrive a day or two after the mail, so the newest days of any period are still filling.

Per-domain tools that take a period also return `coverage`: `firstReport` and `lastReport`, the span of days the project has reports for the domain (reports are kept 18 months), or `noReports`. A header note says when the period starts before the first report (DMARKOFF did not see the domain then), runs past the last one (reports stopped, or no mail was sent), or includes days whose reports are still arriving (their zeros are not days without mail).

Tools that read or add a project's domains take the name lower-case and without a trailing dot.

---

## Analytics tools (require API key)

### `projects_overview`

**Start here.** Overview of all projects your API key has access to — severity breakdown, domain counts, compliance %, DMARC policy distribution.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days before `stats_period_end`, or 14 days ago) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD`, inclusive (default: messages and compliance end 2 days ago, `missedReports` yesterday) |
| `page` | int | no | Page number, default 1 |
| `limit` | int | no | Items per page, default 20, max 100 |

The answer carries `period` and `missedReportsPeriod`, the days read for messages and compliance and for `missedReports`: one period when `stats_period_end` is given. Each call reads the key's projects anew (at most every 10 seconds per key), so a project just created or shared in the web app shows up.

Returns per-project: domain count, severity breakdown (Critical/Warning/Healthy/Unknown counts), DMARC policy distribution (none/quarantine/reject/missing), `dmarcAREmail` and, when set, `monitoringEmail` — an MSP mailbox published in `rua` instead, which forwards the reports to DMARKOFF — and three compliance rates:

- `compliancePct` — compliant messages over all messages. Forwarded mail is excluded from the numerator but stays in the denominator, so this is the conservative floor the DMARKOFF app shows.
- `compliancePctNoForwards` — the same over the mail a DMARC policy can actually act on (forwarded mail dropped from both sides). Quote this one when the question is "is our mail set up correctly", and read the `sourceReason` of the unknown rows in `get_domain_source_details` for why mail failed.
- `complianceKnownPct` — the same over known sources only: the DMARC compliance of known-source mail. Mail is classified known only when it passes DMARC (on reports parsed since December 2025), so this reads 100 whenever the period has known mail; your own mail that fails DMARC is classified unknown. It is no measure of your own senders.

Also per project:
- `noReports`, `noEligibleMail`, `noKnownMail` — no report in the period, all of its mail forwarded, or none of it from known sources: the rates affected are 0 by convention, not a measured 0%.
- `missedReports` — reports DMARKOFF dropped on arrival, per reason: `unpaid_domain`, `excluded`, `parked`, `auto_add_disabled` (a domain the project did not hold while adding domains from reports was off). Counted on the UTC day a report arrived, by the domain's state then.
- `nextStep` — on a project without domains: add them with `add_domain` and publish the record it returns (in a shared project, only its owner can).

---

### `list_domains`

List domains in a project with filtering and pagination. Returns per-domain health status, DMARC policy, `dmarcStatus`, `reportStatus` and user override; no message counts or compliance rates. Excluded domains are not listed. An empty page says why in its header: the project has no domains yet (the next step is `add_domain`), no domain matches the filters, or every domain is excluded.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `severity` | string | no | Filter by severity: `Unknown`, `Healthy`, `Info`, `Warning`, `Critical` |
| `query` | string | no | Search by domain name |
| `paid` | bool | no | `true` = paid only, `false` = unpaid only, omit = all |
| `page` | int | no | Page number, default 1 |
| `limit` | int | no | Items per page, default 20, max 100 |

---

### `get_domain_full_data`

**Primary diagnostic tool.** Everything about a domain in one call. Use this instead of calling narrow tools individually.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name (e.g. `example.com`) |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days before the end) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD`, inclusive (default: today) |

Returns:
- `period` and `coverage` (see the top of this page)
- Base info: severity, DMARC status, policy, report status, user override
- Health status per dimension: DMARC record, SPF record, DKIM record, SPF alignment, DKIM alignment, DMARC compliance, message volume
- Stats summary: compliance %, source counts, fail counts
- SPF records: return paths, record text, lookup count, errors, pass/fail volume over the period
- DKIM records: selectors, signing domains, record text, errors, pass/fail volume over the period. A key's `total` counts every message carrying that signature, wherever it sits among the message's signatures
- Daily timeline: per-day volume and compliance breakdown. Over 60 days, each entry is a bucket of ceil(days/60) days. Days without mail and an entry dated today are left out
- Statistical quantiles (5th, 90th, 95th percentile) of the daily volume, unknown, dmarcFail, dkimFail and spfFail counts — use as anomaly baselines. They count every day of the period, a day without reports as 0, yesterday and today included; for a settled baseline, end the period two days before today
- `previousPeriod`: the same headline numbers over the window before this one — the same number of days, ending the day before it starts
- `monitoring` — only for a domain that is NotMonitored, Pending, excluded or unpaid: `status`, `expectedAddress` (what `rua` must carry), `recordHealthAsOf` (the day DMARKOFF last read the DMARC record) and a `note` with the cause and the fix in the web app ("Reinitiate monitoring", "Convert to paid" or "Include in monitoring"). Record health of such a domain is not current, so a Healthy severity does not mean it is fine.

A domain the project does not hold is refused with `domain not found`.

Key fields:
- `compliancePct` — compliant messages over all messages, the number the DMARKOFF app shows. Forwarded mail is excluded from the numerator but not from the denominator, so this reads lower than the raw pass rate in `get_domain_stats` whenever a domain has forwarding traffic. Different metrics, not a contradiction.
- `pctEligibleForPolicy` — the same rate over eligible messages only: compliant / (messages − forwarded). E.g. 99.66 means 99.66% of non-forwarded mail passes DMARC. When all of the period's mail was forwarded it is 0 by convention, and `noEligibleMail` is true (`previousPeriod.noEligibleMail` for that window).
- `pctFromKnownSources` — not the share of mail from known sources: the DMARC compliance of known-source mail, forwarded mail left out (`complianceKnownPct` of `projects_overview`, for one domain). It reads 100 whenever the period has known mail, so it is no measure of your own senders.
- `previousPeriod.changes.messagesPct` — volume change relative to the previous window. Absent when that window carried no mail; `changes.noPreviousData` says so, because there is no percentage of zero.
- `previousPeriod.changes.beforeFirstReport` / `partlyBeforeFirstReport` — the comparison window ends before `coverage.firstReport`, or starts before it: there is nothing, or only part of the window, to compare with, so `messagesPct` is not growth.
- `previousPeriod.changes.compliancePctPoints` — compliance change in percentage points. Every other field of `previousPeriod` is a count to compare directly.
- `errors[].fix` / `errors[].fixDescription` — on every DMARC, SPF and DKIM record error: the fix code and the remediation text the app shows for it.
- `errors[].failedCheck` / `errors[].checkDescription` — the check that produced the error and what it means, as the app explains it; `checkDescription` is on the first error of each check in a list.

---

### `get_domain_detail`

Lightweight domain info: DMARC record text, policy, health status per dimension, DNS errors, name servers, monitoring status. Does NOT include SPF/DKIM records, timeline, or stats. For a domain that is NotMonitored, Pending, excluded or unpaid it carries the same `monitoring` block as `get_domain_full_data`. A domain the project does not hold is refused with `domain not found`.

Use when you only need to check DMARC configuration without full analytics.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |

---

### `get_domain_activity_health`

DMARKOFF's daily check of SPF alignment, DKIM alignment, DMARC compliance and message volume, the last N checks. Each entry is one check: `date` is the UTC day it ran, and its figures are those of the report day before, as of the check — reports that arrive later are not in it, and the entry is never updated. Each metric has a `severity`, a `value` and a `quantile`: the baseline the check compared `value` with, in `value`'s units (not a 0–1 rank) — a percentile of the same daily figure over the report days from 14 to 2 days before the check. For SPF, DKIM and DMARC that is the 95th percentile of the daily percent. For volume, `direction` says which: `up` the 95th percentile of the daily counts, `down` and `none` (no message reported for the day) the 5th.

Use after `get_domain_full_data` to understand *when* and *why* severity changed.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `days` | int | no | Number of checks (default: 7, max: 30) |

---

### `get_domain_stats`

DMARC compliance statistics for a period: totals, SPF/DKIM/DMARC pass/fail counts, compliance %. The answer is `{period, coverage, stats}`. `compliantPct` here is the raw DMARC pass rate of the report records, forwarded mail that passed included, so it reads higher than `compliancePct` for a domain with forwarding.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days before the end) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD`, inclusive (default: today) |

---

### `get_domain_timeline`

Day-by-day timeline: message volume, compliant/failed/forwarded/unknown counts. Every day of the period has an entry, zeros for a day without reports; yesterday's and today's counts can still rise.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` (default: 14 days before the end) |
| `stats_period_end` | string | no | End date `YYYY-MM-DD`, inclusive (default: today) |

---

### `get_domain_anomaly_report`

Compare the last settled day against the 14 days before it, for one or more domains. Detects compliance drops, volume drops, days without reports, reports that stopped, reports for a parked domain, spikes in unknown senders and missing return-paths.

The day evaluated is **2 days ago, in UTC**. DMARC aggregate reports arrive a day or two after the traffic they describe, so yesterday is still filling in and is not judged — reading its shortfall as a drop would flag almost every domain. The baseline is the 14 days before the evaluated one, all of them settled.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domains` | string | yes | Comma-separated domain names (e.g. `example.com` or `example.com,example.net`) |

Returns, per domain:
- `evaluatedDate`, `baselineStart`, `baselineEnd` — the day judged and the 14-day baseline before it
- The evaluated day: `messages`, `forwardCount`, `unknownCount`, `dmarcPct`, `dkimPct`, `spfPct`
- Baseline quantiles, over the baseline days that had reports: `messagesQ5`, `messagesQ95`, `unknownQ95`, `dmarcQ5`, `dmarcQ95`, `spfQ95`, `dkimQ95`
- `baselineReportDays` — how many of the 14 baseline days had reports
- `sameWeekdayReportDays` — how many of the 2 baseline days with the evaluated day's weekday had reports
- `baselineMaxGapDays` — the longest gap in days between baseline days with reports, the first counted from the day before `baselineStart`, or from the day before `firstReportDate` when that is later (0 with none)
- `totalMessages`, `totalForward`, `totalUnknown` — totals over the baseline
- `lastReportDate` — the newest report date, settled or not; `firstReportDate` — the oldest one the project holds (1970-01-01 with none)
- `missingReturnPaths` — ESPs sending with their own Return-Path on the evaluated day
- `paid`, `parked`, `userOverride`, `dmarcStatus` — the domain's state in the project, as `list_domains` shows it. Absent for a domain the project does not hold, which is judged on its reports alone. Every domain the project holds has a row, also without any report (`lastReportDate` 1970-01-01).

The domain's state comes first, and the summary names it in the domain's header (`parked`, `excluded from monitoring`, `unpaid`, `not monitored`):
- **Parked** — the domain is not meant to send mail, so it is never judged on volume or silence: no gap, stop, sparse sender or "no baseline yet". DMARKOFF processes one report a day for a parked domain, so `messages` is a floor ("at least N messages"). Mail on the evaluated day is judged by its DMARC result, which replaces the compliance, unknown-sender and return-path checks:
  - some of it passes DMARC — a warning: a forgotten or compromised sender is sending as a domain that should send none; check its sources with `get_domain_senders` and `get_domain_source_details`;
  - all of it fails — spoofing: a note while the domain's DMARC policy quarantines or rejects it, a warning when the policy in effect is `none` (including `quarantine` with `t=y`).

  Reports only after the evaluated day are a note: those days are not settled, the row holds no DMARC result for them, and each is judged once it is the evaluated day. Without any, it is a note — no mail expected — that names the baseline days that had reports, if any.
- **Excluded** (`userOverride` = `Excluded`) — DMARKOFF no longer processes the domain's reports. A single note, and nothing is judged, on a day with reports too.
- **Unpaid** (`paid` = false, not parked or excluded) — DMARKOFF processes at most one report a day for it for 3 days after it became unpaid (or was added unpaid), then none. A note says so, and volume and silence are not judged; the compliance, unknown-sender and return-path checks of the day apply.
- **Not monitored** (`dmarcStatus` = `NotMonitored`) — the project's address (its `monitoringEmail` when set, DMARKOFF's reporting address otherwise) was not in the domain's DMARC `rua` at the last check, so its reports may stop. It is judged as usual, and reports that stopped are a real warning.

What it flags:
- **Volume drop** — the evaluated day is more than 20% under the baseline's 5th percentile.
- **No reports** — the evaluated day has no reports at all. That is not counted as a drop: the mail may have stopped, or the reports never came. It is a warning (a possible reporting gap) only for a domain with reports on at least 7 of the 14 baseline days and on both baseline days with the same weekday (`sameWeekdayReportDays` = 2), or for a new domain as below. Otherwise it is a note: a domain that sends on weekdays only is quiet every Saturday. `lastReportDate` shows whether reports have arrived since.
- **Reports stopped** — no reports for more than 7 days before the evaluated day, and for longer than any gap in the baseline (`baselineMaxGapDays`). A warning whether or not the domain is a sparse sender, so a daily sender that stops stays flagged rather than turning into a sparse sender a week later; also for a domain with no reports in the baseline at all.
- **Sparse sender** — reports on fewer than 7 of the 14 baseline days, such as weekly batches. Days without mail are normal for such a domain, so its volume is not judged and the report says so instead of flagging every quiet day, with the days since the last report.
- **New domain** — its first report (`firstReportDate`) falls inside the baseline. The baseline days before it are not days without reports, so its report days are counted from that report: with 1–6 of them it is a new domain rather than a sparse sender, and its volume is not judged. With 7 or more, a day without reports is a warning when it had reports on the one baseline day of the same weekday since its first report.
- **No baseline yet** — no reports in the 14 baseline days, but reports on the evaluated day or after it: a new domain. Its volume is not judged, and it is not called a sparse sender.
- **Compliance drop** — `dmarcPct` under the baseline's 5th percentile (`dmarcQ5`).
- **Unknown senders** — `unknownCount` over the baseline's 95th percentile: strangers or spoofing, or the domain's own mail failing DMARC (the `sourceReason` of the unknown rows in `get_domain_source_details` tells which).

---

### `get_domain_senders`

Top sending sources broken down by source type (known/unknown/forward), with compliance stats per source. The top 20 of each type. Use before `get_domain_source_details` to identify which senders to investigate.

Mail is classified known only when it passes DMARC. Unknown holds strangers and spoofing, but also the domain's own mail that fails DMARC; the `sourceReason` of the rows in `get_domain_source_details` tells them apart. The type is decided per report record, so one provider can appear under two types.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `source_type` | string | no | Filter: `known`, `unknown`, or `forward`; omit it (or pass `all`) for every type. Any other value is refused |
| `group_by` | string | no | Row identity: `provider` (default, one row per ESP/ISP) or `domain` (one row per sending domain, with its place in the readiness plan) |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

`group_by=domain` splits a provider that hosts several senders into the hostnames actually sending — a cloud whose customers each get their own hostname, or unknown traffic spread over many hosts of the same provider. Row identity is in `groupKey`; `isp`, `brandDomain`, `sender` and `ipDomainName` describe it, and `domainsCount` / `providersCount` / `ipCount` say how many distinct sending domains, providers and IPs it covers. A high `domainsCount` in provider mode is the signal to switch.

With `group_by=domain`, each row is one sending source, keyed the way the readiness plan keys its sources. Its `groupKey` is the source key that `set_source_decision` takes, and the row carries four more columns:
- `decision` — your call on the source: `ours`, `not_ours` or `ignore`; null for none.
- `planRole` — `blocker` (holds the plan's next step), `confirmation` (unattributed, so you have to call it not yours) or `excluded` (looks like your own mail, but you took it out); null when the step does not depend on the source.
- `verdict` — the plan's call on the source's mail: `aligned`, `needs_dkim`, `relay`, `unattributed`, `inactive` or `low_volume`. A decision never changes it.
- `decisionPending` — true while the plan has yet to take a decision change in, usually within about a minute; null when the plan does not list the source or is not current.

`planRole` and `verdict` come from the plan's own evaluation, whatever period you ask for. The header says when the plan was evaluated and whether it is still current. A row with neither is not in the plan. Rows grouped by provider carry none of these columns, because a provider row covers several sources. The table keeps the 20 largest rows of each type, so a blocker or a source to confirm can be missing from it. `get_enforcement_status` lists them all.

---

### `get_domain_source_details`

Per-record source view with flexible grouping and filtering. Use to drill into specific senders, isolate authentication failures, or investigate by provider/IP/reporter/report. The answer is `{period, coverage, page, totalRecords, rows}`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `group_by` | string | no | `isp` (default) / `ip` / `host` / `reporter` / `report` |
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
| `reporter` | string | no | Filter by DMARC reporter: its name or brand domain as `group_by=reporter` shows them (e.g. `Mail.Ru` or `google.com`), any case but exact |
| `problems_only` | bool | no | ISP grouping only: show only rows with auth failures |

A value outside the ones listed for `eval_*`, `source_type` or `disposition` is refused.

Note: with `group_by=isp`, the same provider may appear multiple times with different countries (one row per provider+country).

`group_by=report` gives one row per aggregate report, newest window first: `reporter`, `reportKey` (tells two reports apart), `windowStart` / `windowEnd` (UTC), `reportDate` (the day it is filed under: the UTC day its window begins), `receivedAt`, `lagHours` (`receivedAt` − `windowEnd`), `records`, `total`, `dmarcPass`, `quarantined`, `rejected`. Every other filter but `problems_only` narrows it too, so to see which reports arrived and what each counted in full, filter by `reporter` alone. Next to the rows, `reporterLag` gives per reporter `reports`, `medianLagHours`, `typicalLagHours` (90th percentile), `maxLagHours` and `latestWindowEnd` (the end of the newest window it reported for this domain). It covers the project's reports of the last 30 days from all of its domains, whatever the period, as `reporterLagScope` says. A report for a window ending at T is due by about T + `typicalLagHours`: before that, a missing one is most likely still on its way.

`spfRecordError` — on a row whose SPF failure one of the domain's own SPF records explains, because receivers cannot evaluate it (two SPF records, a syntax error, more than 10 DNS lookups, an include without SPF): the stored check with `returnPath`, `evaluation`, `permErrorScope`, `failedCheck`, `text`, `fixDescription` and `lastChecked`. The record has to be fixed whoever sent the row. Never on forwarded or relayed rows.

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
| `spf_permerror` | unknown | SPF record broken: your SPF record lists the IP, but the receiver could not evaluate the record (permerror, most often more than 10 DNS lookups or a syntax error). Fix the record |
| `dmarc_not_pass` | unknown | Not aligned with your domain: DMARC failed although your SPF record lists the IP (or, rarely, your DKIM passed). The IP is usually shared — your mail forwarded on the way, someone else's account using your domain, or a service not set up for it. If it is yours, sign it with DKIM for your From domain; if not, treat it as spoofing |
| `dmarc_pass_but_not_ours` | unknown | DMARC passed, but this could not be attributed to your infrastructure |
| `relay_broke_dkim` | unknown | Your DKIM signature is present but no longer verifies, after a known gateway, forwarder or mailbox provider relayed the mail or the receiver reported it as forwarded. Probably your mail changed on the way. A forgery relayed the same way looks identical, so it stays unknown |

`sourceReasonText` is the wording of DMARKOFF's shared source classifier, a short label followed by a description. Empty on reports parsed before the classifier started recording the reason.

---

### `get_geo_sources`

Top 100 sending locations grouped by country, with compliance stats per location.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `source_type` | string | yes | `known`, `unknown`, or `forward`; any other value is refused |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

---

### `get_spf_records`

SPF records from your DMARC data — return paths with recent activity, record text, DNS-lookup count, errors, pass/fail volume over the last 15 days (`today − 14 → today`, UTC; the header names them). Included in `get_domain_full_data`; call this separately only when you need this slice specifically.

Each record also carries `evaluation` and `permErrorScope` (see `dns_check_spf`) and `lastChecked`, when DMARKOFF last checked it. A record last checked before DMARKOFF recorded the evaluation has none, and `dns_check_spf` gives the current one.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |

---

### `get_dkim_records`

DKIM records from your DMARC data — selectors, signing domains, record text, errors, pass/fail volume over the last 15 days (`today − 14 → today`, UTC; the header names them). A key's `total` counts every message carrying that signature, wherever it sits among the message's signatures. Included in `get_domain_full_data`; call this separately only when you need this slice specifically.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |

---

### `get_policy_overrides`

Why messages were **not** handled the way your published DMARC policy says — the receiver reported a policy override in its aggregate report. Grouped by ISP, sender host, disposition and SPF/DKIM results, largest first (top 30). The answer is `{period, coverage, groups}`, `groups` a list even when empty.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

`override_type` values: `arc_pass` (the receiver validated an ARC chain and trusted the original authentication — typical for mail forwarded through Google or Microsoft), `arc_fail`, `forwarded`, `trusted_forwarder`, `sampled_out` (the `pct=` tag excluded the message from enforcement), `local_policy`, `other`. `arc_sealer` names the domain that sealed the chain when the report says so.

Use when the policy is `quarantine` or `reject` but mail still reached inboxes, or when forwarded traffic needs explaining. An empty result means no receiver reported an override in the period, or, as the header then says, that the period has no reports at all.

---

### `get_smtp_rejections`

Mail that receivers throttled or rejected at SMTP time, from the `Sender requirement failed: <code>` comments Google and a few others add to aggregate reports. Grouped by source type, ISP, SMTP code, disposition and auth results, largest first (top 20). The answer is `{period, coverage, groups}`, `groups` a list even when empty.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `stats_period_start` | string | no | Start date `YYYY-MM-DD` |
| `stats_period_end` | string | no | End date `YYYY-MM-DD` |

`smtpCategory` names the reason behind the code: `rate_limit_spf_failure` (421-4.7.27), `rate_limit_dkim_failure` (421-4.7.30), `rate_limit_no_alignment` (421-4.7.32), `rate_limit_no_tls` (421-4.7.29), `blocked_ptr_or_rdns_mismatch` (550-5.7.25), `blocked_spf_failure` (550-5.7.27), `blocked_dkim_failure` (550-5.7.30), `blocked_other` (550-5.7.1), plus the general 421/550 buckets. Each row also carries the receiver's own wording, the reporters that saw it and the message count.

Use when you hear about Gmail bounces or "temporarily rate limited" errors. An empty result means no aggregate report carried such a comment — not that nothing bounced. When the period has no reports at all, the header says so.

---

### `get_record_history`

Stored snapshots of a record, newest first, each with its health and errors at the time. DMARKOFF stores one at its first DMARC check of a domain, or when a check first finds an SPF or DKIM record, and after that only when a check sees a change: a different record, for SPF also a different lookup count or `rules_changed` (below). Each snapshot is kept 12 months. Each one lists `changedFromPrevious` — the fields that differ from the next older one; a DKIM key rotation shows as `record`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `record_type` | string | yes | `dmarc`, `spf` or `dkim` |
| `return_path` | string | no | `spf` only: the return path to trace (default: the domain itself) — take it from `get_spf_records` |
| `dkim_domain` | string | no | `dkim` only: the signing domain (default: the domain itself) |
| `selector` | string | for dkim | `dkim` only: the selector — list them with `get_dkim_records` |
| `limit` | int | no | Snapshots to return, default 10, max 50 |

SPF snapshots also carry `reason`, why DMARKOFF saved them: `record_changed` (a first snapshot too), `lookup_count_changed` or `rules_changed`. `rules_changed` means DMARKOFF's SPF checker was updated and now rates the same DNS record differently, while the record itself did not change. They carry `evaluation` and `permErrorScope` too (see `dns_check_spf`), empty on snapshots taken before DMARKOFF recorded them, and `changedFromPrevious` names `evaluation` when it changed. `reason` is absent on snapshots stored before mid-March 2026 and `lookupCount` on those before early March 2026; `changedFromPrevious` lists `lookupCount` only when both snapshots carry it.

Use after `get_domain_activity_health` or `get_domain_timeline` showed *when* compliance dropped, to check whether a DNS change explains it. No snapshots means the record has not been checked (or found) yet, has not changed in the last 12 months, or the return path / selector does not match a monitored record — not that it is absent from DNS. A domain the project does not hold is refused with `domain not found`.

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

When the record cannot be read because the DNS lookup failed temporarily (`temperror`: a SERVFAIL, a timeout), the tool says so and returns no tree. That is not a missing record — receivers retry such mail.

The response also carries the record's `evaluation` and `permErrorScope` (see `dns_check_spf`). For a record receivers do not evaluate as one record (`permerror` with `permErrorScope` `record`) the tool returns only the volume (`totals`) and, when there are several records, every record. A syntax error fails SPF for every message; with several SPF records receivers differ, so the result is unpredictable. Fix the record first. A permerror reached partway keeps the per-mechanism view, and the header names the failing term. Terms marked `ignored` are never evaluated by receivers and get no traffic.

---

### `get_enforcement_status`

DMARKOFF's own call on whether a domain can move to its next DMARC policy (`none` → `quarantine` → `reject`), or first take `pct` off a record that has it. It returns the readiness plan behind the policy recommendation in the DMARKOFF app and the policy-upgrade email, together with your decisions on the domain's sources. Use it for "can we go to reject?" or "what holds the domain?". The tool tells the assistant to repeat the plan's verdict rather than work one out from compliance figures.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name (e.g. `example.com`) |

Anyone with access to the project can read the plan. A viewer of a shared project sees who made each decision, without their email.

`status`:
- `current` — the plan speaks for the domain. `ready` says whether it can take `nextStep` now, and `nextRecord` is the record to publish for that step. `nextRecordTesting` is the same record with `t=y`, a variant for a cautious start. At `reject` without `pct` there is no next step, so `nextStep` is null and `ready` is false.
- `not_current` — the last plan no longer describes the domain. `reason` is `domain_inactive`, `older_than_72h`, `policy_changed`, `step_mismatch`, `pct_mismatch` or `no_policy`. There is no verdict. `evaluatedAt`, `step` and `pct` say what the plan judged, and `domainState` what the domain is now.
- `none` — no plan yet. DMARKOFF evaluates active domains (paid, not parked, monitored, not excluded) once a day, and `domainState` shows which of these the domain is.

Every answer carries `decisions`, whatever its `status`. These are your live decisions on the domain's sources, with `sourceKey`, `provider`, `decision`, `owner`, `note`, `updatedBy`, `updatedAt`, `confirmedAt` and `pendingRecompute`. The plan takes a change in when DMARKOFF re-evaluates the domain, usually within about a minute. Against a current plan, `pendingRecompute` is true until then, and a decision you removed stays listed with `decision` null. `pendingRecompute` is null when the plan is not current or does not list the source.

Fields of a current plan:
- `evaluatedAt`, `lastDay`, `step`, `testing` — when the plan was evaluated, the last report day it read, the policy in force, and whether the record has `t=y`.
- `readinessPct` — the share of the domain's own mail signed with aligned DKIM over the last `windowDays` days, in percent; null without own mail to judge. It shows progress. The blockers and the open confirmations decide the verdict, so one blocking source can hold the step at 99%.
- `windowDays` — 14, or 28 when the last 14 days give no evidence (under 20 judged own messages, and no source you call ours that signs all of its mail).
- `tenure` — `reportDays` and `daysAtPolicy` against the `requiredReportDays` and `requiredDaysAtPolicy` the step needs.
- `blockers` — what holds the step, by `kind`:
  - `needs_dkim` — an own source that does not sign with aligned DKIM, with its `messages`, `shareOfOwnMailPct` and `dkimAlignedPct`. Subtype `not_aligned` means part of its mail fails DMARC and would be lost at the step. `spf_only` means it aligns by SPF alone where it lacks DKIM.
  - `tenure` — too few days of reports, or too few days at the current policy.
  - `record` — a record with a critical error; `record` names the return path or selector.
  - `regression` — the policy went up, or `pct` came off, and an own source is being cut.
  - `evidence` — nothing shows the step is safe. Subtype `low_volume` means too little own mail, `no_mail` no own mail in the last 28 days.
- `signedBy` — on a `needs_dkim` blocker and on `info.spfOnly` / `info.smallNotAligned`: who signed the source's mail without aligned DKIM, largest first, up to 5. It tells you where the fix is:

  | `kind` | What it means | Where to fix it |
  |--------|---------------|-----------------|
  | `vendor_default` | The sending service (`service`) signed with its own default domain | Set up DKIM for your domain in that service |
  | `platform` | Another organisation (`domain`) signed the mail as itself | Ask it to sign with your domain, or send that mail through one of your own senders |
  | `own_other_domain` | Signed with another domain of the project | Set up DKIM for this domain at the same sender |
  | `own_failed` | Your own signature failed | `result` `permerror` means the key at the selector is missing or broken, and `temperror` that receivers could not look it up. Any other result means the message changed after signing (a forwarder, a mailing list, a relay), which a DKIM setup at the sender does not fix |
  | `own_passed` | Your own signature passed but did not align | Check `adkim=s` and the domain it signs with |
  | `foreign_failed` | Only another organisation's signature, and it failed | At that sender |
  | `unsigned` | No DKIM signature | Enable DKIM at the sender |

- `confirmations` — unattributed sources you have to call not yours before the step, with `messages`, `shareOfAllMailPct` and the classifier's `topReasonText`. A domain with an open confirmation is not ready.
- `sources` — the 50 largest sources, as a table of `sourceKey`, `provider`, `category`, `verdict`, `decision`, `messages` and `dkimAlignedPct`. `verdict` is one of `aligned`, `needs_dkim`, `relay`, `unattributed`, `inactive`, `low_volume`.
- `excluded` — sources that look like the domain's own but were taken out (`not_ours` or `ignore`), with who decided and when.
- `info` — findings that never block: spoofing and relay volume (`spoofMessages`, `relayMessages`), own sources that do not align SPF (`spfNotAligned`), and own sources that need DKIM but are too small to hold the step (`spfOnly`, `smallNotAligned`). These are still worth fixing, because their mail that fails DMARC is lost once the policy goes up.
- `pct` — present only while the record has `pct` below 100 at `quarantine` or `reject`. Receivers that still read `pct` apply the policy to only that share of the mail failing DMARC. The next step then takes `pct` off at the same policy (`nextStep` equals `step`), `nextRecord` is the record without `pct` (at `reject` it also adds `np=reject` when the record has no `np`, as every step to `reject` does), and there is no `nextRecordTesting`. The plan judges the next policy only after that.
- `regression` — the rollback watch, for 30 days after the policy went up or `pct` came off (`from`, `changedAt`, `until`). `sources` lists own sources whose mail receivers now quarantine or reject, with `cutMessages` and `cutPct`. `questions` lists sources you took out that looked like your own before the change and whose mail the new policy now cuts. A question with `confirmed` false is open. You answer it with `set_source_decision` and `confirm`, or change the decision if the source is yours after all.

A decision is about who a source belongs to. Marking a source ours does not authenticate its mail, and the source stays a blocker until it signs with aligned DKIM.

---

## Live DNS tools (no project needed)

These tools query DNS directly in real time and work for any domain — including ones not in your DMARKOFF account. They still need your API key: every request to the server is authenticated. What they do not need is a monitored project or collected report data.

---

### `dns_check_spf`

Live SPF lookup: record text, what receivers evaluate it to, parsed include tree, DNS-lookup count (flags if over the 10-lookup limit), void lookups, errors.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name (e.g. `example.com`) |

Returns, besides the record and its include tree:
- `evaluation` — what receivers' SPF check returns for the record: `ok` (pass or fail then depends on the sending IP), `none` (no record), `permerror` or `temperror`. DMARC counts a permerror as an SPF fail.
- `permErrorScope` — on a permerror. `record` means receivers that follow RFC 7208 do not evaluate it as one record, and `lookupCount` 0 means "not counted". With a syntax error anywhere in it, SPF fails for every message. With several SPF records receivers differ — some return permerror, some report fail, some evaluate one of the records — so the result is unpredictable: merge them into one. `evaluation` means the record breaks partway (an include or redirect target without a valid SPF record, more than 10 DNS lookups, more than 2 void lookups). Senders matched before that point still pass, and every other sender gets permerror.
- `voidLookups` — lookups of the record's mechanisms that returned no answer.
- `records` — every SPF record at the name, when there is more than one.
- `ignored` — on parts of the include tree that receivers never evaluate.

An `include:` or `redirect=` whose target is built from sender macros (Proofpoint's `include:%{ir}.%{v}.%{d}.spf.has.pphosted.com`, Valimail's `%{i}…`) counts as one DNS lookup itself; the target is a record of its own for each sending IP, and its terms count in neither `lookupCount` nor `voidLookups`. For a sender the record does not list, an Info error says when that target takes it over the 10-lookup or 2-void-lookup limit, and a Critical error with permerror when the target does not exist for it.

A result other than `ok` also gets a one-line verdict ahead of the details.

---

### `dns_check_dmarc`

Live DMARC lookup: record text, parsed policy (none/quarantine/reject), errors. `records` lists the DMARC records when several are published at `_dmarc.<domain>`: receivers then use none of them.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |
| `project_id` | int | no | A project the domain belongs to: also checks the record against it. Pass it after publishing a record for a domain of a project |

With `project_id` the answer adds `reportsToProject`:
- `address` — what `rua` must carry: the project's `monitoringEmail` when set, DMARKOFF's reporting address otherwise.
- `found` — whether the live record has it, the same check that makes a domain Monitored; null when that cannot be told.
- `inProject` — whether the project holds the domain (null when that could not be read), and `dmarcStatus` its status there.
- `note` — what follows: when `dmarcStatus` turns Monitored and reports come, or why not (the domain is not in the project or is in another one, is excluded or unpaid, or publishes several records). An inherited record makes the domain Monitored, but its reports are filed under the organizational domain.

`tempError` is true when the lookup failed at some step of the search, and `lookupFailure` names the step (`qname`, `_dmarc.<name>`) and the cause: `servfail`, `refused`, `timeout` or `error`. An empty record then means the record could not be checked — not that the domain has no DMARC.

---

### `dns_check_dkim`

Live DKIM key lookup for a specific selector: queries `<selector>._domainkey.<domain>`, returns key record, key length, errors.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |
| `selector` | string | yes | DKIM selector (e.g. `google`, `selector1`) |

---

### `dns_check_txt`

All TXT records for a domain with TTL and type classification: `spf`, `spf-ignored`, `dkim`, `dmarc`, `mta-sts`, `tlsrpt`, `bimi`, `site-verification`, `acme`, `mcp-registry-auth`, `sender-id`, `other`.

`spf-ignored` is a record that looks like SPF but that receivers do not read as SPF, so it is never counted as a second SPF record. Its error (`txt_spf_record_format_ok`) explains why and what to do.

Useful for detecting duplicate SPF records (RFC 7208 makes multiple `v=spf1` entries a permerror, but receivers differ, so SPF results are unpredictable — merge them into one) or verifying a newly published record is visible.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |

---

### `generate_dmarc_record`

Builds the DMARC TXT record a domain should publish. Reads what is currently in DNS and merges the requested tags into it, repairing what it builds on: tags the checker accepts stay as written; an invalid `p` becomes `none`, other invalid `sp`, `np`, `adkim`, `aspf`, `t`, `psd` or `fo` values are dropped, and tags a missing `;` ran together are split again (a swallowed `p`, `sp`, `np` or `t` is not recovered).

Without `p`, `sp` or `np` the record keeps the policy receivers apply now, for the domain and its subdomains. A record with an invalid `p`, `sp` or `np` is applied as `none` as a whole (RFC 9989 §4.10.1), so it comes back as `p=none` without its `sp` and `np` above none. With only `sp` or `np` passed, a valid written `p` stays — and on such a record that brings it into force: `sp=none` on `p=reject; sp=x; np=reject` gives `p=reject; sp=none; np=reject`. To change only the subdomains of such a record, pass `p`, `sp` and `np` together.

When the current record cannot be read, the note names the DMARC lookup step that failed and why.

Pass `project_id` for a domain in a DMARKOFF project: the address DMARKOFF checks is added to `rua` — the project's `monitoringEmail` when set (an MSP mailbox that forwards the reports to DMARKOFF), DMARKOFF's reporting address of the project otherwise — which is what makes DMARKOFF receive aggregate reports for it. `reportingAddress` is the address added.

Returns the host to publish at (`_dmarc.<domain>`), the current record with its status, and the recommended record. `currentRecordStatus` is `published`, `inherited` (only a parent domain publishes one: the domain needs a record of its own for its reports to be filed under it), `none`, `unusable` or `lookupFailed`. `unusable` is something receivers ignore: several records or a CNAME/TXT conflict, which `note` says to fix in DNS first, or a record at the domain itself rather than at `_dmarc`, which the recommended record replaces. For a record the domain publishes, `changes` lists what the recommended record changes, tag by tag (`{tag, from, to}`; no `from` for an added tag, no `to` for a removed one). Nothing is published — you add the TXT record in your DNS zone, then verify with `dns_check_dmarc` and `project_id`.

Use it for onboarding and for tag changes. It keeps `pct` as it is. To move a domain to a stricter policy or to take `pct` off, publish `nextRecord` from `get_enforcement_status`, the readiness plan's record for that step.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | yes | Domain name |
| `project_id` | integer | no | Adds the address DMARKOFF checks for the project to `rua` |
| `p` / `sp` / `np` | string | no | Policy, subdomain policy, non-existent subdomain policy: `none`, `quarantine`, `reject` |
| `adkim` / `aspf` | string | no | Alignment: `s` (strict) or `r` (relaxed) |
| `fo` | string | no | Failure reporting options: `0`, `1`, `d`, `s` joined by `:` |
| `t` | string | no | RFC 9989 test mode: `y` or `n` |
| `psd` | string | no | Public suffix domain flag: `u`, `n`, `y` |
| `rua_emails` | string | no | Comma-separated addresses replacing the ones in the current record; the project's address is always kept |
| `transfer_ruf` | string | no | Keep the `ruf` tag of the current record: `true` or `false` |

---

## Onboarding tools (require API key)

### `add_domain`

Adds one or more domains to a project so their DMARC reports start being collected. Up to 20 per call.

Requires **owner** access to the project — domains cannot be added to a project shared with you. Each non-parked domain consumes a paid domain slot on the account's plan; `parked` domains (zones that send no mail) do not.

Domains already in the account are reported as `skipped`, with where they are (`projectId`, and `dmarcStatus` when in this project); invalid ones as `failed`; the rest are still added. A domain added with a warning (its monitoring did not start, say) is in `added` and in `warnings`. `add_domain` does not move a domain from another project, nor change whether one is parked, paid or excluded. Repeating a call is safe: a domain already in this project that is not Monitored, excluded or unpaid gets its DNS setup again.

Registering the domain is only half the job: reports start flowing once the domain's own DMARC record has the project's address in `rua` (`dnsSetup.reportingAddress`: the project's `monitoringEmail` when set). A record inherited from a parent domain is not enough. For a single domain, `dnsSetup` is the exact record, as `generate_dmarc_record` returns it, and `nextStep` says what to do with it. For several, it is a template plus the domains sorted by what they publish:
- `templateApplies` — no DMARC record, or an inherited one the template would not change: publish the template as is.
- `needsMerge` — a record of their own, something receivers ignore, or an inherited one the template would change: call `generate_dmarc_record` for each, as the template would drop their tags, add one more record or lower their policy.
- `alreadyReporting` — their own live record already has the address: nothing to publish.
- `notInDns` — the domain does not exist in DNS: check the spelling.
- `checkFailed` — the lookup failed: call `generate_dmarc_record` for each.

When the project's address cannot be read, there is no `dnsSetup`, and `nextStep` says to call `generate_dmarc_record` with `project_id`.

Once the record is live, DMARKOFF checks a newly added domain every 5 minutes for 24 hours, so `dmarcStatus` turns Monitored within about 5 minutes; a domain still without the address after 24 hours becomes NotMonitored and is no longer checked until "Reinitiate monitoring" in the web app, or until a report of it is processed. The first reports usually arrive 24–48 hours after the record goes live; until then the data tools answer with `coverage.noReports`. Check the record with `dns_check_dmarc` and `project_id`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | integer | yes | Project to add the domains to |
| `domains` | string[] | yes | 1 to 20 domain names |
| `parked` | boolean | no | Mark as parked: no mail expected, at most one report a day processed (so mail sent as the domain still shows), no paid slot |

---

## Source decisions (require API key)

### `set_source_decision`

**Writes.** Records your call on one sending source of a domain, or clears it. The readiness plan (`get_enforcement_status`) counts the domain's sources by these decisions. The tool tells the assistant to use it only to save a call you made and asked it to save.

Requires the project's owner, or a viewer of a shared project whose owner lets it manage sources. The change goes through the DMARKOFF API, which checks access on every call.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_id` | int | yes | Project ID |
| `domain` | string | yes | Domain name |
| `source_key` | string | yes | The source's `sourceKey` in `get_enforcement_status`, or its `groupKey` in `get_domain_senders` with `group_by=domain`, exactly as listed (case- and space-sensitive, up to 255 bytes) |
| `decision` | string | yes | `ours`, `not_ours`, `ignore`, or `clear` to remove the decision. With `confirm`, the exclusion saved on the source (`not_ours` or `ignore`) |
| `owner` | string | no | Who runs the source on your side (e.g. `Marketing / Klaviyo`), up to 100 characters. Omit to keep the current one, pass `""` to clear it. Ignored with `clear` and `confirm` |
| `note` | string | no | A note on the decision, up to 1000 characters. Omit to keep the current one, pass `""` to clear it. Ignored with `clear` and `confirm` |
| `confirm_exclude_own_source` | bool | no | Set only after you confirmed taking out a source the plan judges your own (default false) |
| `confirm` | bool | no | Confirms the exclusion already saved on the source, as your answer to a rollback-watch question, instead of changing it (default false) |

Decisions:
- `ours` — the domain's own mail. It authenticates nothing, and the source stays a blocker until it signs with aligned DKIM.
- `not_ours` — someone else sending as the domain. This is how you answer a confirmation in the plan.
- `ignore` — left out of the readiness count.
- `clear` — removes the decision, so the source counts by its verdict again.

Taking out a source the plan judges your own (`not_ours` or `ignore` on a source with verdict `aligned` or `needs_dkim`, or on a `needs_dkim` blocker) is not saved on the first call. The tool answers `confirmation_required` with the source's verdict, its messages and what taking it out does. It saves only when called again with `confirm_exclude_own_source` after you confirm. Taking out a blocker can make the domain read ready, and once the policy goes up, that source's mail is quarantined or rejected. The check covers the sources the domain's last plan names (its 50 largest, the excluded ones and the blockers), and a domain without a plan gets no warning.

`confirm` answers the rollback watch. After the policy goes up, `get_enforcement_status` lists in `regression.questions` the sources you took out whose mail the new policy now cuts. With `confirm` and the question's decision, you confirm that the source stays out. Only the decision's `confirmedAt` changes, and DMARKOFF does not ask about the source again for this policy change. A confirm is refused when the source has no decision or a different one. Saving the same decision again without `confirm` does not answer the question.

Returns:
- `status` — `saved`, `cleared`, `confirmed`, `confirmation_required` (nothing was saved) or `not_confirmed` (an error: nothing was confirmed).
- `decision` — the decision as saved.
- `previous` — the decision the source had before the call, null for none.
- `warning` — on `confirmation_required`, the source's verdict and messages behind the question. On a saved decision, DMARKOFF's warning that it takes out a source the plan judges your own.
- `requested`, `snapshotEvaluatedAt` — on `confirmation_required`, the decision you asked for and when the plan behind the question was evaluated.
- `next` — what happens to the plan.

Each change is logged with the API key as its author. The plan takes it in when DMARKOFF re-evaluates the domain, usually within about a minute. A domain DMARKOFF does not evaluate (unpaid, parked, not monitored or excluded) takes it in only once it is evaluated again. A key that no report of the domain carries is refused.

When a call times out, its outcome is unknown. Check `decisions` in `get_enforcement_status` before trying again. Repeating a `confirm` is safe, because it changes nothing but `confirmedAt`.

---

## Field glossary

| Field | Meaning |
|-------|---------|
| `period` | The days a tool read: `start`, `end`, `days` (both ends inclusive), and `endClamped` when the end asked for was after today. |
| `coverage` | The days the project has reports for the domain, whatever the period: `firstReport` and `lastReport`, or `noReports`. A period before `firstReport` is one DMARKOFF did not see the domain, not one without mail. |
| `dmarcStatus` | Whether the project's address (its `monitoringEmail` when set, DMARKOFF's reporting address otherwise) was in the domain's DMARC `rua` at the last check — NOT whether a DMARC DNS record exists, nor whether reports arrive (`coverage` says that). `Monitored`; `Pending` — not yet, while DMARKOFF checks the domain every 5 minutes for 24 hours after it was added; `NotMonitored` — not found, and DMARKOFF no longer checks it (also set when the domain is excluded or unpaid). Use `dns_check_dmarc` with `project_id` for the actual DNS record. |
| `monitoring` | In `get_domain_full_data` and `get_domain_detail`, for a domain that is NotMonitored, Pending, excluded or unpaid: why, the address `rua` must carry, and the fix. |
| `reportStatus` | A sticky flag, not whether reports arrive now: `Receiving` once a report reached DMARKOFF since the domain was added; only a domain DMARKOFF evaluates daily (paid, not parked, Monitored, not excluded) goes back to `NotReceiving` after 2 days without reports. Whether the project still gets reports is `coverage.lastReport`. |
| `reportProcessingStatus` | `Processing` / `NotProcessing`: a label set when the domain is added, paid, unpaid, parked, excluded or included, which nothing decides by. Which reports are processed follows `paid`, `parked` and `userOverride`. |
| `userOverride` | `Monitored`, the default (not excluded), or `Excluded`: the user took the domain out of monitoring in the web app, and its reports are not processed. Excluded domains are not listed in `list_domains`. |
| `parked` | The domain is not meant to send mail: health metrics do not apply, and no reports is its normal. Mail reported for it is sent as a domain that should send none — spoofing when it fails DMARC, a forgotten or compromised sender when it passes — and is worth a look at its sources. DMARKOFF processes one report a day for a parked domain, so its volumes are a floor. |
| `policy_override` | Override applied by the receiving mail server (from the DMARC XML report). Not the same as the user's manual override. |
| `compliancePct` | Compliant messages over all messages — the number the DMARKOFF app shows. Forwarded mail is excluded from the numerator but stays in the denominator, so it reads lower than the raw pass rate in `get_domain_stats` when a domain has forwarding traffic. Different metrics, not a contradiction. |
| `pctEligibleForPolicy` | The same rate over eligible messages only: compliant / (messages − forwarded). 0 by convention when all mail was forwarded, and `noEligibleMail` says so. |
| `pctFromKnownSources` / `complianceKnownPct` | The DMARC compliance of known-source mail, not the share of mail from known sources. Mail is classified known only when it passes DMARC, so it reads 100 whenever there is known mail: no measure of your own senders. |
| `dmarcFail` | Two definitions, both matching the app: in a stats summary it is `messages − compliant`, so forwarded mail counts as a failure and it stays consistent with `compliancePct`; in a timeline row it is `messages − compliant − forwarded`. The summary is therefore larger than the sum of the timeline column. For messages that actually failed the policy check, use `get_domain_stats`. |
| `fix` / `failedCheck` / `fixDescription` | On every record error: the fix code, the check that produced it, and the remediation text the app shows for that code. |
| `checkDescription` | What the failed check means, as the app explains it. On the first error of each `failedCheck` in a list; later errors of the same check leave it out. |
| `sourceReason` / `sourceReasonText` | Why a source was classified known / unknown / forward — see the table under `get_domain_source_details`. |
| `spfRecordError` | In `get_domain_source_details`: the stored check of one of the domain's own SPF records that receivers cannot evaluate, on a row whose SPF failure it explains. Fix the record, whoever sent the row. |
| `excludeFromHealth` | On SPF/DKIM records: the record was suppressed in the app. It is still listed with its own health and errors, but it does not feed the domain's severity — so a Critical record under a Healthy domain is expected, not a contradiction. |
| `missingReturnPaths` | ESPs sending with their own Return-Path instead of your domain — causes SPF alignment failure. Fix: configure a Custom Return-Path at the ESP. |
| `evaluation` / `permErrorScope` | On SPF records, live and stored: what receivers' SPF check returns, `ok`, `none`, `permerror` or `temperror`. `permErrorScope` `record` means receivers that follow RFC 7208 do not evaluate it as one record: a syntax error fails SPF for every message, several SPF records give results that differ by receiver. `evaluation` means it breaks partway, and senders matched before that point still pass. |
| `ready` / `nextStep` / `nextRecord` | In `get_enforcement_status`: whether the domain can take its next step now, the policy that step reaches (the current one when the step takes `pct` off), and the record to publish for it. |
| `readinessPct` | The share of the domain's own mail signed with aligned DKIM over `windowDays` days. It shows progress. The blockers and confirmations decide the verdict. |
| `verdict` | The readiness plan's call on a source's mail: `aligned` (own mail signed with aligned DKIM), `needs_dkim` (own mail that is not), `relay` (mostly forwarded or gateway mail), `unattributed` (mostly mail nobody has identified, spoofing included), `inactive` (sent in the last 28 days but not in the last 14), `low_volume` (too little own mail to judge). A decision never changes it. |
| `planRole` | A source's place in the readiness plan: `blocker` holds the next step, `confirmation` is an unattributed source you have to call not yours, `excluded` looks like your own mail but you took it out. Null when the step does not depend on the source. |
| `decision` | Your call on a source: `ours`, `not_ours` or `ignore`, saved with `set_source_decision`. It records who the source belongs to. Marking a source ours does not authenticate its mail. |
| `pendingRecompute` / `decisionPending` | The plan has yet to take a decision change in. DMARKOFF re-evaluates an active domain usually within about a minute of a change. Null when the plan does not list the source or is not current. |

Which SPF/DKIM records are listed at all: those the domain's health is computed from — contributing and seen in reports within the last 14 days (for SPF, records flagged as always contributing are listed regardless of activity). A return path or selector missing from the list is inactive or non-contributing; that is not evidence it is absent from DNS — check with `dns_check_spf` / `dns_check_dkim`.

## Recommended workflow

```
projects_overview
  → project without domains, or a domain to start monitoring:
      add_domain → publish the record in dnsSetup → dns_check_dmarc (project_id)
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
"is this report late or missing?"   → get_domain_source_details (group_by=report, reporter)
"will its reports reach us?"        → dns_check_dmarc (project_id)
SPF over the 10-lookup limit        → get_spf_usage
"can we go to quarantine / reject?" → get_enforcement_status
"this sender is ours / not ours"    → set_source_decision (only on your word)
```

Any question about a stricter policy goes to `get_enforcement_status`, and the assistant repeats its verdict. Compliance figures alone do not say whether a domain is ready.

For quick DNS verification without a project: use `dns_check_spf`, `dns_check_dmarc`, `dns_check_dkim`, `dns_check_txt` directly.
