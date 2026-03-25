# 🔒 TechOptima.ai — Security & Performance Audit Report

**Date:** 2026-03-25  
**Target:** `https://techoptima.ai`  
**Server:** nginx (IP: `49.205.64.2`)  
**Tech Stack:** React SPA (MUI/Material-UI), client-side routing  

---

## Executive Summary

TechOptima.ai demonstrates a **solid baseline security posture** with strong TLS, HTTP security headers, and Content Security Policy. However, there are several areas requiring attention — most notably **DMARC enforcement, CSP hardening, missing cache controls**, and **SPA-related SEO/security implications**.

| Area | Grade | Notes |
|------|-------|-------|
| **TLS/SSL** | ✅ A | TLS 1.3 only, valid cert |
| **HTTP Headers** | ✅ B+ | All major headers present; CSP has `unsafe-inline`/`unsafe-eval` |
| **CSP Policy** | ⚠️ B- | Functional but weakened by `unsafe-inline` + `unsafe-eval` |
| **HTTPS Enforcement** | ✅ A | 301 redirect + HSTS preload |
| **Email Security** | 🔴 D | DMARC `p=none`, no SPF record found |
| **Caching** | 🔴 F | No Cache-Control headers |
| **Performance** | ⚠️ C+ | 178KB page, TTFB ~416ms, Brotli enabled |
| **Sensitive Path Handling** | ⚠️ B | SPA fallback returns 200 for all paths (not a real leak, but impacts crawlers) |

---

## 1. TLS/SSL Configuration ✅

| Property | Value |
|----------|-------|
| Protocol | **TLS 1.3 only** (TLS 1.2 rejected) |
| Certificate | Sectigo DV (Domain Validated), wildcard `*.techoptima.ai` |
| Key | RSA **2048-bit**, SHA-256 signature |
| Valid | Sep 4, 2025 → Oct 5, 2026 |
| SANs | `*.techoptima.ai`, `techoptima.ai` |
| Chain | Full (4 certs: leaf → DV R36 → Root R46 → USERTrust RSA) |
| CT Logs | ✅ 3 SCT entries present |

> [!TIP]
> **Strong configuration.** TLS 1.3 only is best practice. Consider upgrading to **ECDSA P-256** key for better performance (smaller handshake, faster ECDH).

---

## 2. HTTP Security Headers

### Present ✅

| Header | Value | Assessment |
|--------|-------|------------|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | ✅ Excellent (2-year max-age with preload) |
| `X-Frame-Options` | `DENY` | ✅ Clickjacking protection |
| `X-Content-Type-Options` | `nosniff` | ✅ MIME sniffing prevention |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | ✅ Good balance of privacy/functionality |
| `X-XSS-Protection` | `1; mode=block` | ⚠️ Deprecated; CSP supersedes this |
| `Content-Security-Policy` | See detailed analysis below | ⚠️ Has weaknesses |

### Missing 🔴

| Header | Risk | Recommendation |
|--------|------|----------------|
| `Permissions-Policy` | Medium | Restrict camera, microphone, geolocation, etc. |
| `Cross-Origin-Opener-Policy` | Low | Add `same-origin` for Spectre mitigation |
| `Cross-Origin-Resource-Policy` | Low | Add `same-site` to prevent cross-origin resource loading |
| `Cross-Origin-Embedder-Policy` | Low | Add for `SharedArrayBuffer` isolation |

---

## 3. Content Security Policy — Detailed Analysis

```
default-src 'self';
script-src 'self' 'unsafe-inline' 'unsafe-eval' https://www.googletagmanager.com https://www.google-analytics.com;
style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src 'self' https://fonts.gstatic.com;
img-src 'self' data: https://img.freepik.com https://media4.giphy.com https://helpdesk.techoptima.ai https://media.istockphoto.com https://encrypted-tbn0.gstatic.com https://lh3.googleusercontent.com;
connect-src 'self' https://helpdesk.techoptima.ai https://ipapi.co https://www.google-analytics.com https://stats.g.doubleclick.net;
frame-ancestors 'none';
object-src 'none';
```

