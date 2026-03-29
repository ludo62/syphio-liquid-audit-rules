# PERF-016 | Missing `preconnect` for critical third-party origins

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Layout files (`theme.liquid`, `password.liquid`), any template that loads above-the-fold resources from third-party origins

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

When a browser encounters a resource URL from an origin it has not previously connected to, it must complete three sequential network operations before the first byte of the resource can transfer: DNS resolution (the browser queries a DNS resolver to translate the hostname to an IP address), TCP handshake (a three-way SYN/SYN-ACK/ACK exchange to open a connection), and TLS negotiation (a 1–2 round-trip cryptographic handshake to establish an encrypted channel). On a typical broadband connection, this sequence adds 100–300ms of latency per origin. On mobile networks with higher per-packet round-trip times, the overhead can reach 400–600ms. This overhead is unavoidable on the resource's first request — it cannot be eliminated by HTTP/2 multiplexing or CDN optimization because it precedes any HTTP-layer communication.

`<link rel="preconnect" href="https://example.com">` instructs the browser to initiate the DNS+TCP+TLS sequence for the specified origin in parallel with HTML parsing, before the browser has encountered any resource from that origin in the document. For origins that serve above-the-fold resources — Google Fonts CSS, Shopify CDN font files, hero image CDNs, analytics scripts that affect rendering — preconnecting eliminates the connection setup latency from the critical path entirely. In real-world measurements, preconnect for Google Fonts origins improves LCP by 80–250ms on desktop and 150–400ms on mobile. For a Shopify theme that loads a custom font, an analytics library, and a third-party widget in the `<head>`, the cumulative unpreconnected overhead can exceed 800ms on the critical render path.

The detection target for this rule is the gap between third-party origins that appear in resource URLs in the `<head>` and the set of `<link rel="preconnect">` hints present in the same `<head>`. Origins serving fonts, render-blocking stylesheets, or scripts loaded without `async`/`defer` are highest priority. Shopify's own CDN (`cdn.shopify.com`) does not require preconnect — browsers receive that origin from Shopify's infrastructure early in the connection lifecycle — but Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`), Adobe Fonts (`use.typekit.net`, `p.typekit.net`), and common app CDNs do. The rule fires when a third-party origin is detected in a resource attribute without a corresponding `preconnect` hint in the same document head.

---

## Detection Logic (AST)

1. **Collect all `HtmlElement[link]` nodes with resource-bearing `rel` values.** Walk the AST for `HtmlElement` nodes with `tagName === "link"`. Collect nodes where the `rel` attribute value is `"stylesheet"`, `"preload"`, or `"modulepreload"`. For each, extract the `href` attribute value and parse the origin (scheme + hostname).

2. **Collect all `HtmlElement[script]` nodes with `src` attributes.** Walk `HtmlElement` nodes with `tagName === "script"`. For each with a `src` attribute, extract the origin from the `src` value. Exclude nodes with a `defer` or `async` attribute only if the origin's resources are not render-critical (heuristic: if no matching preconnect exists, still emit LOW for deferred third-party scripts).

3. **Build a set of preconnected origins.** Collect all `HtmlElement[link]` nodes where `rel === "preconnect"` or `rel === "dns-prefetch"`. Extract the `href` value as the preconnected origin. Origins covered by `preconnect` are compliant; origins covered only by `dns-prefetch` are flagged separately by PERF-017.

4. **Diff resource origins against preconnected origins.** For each third-party resource origin (any origin not matching the current Shopify store's `myshopify.com` or `cdn.shopify.com` domains), check whether it appears in the preconnected origins set. Any third-party origin serving a render-blocking resource without a matching `preconnect` is a violation with severity MEDIUM. Third-party origins serving deferred or async resources without `preconnect` are advisory LOW.

**Why regex fails here:** A regex approach requires two separate scans — one for resource URLs, one for preconnect hints — and then string-matching of extracted origins. Origin extraction from URLs is not reliably achievable with a single regex: URLs may be partially Liquid-interpolated (e.g., `href="{{ cdn_url }}"`), may use `//`-relative protocol syntax, or may have subdomains that are distinct preconnect targets. The AST walker resolves `HtmlAttribute.value` nodes as typed structures, handles Liquid interpolation within attribute values as `LiquidDrop` nodes, and can perform origin normalization before the set-diff check — operations that a line-level regex cannot perform without multi-pass coordination.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Three distinct third-party origins loaded in the <head>
  with no preconnect hints for any of them.

  Failure mode 1: fonts.googleapis.com CSS request incurs full DNS+TCP+TLS
  before the @font-face declarations can be discovered. On mobile this
  adds 150–300ms before font loading even begins.

  Failure mode 2: fonts.gstatic.com (WOFF2 host) has no preconnect — even
  after the Google Fonts CSS is fetched and parsed, each font file incurs
  its own DNS+TCP+TLS round-trip on first request.

  Failure mode 3: use.typekit.net (Adobe Fonts) is a render-blocking
  stylesheet with no preconnect, adding another independent 100–300ms
  connection overhead on the critical path.

  Failure mode 4: The cumulative unoptimized overhead across three origins
  can reach 600–900ms of avoidable connection latency before any above-fold
  content is visible — a direct LCP regression.
{%- endcomment -%}

