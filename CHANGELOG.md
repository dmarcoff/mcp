# Changelog

## 2026-10-15

### Added
- `get_enforcement_status` — DMARKOFF's readiness plan for a domain. It says whether the domain can move to its next DMARC policy (`none` → `quarantine` → `reject`), or first has to take `pct` off a record that has it. It names what holds the step: own senders that do not sign with aligned DKIM, too few days of reports or at the policy, a record with a critical error, or too little own mail to judge. For a sender that needs DKIM, `signedBy` shows who signed its mail instead, which tells you where to set DKIM up. It also lists the unattributed sources you have to confirm are not yours, and gives the record to publish for the step (`nextRecord`). The plan is the same one behind the policy recommendation in the DMARKOFF app and the policy-upgrade email, and the tool tells the assistant to repeat its verdict instead of working one out from compliance figures. A plan that no longer describes the domain reads `not_current` with the reason and gives no verdict. For 30 days after the policy goes up or `pct` comes off, `regression` shows the rollback watch: own sources the change cuts, and questions about sources you took out.
- `set_source_decision` — records your call on one sending source of a domain (`ours`, `not_ours` or `ignore`), or clears it. The readiness plan counts the domain's sources by these decisions. This is the second tool that writes. It goes through the DMARKOFF API and needs owner access to the project. Taking out a source the plan counts as your own (its largest sources and the ones that hold the step) is not saved on the first call. The tool answers `confirmation_required` with the source's verdict and message count, and saves only when called again after you confirm. With `confirm`, it answers a rollback-watch question: you confirm that a source you took out stays out, now that the new policy cuts its mail. Each change is logged with the API key as its author, and the plan usually takes it in within about a minute.

### Changed
- `get_domain_anomaly_report` returns `paid`, `parked`, `userOverride` and `dmarcStatus` for every domain the project holds, with a row even for one without any report (`lastReportDate` 1970-01-01), and names the state in the domain's header: parked, excluded from monitoring, unpaid, not monitored. A domain the project holds but has no reports for no longer reads "may not belong to the specified project".
- `get_domain_senders` with `group_by=domain` gives each row its place in the readiness plan: `decision` (your call on the source), `planRole` (`blocker`, `confirmation` or `excluded`), `verdict` (the plan's call on the source's mail) and `decisionPending` (the plan has yet to take a decision change in). Rows grouped by provider are unchanged.
- `dns_check_spf` returns `evaluation`, the result receivers' SPF check gives for the record: `ok`, `none`, `permerror` or `temperror`. A permerror also carries `permErrorScope`. With `record`, receivers never evaluate the record (a syntax error, or several SPF records), so SPF fails for every message. With `evaluation`, the record breaks partway, and senders matched before that point still pass. The tool also returns `voidLookups` and every record when a name has several. A result other than `ok` gets a one-line verdict ahead of the details.
- Stored SPF records in `get_spf_records` and `get_domain_full_data` carry `evaluation`, `permErrorScope` and `lastChecked`. A record last checked before DMARKOFF recorded the evaluation has none, and `dns_check_spf` gives the current one.
- SPF snapshots in `get_record_history` carry `reason`, `evaluation` and `permErrorScope`. `reason` says why DMARKOFF saved the snapshot: `record_changed`, `lookup_count_changed` or `rules_changed`. `rules_changed` means DMARKOFF's SPF checker changed and now rates the same DNS record differently. `changedFromPrevious` names `evaluation` when it changed.
- `sourceReasonText` comes from DMARKOFF's shared source classifier, a short label and a description for every reason. The `relay_broke_dkim` text changed. It now covers mail whose DKIM signature is present but no longer verifies, after a known gateway, forwarder or mailbox provider relayed it or the receiver reported it as forwarded. It stays unknown because a forgery relayed the same way looks identical.
- `generate_dmarc_record` points to `get_enforcement_status` for a stricter policy. It is meant for onboarding and tag changes, and it keeps `pct` as it is. For the next policy step, publish `nextRecord` from the readiness plan.

