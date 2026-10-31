# Changelog

All notable changes to ShadowSaaS Surface Scanner are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project follows [Semantic Versioning](https://semver.org/).

## [1.1.0] - 2026-10-31

### Added
- `--proxy URL` — route HTTP probes and CT log lookups through a proxy.
  Supports `http://`, `https://`, `socks5://`, and `socks5h://`. DNS
  CNAME/A resolution is intentionally **not** routed through the proxy
  (raw DNS queries, not HTTP), and the tool prints a note when a
  `socks5://`/`socks5h://` proxy is configured so this isn't a silent gap.
- `--proxy-ca PATH` — trust a custom CA bundle (PEM) for the CT log
  HTTPS lookups (`crt.sh`, `certspotter`, `hackertarget`) when inspecting
  traffic through an intercepting proxy such as Burp. Subdomain probes
  (`http_probe`) are unaffected — they already use `verify=False` on
  purpose, since dangling/expired certs on abandoned subdomains are
  expected there, not a MITM signal.
- Proxy URL validation: a malformed `--proxy` value (nested scheme, e.g.
  `socks5h://http://127.0.0.1:8080`, or a URL missing host/port) is now
  rejected immediately with a clear error, instead of producing an
  opaque connection failure later.
- A certificate verification failure while `--proxy` is active and no
  `--proxy-ca` is set is now detected as a systemic condition (the same
  untrusted CA affects every HTTPS request through that proxy) rather
  than a per-provider issue:
  - The CT log cascade fails fast (1 attempt, not 3x backoff per
    provider) instead of retrying three CT log sources against the same
    certificate error.
  - The entire target scan aborts — including `--speculative` and
    `--bruteforce` fallbacks — rather than silently falling back to a
    partial enumeration that would be indistinguishable from a genuine
    zero-subdomain result.
  - `summary.warnings` is added to the JSON/HTML output whenever this
    happens, so the condition is visible even to someone who only reads
    the report later, not just the console output at scan time.
- `requests.exceptions.ProxyError` during a subdomain HTTP probe is now
  surfaced with a clear message instead of being silently absorbed into
  the same code path as a genuinely offline host — a broken proxy no
  longer produces a false "not a takeover" result.

### Changed
- CLI entry point moved from a bare `if __name__ == "__main__":` block
  into a `main()` function, eliminating `redefined-outer-name` lint
  warnings that the bare module-level script triggered for every
  function parameter named `proxies`/`verify`.
- Version string centralized into a single `__version__` constant;
  previously it was hardcoded independently in five places (module
  docstring, two User-Agent headers, the CLI help banner, and the HTML
  report), which had already drifted out of sync during development.

### Compatibility
- Fully backward compatible. Every new parameter (`proxies`, `verify`,
  `warnings`) defaults to the v1.0 behaviour when omitted — running
  without `--proxy`/`--proxy-ca` produces byte-identical HTTP requests
  to v1.0.
- No changes to the detection logic (`risk_score`, `detect_saas_by_domain`,
  `fingerprint_saas`, provider fingerprinting) — only the transport layer
  and CLI changed.

## [1.0.0] - 2026-04-29

Initial public release. Subdomain enumeration via CT log cascade
(crt.sh → certspotter → hackertarget), DNS resolution, HTTP probing,
and conservative SaaS takeover detection for Azure, AWS CloudFront,
GitHub Pages, Heroku, Vercel, Netlify, Shopify, and Firebase.