> [!WARNING]
> **`'unsafe-inline'` + `'unsafe-eval'` in `script-src`** significantly weakens XSS protection. These effectively bypass CSP's primary defense against script injection.

| Directive | Issue | Severity |
|-----------|-------|----------|
| `script-src 'unsafe-inline'` | Allows inline `<script>` and event handlers — XSS risk | 🔴 High |
| `script-src 'unsafe-eval'` | Allows `eval()`, `Function()`, `setTimeout(string)` | 🔴 High |
| `style-src 'unsafe-inline'` | Allows inline styles (lower risk, but weakens defense-in-depth) | ⚠️ Medium |
| `img-src data:` | Allows data URIs for images (potential exfiltration vector) | ⚠️ Low |
| Missing `base-uri` | No `base-uri` directive — allows `<base>` tag injection | ⚠️ Medium |
| Missing `form-action` | No `form-action` directive — forms can submit anywhere | ⚠️ Medium |
| Missing `upgrade-insecure-requests` | Should auto-upgrade HTTP to HTTPS | ⚠️ Low |

**Recommendations:**
1. Replace `'unsafe-inline'` with **nonces** (`'nonce-<random>'`) or **hashes**
2. Remove `'unsafe-eval'` — refactor code to avoid `eval()`
3. Add `base-uri 'self'; form-action 'self'; upgrade-insecure-requests;`

---

## 4. HTTPS Enforcement ✅

```
HTTP/1.1 301 Moved Permanently
Location: https://techoptima.ai/
```

- ✅ HTTP → HTTPS redirect working (301 permanent)
- ✅ HSTS with `preload` flag and `includeSubDomains`

> [!NOTE]
> The HTTP redirect response has `X-Frame-Options: SAMEORIGIN` vs the HTTPS response which has `DENY`. These should be consistent — both should use `DENY`.

---

## 5. Email Security (DNS) 🔴

| Record | Value | Assessment |
|--------|-------|------------|
| **SPF** | ❌ Not found | 🔴 No SPF TXT record — email spoofing possible |
| **DMARC** | `v=DMARC1; p=none;` | 🔴 **No enforcement** — only monitoring mode |
| **DKIM** | Not verifiable externally | Cannot confirm |
| **MX** | None found | Possibly using external email (no MX records) |

> [!CAUTION]
> With `p=none` DMARC and no SPF, **anyone can send emails as @techoptima.ai**. This is a significant phishing/impersonation risk, especially for an AI security company.

**Immediate actions:**
1. Add SPF record: `v=spf1 include:<email-provider> -all`
2. Upgrade DMARC to `p=quarantine` (then `p=reject` after monitoring)
3. Ensure DKIM signing is configured

---

## 6. Sensitive Path Exposure

All paths return HTTP **200** with identical content (178,216 bytes) — this is the **React SPA fallback** behavior where nginx serves `index.html` for all routes.

| Path | Status | Real Exposure? |
|------|--------|---------------|
| `/.env` | 200 | ❌ No (SPA fallback) |
| `/.git/config` | 200 | ❌ No (SPA fallback) |
| `/api` | 200 | ❌ No (SPA fallback) |
| `/swagger` | 200 | ❌ No (SPA fallback) |
| `/.well-known/security.txt` | 200 | ❌ No (SPA fallback) |

> [!IMPORTANT]
> While not a real data leak, returning **200 OK** for paths like `/.env` and `/.git/config` is problematic:
> - **Security scanners** will flag these as vulnerabilities (false positives waste time)
> - **Search engines** may index ghost routes
> - Best practice: configure nginx to return **404** for known sensitive patterns before the SPA fallback

**Recommended nginx rule:**
```nginx
location ~ /\.(env|git|svn|htaccess|htpasswd) {
    return 404;
}
```

---

## 7. Robots.txt & Sitemap

### robots.txt
```
User-agent: *
Disallow:
```
- ⚠️ **Wide open** — all crawlers can access everything
- Missing: `Sitemap:` directive pointing to sitemap.xml
- Consider restricting internal-only paths