### Fixed
- `get_domain_anomaly_report` judged a parked domain — one not meant to send mail — like any other. Silent for a while, it read "reports may have stopped", although no reports is what a parked domain should have; with reports, it was at best a sparse sender, although a report for a parked domain means mail is being sent as a domain that should send none. The tool now reads the domain's state in the project, and a parked domain is never judged on volume or silence. A day without reports is a note. Mail on the evaluated day is judged by its DMARC result: mail that passes is a warning, a forgotten or compromised sender to find with `get_domain_senders`; mail that all fails is spoofing, a note while the domain's DMARC policy quarantines or rejects it and a warning when the policy in effect is `none`. That DMARC result replaces the compliance, unknown-sender and return-path checks for a parked domain. The message count is a floor: DMARKOFF processes one report a day for a parked domain.
- DMARKOFF does not process the reports of a domain excluded from monitoring, yet `get_domain_anomaly_report` judged it on them: an excluded daily sender got the "possible reporting gap" warning from its first day without reports, and "reports may have stopped" a week later. An excluded domain now gets a single note, its reports are no longer processed, and nothing is judged.
- An unpaid domain gets at most one report a day processed, and none once it has been unpaid for more than 3 days, so its volume and its silence measured the limit, not its mail. `get_domain_anomaly_report` now says so in a note and judges neither; the compliance, unknown-sender and return-path checks of the day still apply.
- `generate_dmarc_record` and `add_domain` added the project's reporting address to `rua` a second time when the record already had it with a size limit, such as `!10m`.
- `get_spf_usage` read a name with several SPF records as having no SPF record. For a record receivers never evaluate (several records, or a syntax error), it now returns only the volume, because SPF fails for every message and a per-mechanism view would mislead. A permerror reached partway keeps the per-mechanism view, and the header names the failing term.

---

## 2026-10-14

### Fixed
- `get_domain_anomaly_report` reported an "unusual drop" on practically every domain. It judged yesterday while yesterday's reports were still arriving — DMARC aggregate reports come in a day or two after the traffic — so the day read as empty or half-full against a full baseline. It now judges the last settled day, 2 days ago in UTC, against the 14 days before it, and the response says which day that is (`evaluatedDate`) and which baseline (`baselineStart`, `baselineEnd`).
- A domain that sends now and then — weekly batches, a few messages a month — was flagged as a drop on every day it did not send. A domain with reports on fewer than 7 of the 14 baseline days is now reported as a sparse sender and its volume is not judged; `baselineReportDays` gives the count.
- A day without any reports is no longer called a volume drop. The report says there were no reports for that day and when the last one arrived, and treats it as a possible reporting gap only for a domain with reports on at least half the baseline days and on both baseline days with the same weekday (`sameWeekdayReportDays`), so a domain that sends on weekdays only is not flagged every Saturday.
- A domain whose reports stop is flagged as such: no reports for more than a week, and for longer than any gap in its baseline (`baselineMaxGapDays`), is a warning that reports may have stopped, also once so few report days are left in the baseline that the domain would read as a sparse sender. A domain with no reports in the 14 baseline days but reports on the evaluated day or later is said to have no baseline yet instead of being called a sparse sender.
- The compliance check fired on most days for any domain whose compliance varies at all: it compared the day against the baseline's 95th percentile, which almost every ordinary day falls below. It now compares against the 5th percentile, `dmarcQ5`.

### Changed
- A volume drop in `get_domain_anomaly_report` means more than 20% under the baseline's 5th percentile, the same margin DMARKOFF's own anomaly detection uses.

---

## 2026-09-09

### Added
- `get_policy_overrides` — why receivers did not apply the published DMARC policy: ARC pass or fail, forwarded, trusted forwarder, sampled out, local policy, with the domain that sealed the ARC chain. The answer to "we publish `p=reject`, so why did this still reach the inbox".
- `get_smtp_rejections` — mail receivers throttled or rejected at SMTP time, taken from the "Sender requirement failed" comments in aggregate reports. Gmail's 421 and 550 codes come back categorised by reason: SPF failure, DKIM failure, missing alignment, no TLS, PTR mismatch.
- `get_record_history` — stored snapshots of a DMARC record, one SPF return path or one DKIM selector, newest first, each naming the fields that changed since the previous check. Answers "when did this change, and what was it before".
- `get_spf_usage` — which mechanisms of an SPF record actually carried mail. The record is resolved live and every sending IP seen in the reports is matched against its `ip4`/`ip6` networks: includes with no traffic are removal candidates with the DNS lookups they cost, and senders no mechanism covers are listed separately. For records at or over the 10-lookup limit.
- `get_domain_senders` takes `group_by`: `provider` (default, one row per ESP or ISP) or `domain` (one row per sending domain). A provider hosting several senders splits into the hostnames that actually send.
- `get_domain_full_data` returns `previousPeriod` — the same headline numbers over the window before the requested one, the same number of days, ending the day before it starts — with the volume change in percent and the compliance change in percentage points. A previous window that carried no mail reports no percentage and says so, rather than calling a domain that just started sending +100%.
- `projects_overview` returns `compliancePctNoForwards` and `complianceKnownPct` beside `compliancePct`: compliance over the mail a DMARC policy can act on, and over known sources only — the health of your own sending infrastructure, unaffected by spoofing attempts.
- Every record error now carries `fix`, `failedCheck` and `fixDescription` — the remediation text the DMARKOFF app shows for that finding — on stored records and on live `dns_check_*` results alike.
- `get_domain_source_details` returns `sourceReason` and `sourceReasonText`: which evidence put a source into known, unknown or forward. "Unknown" on its own does not say whether to hunt for a sender or fix an alignment setting; the reason does, and the two lead to opposite actions.