<!doctype html>
<html lang="{{ request.locale.iso_code }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>

  <link
    rel="stylesheet"
    href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap"
  >

  <link
    rel="stylesheet"
    href="https://use.typekit.net/abc1234.css"
  >

  {{ content_for_header }}

  {{ 'theme.css' | asset_url | stylesheet_tag }}
</head>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add preconnect hints for every third-party origin that serves
  above-the-fold or render-blocking resources.

  Change 1: preconnect to fonts.googleapis.com placed before the stylesheet
  link. The browser begins DNS+TCP+TLS for this origin immediately during
  HTML parsing, so the connection is ready when the parser reaches the
  <link rel="stylesheet"> element. Latency drops from ~200ms to ~0ms.

  Change 2: preconnect to fonts.gstatic.com with crossorigin attribute.
  WOFF2 files are fetched as anonymous CORS requests; crossorigin ensures
  the preconnected connection is placed in the correct connection pool
  and reused for actual font file fetches.

  Change 3: preconnect to use.typekit.net before the Adobe Fonts stylesheet.
  Same principle — connection warmup precedes resource discovery.

  Change 4: Preconnect hints placed as the first resource hints in <head>,
  before any other link or script elements, to maximize the lead time
  available for connection setup during HTML parsing.
{%- endcomment -%}

<!doctype html>
<html lang="{{ request.locale.iso_code }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>

  <!-- Preconnect: fonts.googleapis.com (Google Fonts CSS endpoint) -->
  <link rel="preconnect" href="https://fonts.googleapis.com">

  <!-- Preconnect: fonts.gstatic.com (WOFF2 files); crossorigin required for CORS -->
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

  <!-- Preconnect: Adobe Fonts CSS + kit delivery origin -->
  <link rel="preconnect" href="https://use.typekit.net">

  <link
    rel="stylesheet"
    href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap"
  >

  <link
    rel="stylesheet"
    href="https://use.typekit.net/abc1234.css"
  >

  {{ content_for_header }}

  {{ 'theme.css' | asset_url | stylesheet_tag }}
</head>
```

---

## References

- [MDN — `<link rel="preconnect">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preconnect)
- [web.dev — Establish network connections early](https://web.dev/articles/preconnect-and-dns-prefetch)
- [Google — Preconnect to required origins (Lighthouse audit)](https://developer.chrome.com/docs/lighthouse/performance/uses-rel-preconnect/)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [web.dev — Optimize resource loading with the Fetch Priority API](https://web.dev/articles/fetch-priority)
