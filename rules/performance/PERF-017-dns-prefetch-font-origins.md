# PERF-017 | `dns-prefetch` instead of `preconnect` for font origins

**Category:** Performance

**Severity:** 🟢 LOW

**Affects:** Layout files (`theme.liquid`), any template with `<link rel="dns-prefetch">` targeting font CDN origins

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`<link rel="dns-prefetch">` and `<link rel="preconnect">` are both resource hint directives that instruct the browser to perform early network work for an origin before that origin's resources are explicitly requested. The difference is in scope: `dns-prefetch` initiates only DNS resolution — it translates the hostname to an IP address and caches the result. It does not open a TCP connection and does not perform a TLS handshake. `preconnect` performs all three operations in sequence: DNS resolution, TCP handshake (3-way SYN/SYN-ACK/ACK), and TLS negotiation (1–2 round-trips of cryptographic key exchange). For a font origin like `fonts.googleapis.com` or `fonts.gstatic.com` — where every resource is served over HTTPS and CORS — `dns-prefetch` provides only the DNS portion of the connection setup savings. The remaining TCP + TLS work (typically 80–250ms on broadband, 200–500ms on mobile) is still paid at resource discovery time.

The practical gap between the two hints is measurable. DNS lookup time on a warm resolver is 10–40ms; on a cold resolver or geographically distant DNS server it can reach 80–120ms. TCP + TLS round-trips on a typical HTTPS connection to a CDN origin add 80–200ms on broadband and 200–450ms on mobile LTE. Using `dns-prefetch` instead of `preconnect` for a font origin saves at most 40–120ms (the DNS portion) while leaving 80–450ms of avoidable latency on the table. For a Google Fonts setup serving both a CSS file from `fonts.googleapis.com` and WOFF2 files from `fonts.gstatic.com`, the difference between `dns-prefetch` and `preconnect` on both origins can translate to 160–900ms of additional font load latency — a range that spans from "imperceptible" to "text swap visible to users."

The `preconnect` hint is supported in all modern browsers (Chrome 46+, Firefox 39+, Safari 11.1+, Edge 79+). The only scenario in which `dns-prefetch` is the preferred choice is for speculative origins — origins the page might connect to under certain conditions but will not always use. `preconnect` ties up a TCP connection slot and the browser must keep the connection alive for a minimum period; for origins that are never used on a given page load, this is wasted overhead. For font origins that are unconditionally loaded in the `<head>` on every page, the connection is guaranteed to be used and `preconnect` is strictly superior with no downside. This rule targets the specific case of font CDN origins (Google Fonts, Adobe Fonts, custom font hosts) where `dns-prefetch` appears and should be upgraded to `preconnect`.

---

## Detection Logic (AST)

1. **Identify `HtmlElement[link]` nodes with `rel="dns-prefetch"`.** Walk all `HtmlElement` nodes with `tagName === "link"`. For each, inspect the `HtmlAttribute` array for an attribute with `name === "rel"` and string value `"dns-prefetch"`. Collect all matching nodes as candidates.

2. **Extract and classify the `href` origin.** For each `dns-prefetch` node, inspect the `href` attribute value. Check whether the hostname matches a known font CDN pattern: `fonts.googleapis.com`, `fonts.gstatic.com`, `use.typekit.net`, `p.typekit.net`, `fonts.bunny.net`, or any hostname containing `fonts` as a subdomain or path segment. Matches are primary violations (font origins benefit most from full TCP+TLS warmup).

3. **Check for a redundant or missing `preconnect` sibling.** For each flagged `dns-prefetch` origin, scan sibling `HtmlElement[link]` nodes in the same parent for a `rel="preconnect"` with a matching `href` origin. If a `preconnect` already exists for the same origin, the `dns-prefetch` is redundant (browsers that support `preconnect` ignore `dns-prefetch` for the same origin); emit LOW advisory recommending removal of the `dns-prefetch`. If no `preconnect` exists, emit LOW advisory recommending upgrade to `preconnect`.

4. **Note `crossorigin` requirement for CORS font origins.** For origins that serve font files (`fonts.gstatic.com`, `p.typekit.net`), verify that the recommended `preconnect` replacement includes `crossorigin` attribute. If the existing `dns-prefetch` node lacks `crossorigin`, include in the fix advisory that `crossorigin` must be added when upgrading to `preconnect` for font file origins.