### Changed
- `get_domain_senders` reads a daily sources rollup instead of raw report records. Same numbers, and grouping by sending domain became possible without scanning the raw table.
- Tool descriptions now define `compliancePct` and both meanings of `dmarcFail`: the stats summary counts forwarded mail as a failure, the timeline does not, and the app shows the first. Previously an assistant comparing the two had no way to tell that the gap was a definition rather than a data error.
- SPF and DKIM record lists document what they contain: records the domain's health is computed from. A record marked `excludeFromHealth` is still listed but does not affect the domain's severity, and a return path or selector missing from the list is inactive — not absent from DNS.

### Fixed
- `projects_overview` could leave projects out entirely. Pages were cut from a list ordered differently from the one returned, and the total counted only the projects the account owns, so shared projects fell off the end and were unreachable on any page. Projects now come back ordered by message volume, largest first, with a total that matches the list.
- `get_spf_records` listed a return path only when it had traffic in the last 14 days. A record that drives the domain's SPF severity without sending — the domain's own return path, typically — was missing from the answer while the severity it caused was plainly visible.

---

## 2026-08-20

### Added
- `add_domain` — adds up to 20 domains to a project and returns the DMARC record to publish for them. Requires owner access to the project; each non-parked domain consumes a paid domain slot. The first tool that writes: it creates domains, never updates or deletes.
- `generate_dmarc_record` — builds the DMARC TXT record a domain should publish, merged on top of whatever is already in DNS. With `project_id` it adds the project's reporting address to `rua`, which is what starts report collection.

### Changed
- Protocol revision `2026-07-28` is now supported alongside earlier ones. Clients that speak it negotiate up automatically; older clients are unaffected.
- `dns_check_dmarc` reports the domain's own nameservers instead of the resolver that was queried.
- Record lookups distinguish "no DMARC record published" from "DNS could not be checked" — previously both looked like an empty record.

### Removed
- `scopes_supported` is no longer advertised in the OAuth metadata. No scope was ever issued or enforced, and keeping the declaration would have implied a read-only mode that does not exist. A key's reach is bounded by its project access.

---

## 2025-04-30

### Added
- `get_domain_activity_health` — daily per-day breakdown of SPF/DKIM/DMARC severity and quantile vs 14-day baseline. Shows when and why a domain's health changed.
- `get_domain_source_details` — per-record source view with flexible grouping (isp/ip/host/reporter) and 11 optional filters (eval_spf, eval_dkim, eval_dmarc, source_type, disposition, dkim_domain, dkim_selector, spf_domain, source_ip, isp, ip_domain_name). `problems_only` flag for ISP grouping.
- `get_domain_anomaly_report` now accepts multiple comma-separated domains.
- Statistical quantiles (5th, 90th, 95th percentile) in `get_domain_full_data` and `get_domain_timeline` — use as baselines for anomaly detection.

### Changed
- `get_domain_full_data` now includes daily timeline and records by default (no separate `include_*` flags needed).
- MCP server instructions (system prompt) updated: added field glossary, workflow guidance, and key concept definitions for `missingReturnPaths`, `pctEligibleForPolicy`, `activityState`.

### Removed
- `get_domain_health_trends` — superseded by `get_domain_activity_health` (richer data, quantile-based).

---

## 2025-03-17

### Added
- Initial release: 12 analytics tools + 4 live DNS tools
- OAuth 2.1 support (RFC 8414, RFC 9728) — compatible with Claude.ai, ChatGPT
- Streamable HTTP transport
