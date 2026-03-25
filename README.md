# 🔒 TechOptima.ai — Security & Performance Audit

External security and performance audit of [techoptima.ai](https://techoptima.ai), covering TLS/SSL, HTTP headers, Content Security Policy, email authentication, caching, and more.

## Scorecard

| Area | Grade | Key Finding |
|------|-------|-------------|
| **TLS/SSL** | ✅ A | TLS 1.3 only, valid wildcard cert |
| **HTTP Headers** | ✅ B+ | HSTS preload, X-Frame-Options DENY, Referrer-Policy |
| **CSP** | ⚠️ B- | `unsafe-inline` + `unsafe-eval` weaken XSS protection |
| **HTTPS** | ✅ A | 301 redirect + HSTS with preload |
| **Email Security** | 🔴 D | No SPF, DMARC `p=none` — spoofing possible |
| **Caching** | 🔴 F | No `Cache-Control` headers |
| **Performance** | ⚠️ C+ | TTFB ~416ms, 178KB page, Brotli enabled |

## Critical Findings

1. **No SPF + DMARC `p=none`** — anyone can spoof emails as `@techoptima.ai`
2. **CSP allows `unsafe-eval`** — significantly weakens XSS defense

## Files

| File | Description |
|------|-------------|
| [`walkthrough.md`](walkthrough.md) | Full audit report with 15 prioritized recommendations |
| [`output.md`](output.md) | Raw data collected during the audit (headers, DNS, responses) |

## Methodology

- **TLS**: Protocol negotiation, certificate chain, cipher analysis
- **Headers**: HTTP response header audit against OWASP recommendations
- **CSP**: Directive-level analysis for bypass vectors
- **DNS**: SPF, DMARC, DKIM, MX record validation
- **Paths**: Sensitive file exposure testing (`.env`, `.git`, `/api`)
- **Performance**: TTFB, compression, caching headers, page weight

## License

This audit was conducted for informational purposes. All testing was performed externally using publicly accessible endpoints only.