### sitemap.xml ✅
- 41 URLs indexed
- Proper XML sitemap format with `changefreq` and `priority`

**Issues found:**
- ⚠️ URL with apostrophe: `https://techoptima.ai/ip's` — should be URL-encoded
- ⚠️ Inconsistent casing: `termsOfservices` vs `termsofservices` (case matters for some servers)
- ⚠️ Duplicate entry: `ai-process-optimization-for-bfsi-industry` appears twice
- ⚠️ All priorities set to `0.5` (default) — not useful for crawl prioritization

---

## 8. Performance Analysis

| Metric | Value | Assessment |
|--------|-------|------------|
| **DNS Lookup** | 218ms | ⚠️ Slow — consider DNS pre-warming or faster provider |
| **TCP Connect** | 275ms | Depends on geo-location |
| **TLS Handshake** | 349ms | Normal for TLS 1.3 |
| **TTFB** | **416ms** | ⚠️ Acceptable but can improve |
| **Total Load** | 591ms | Good for initial HTML |
| **Page Size** | 178KB (uncompressed HTML) | 🔴 Large for an SPA shell |
| **Compression** | Brotli (`br`) | ✅ Best-in-class |
| **HTTP Version** | HTTP/2 | ✅ Good |

### Missing Performance Headers

| Header | Impact |
|--------|--------|
| `Cache-Control` | 🔴 **No caching at all** — every visit re-downloads 178KB |
| `ETag` | ✅ Present (content-based caching possible) |
| `Expires` | ❌ Missing |

> [!WARNING]
> **No `Cache-Control` header** means browsers re-fetch the full page on every navigation. For static SPA content, this wastes bandwidth and hurts repeat-visit performance.

**Recommended:**
```
Cache-Control: public, max-age=3600, s-maxage=86400
```
For static assets (`/static/`):
```
Cache-Control: public, max-age=31536000, immutable
```

---

## 9. Additional Observations

| Finding | Details |
|---------|---------|
| **Framework** | React with Material-UI (MUI), likely Create React App (based on `/static/media/` paths) |
| **Analytics** | Google Tag Manager + Google Analytics |
| **IP Geolocation** | Uses `ipapi.co` (leaks visitor IP to third party via `connect-src`) |
| **Helpdesk** | Internal: `helpdesk.techoptima.ai` |
| **Social Links** | X (Twitter), LinkedIn, GitHub, YouTube, Instagram |
| **Compliance Badges** | SOC, ISO 27001, ISO 9001 displayed in footer |
| **security.txt** | ❌ Not implemented — consider creating `/.well-known/security.txt` |

---

## 🎯 Prioritized Recommendations

### Critical (Fix Immediately)
1. **Add SPF record** and upgrade **DMARC to `p=quarantine`** → prevents email spoofing
2. **Remove `'unsafe-eval'`** from CSP `script-src` → major XSS defense improvement

### High Priority
3. **Replace CSP `'unsafe-inline'`** with nonces or hashes
4. **Add `Cache-Control` headers** — especially for static assets
5. **Add `base-uri 'self'`** and **`form-action 'self'`** to CSP
6. **Block sensitive paths** in nginx (`.env`, `.git`, etc.) before SPA fallback

### Medium Priority
7. Add **`Permissions-Policy`** header
8. Create **`/.well-known/security.txt`** with vulnerability disclosure info
9. Add **`Sitemap:`** directive to robots.txt
10. Fix sitemap issues (duplicate entries, URL encoding, inconsistent casing)
11. Make HTTP → HTTPS redirect `X-Frame-Options` consistent (`DENY`)

### Low Priority / Nice-to-Have
12. Upgrade to **ECDSA certificate** for faster TLS handshakes
13. Add CORS headers (`Cross-Origin-Opener-Policy`, etc.) for Spectre isolation
14. Remove deprecated `X-XSS-Protection` header
15. Review `ipapi.co` integration for privacy compliance (sends visitor IPs to third party)
