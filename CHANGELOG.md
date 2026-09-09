# Changelog

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