**Why regex fails here:** A regex scanning for `dns-prefetch` could identify the hint but cannot determine whether a `preconnect` hint for the same origin already exists elsewhere in the document — that requires cross-node comparison within a shared parent context. Additionally, determining whether an origin is a font CDN (and therefore a high-value preconnect target versus a speculative origin where `dns-prefetch` is legitimately preferred) requires hostname classification logic that cannot be reliably encoded in a single regex pattern without significant false-positive and false-negative rates.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: dns-prefetch used for font CDN origins instead of preconnect.

  Failure mode 1: fonts.googleapis.com gets DNS resolution only (~20–80ms
  saved). The TCP handshake and TLS negotiation (~120–300ms combined) are
  still paid at the time the browser discovers the Google Fonts stylesheet.
  The connection setup overhead sits on the critical path.

  Failure mode 2: fonts.gstatic.com (WOFF2 host) gets the same incomplete
  treatment. Each WOFF2 file request incurs full TCP+TLS overhead because
  dns-prefetch did not open a connection — only resolved the address.

  Failure mode 3: The crossorigin attribute is absent from the dns-prefetch
  for fonts.gstatic.com — even if upgraded to preconnect, forgetting
  crossorigin puts the connection in the wrong pool and it is not reused
  for the actual CORS font fetch, making the hint useless.

  Failure mode 4: Total unrecovered latency: ~200–500ms per page load on
  mobile that preconnect would have eliminated entirely.
{%- endcomment -%}

<!doctype html>
<html lang="{{ request.locale.iso_code }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Suboptimal: dns-prefetch only resolves DNS, leaves TCP+TLS unprepared -->
  <link rel="dns-prefetch" href="https://fonts.googleapis.com">
  <link rel="dns-prefetch" href="https://fonts.gstatic.com">
  <link rel="dns-prefetch" href="https://use.typekit.net">

  <link
    rel="stylesheet"
    href="https://fonts.googleapis.com/css2?family=Lato:wght@300;400;700&display=swap"
  >
  <link rel="stylesheet" href="https://use.typekit.net/xyz9876.css">

  {{ content_for_header }}
  {{ 'theme.css' | asset_url | stylesheet_tag }}
</head>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Upgrade dns-prefetch to preconnect for all font CDN origins.

  Change 1: fonts.googleapis.com upgraded from dns-prefetch to preconnect.
  The browser now performs DNS + TCP + TLS for this origin during HTML
  parsing. When the stylesheet link is reached, the connection is ready
  and the CSS fetch begins immediately with no setup overhead.

  Change 2: fonts.gstatic.com upgraded to preconnect with crossorigin.
  Font files are fetched as anonymous CORS requests (crossorigin="anonymous"
  is the default). Without crossorigin on the preconnect, the browser
  opens a non-CORS connection that cannot be reused for CORS font fetches,
  rendering the preconnect hint entirely ineffective.

  Change 3: use.typekit.net upgraded to preconnect. Adobe Fonts CSS and
  kit JavaScript are served from this origin; warming the connection
  reduces render-blocking time for the kit stylesheet.

  Change 4: dns-prefetch hints removed entirely. Browsers supporting
  preconnect ignore redundant dns-prefetch for the same origin; keeping
  both is noise that clutters the resource hint list without benefit.
{%- endcomment -%}

<!doctype html>
<html lang="{{ request.locale.iso_code }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Full DNS+TCP+TLS warmup for Google Fonts CSS delivery endpoint -->
  <link rel="preconnect" href="https://fonts.googleapis.com">

  <!-- Full warmup for WOFF2 host; crossorigin required for CORS font fetches -->
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

  <!-- Full warmup for Adobe Fonts kit endpoint -->
  <link rel="preconnect" href="https://use.typekit.net">

  <link
    rel="stylesheet"
    href="https://fonts.googleapis.com/css2?family=Lato:wght@300;400;700&display=swap"
  >
  <link rel="stylesheet" href="https://use.typekit.net/xyz9876.css">

  {{ content_for_header }}
  {{ 'theme.css' | asset_url | stylesheet_tag }}
</head>
```

---

## References

- [MDN — `<link rel="preconnect">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preconnect)
- [MDN — `<link rel="dns-prefetch">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/dns-prefetch)
- [web.dev — Preconnect and dns-prefetch](https://web.dev/articles/preconnect-and-dns-prefetch)
- [Google — Preconnect to required origins (Lighthouse)](https://developer.chrome.com/docs/lighthouse/performance/uses-rel-preconnect/)
- [web.dev — Font best practices](https://web.dev/articles/font-best-practices)
