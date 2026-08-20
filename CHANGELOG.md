# Changelog

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
