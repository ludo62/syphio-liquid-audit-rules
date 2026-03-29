# PERF-015 | Google Fonts without `&display=swap`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Layout files (`theme.liquid`, `password.liquid`), any snippet or section that embeds a Google Fonts `<link>` or `@import` URL

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Google Fonts operates as a CSS delivery service: when a browser requests a Google Fonts URL (e.g., `https://fonts.googleapis.com/css2?family=Inter:wght@400;700`), the Google servers return a CSS file containing `@font-face` declarations that point to WOFF2 font files hosted on `fonts.gstatic.com`. The `display` URL query parameter controls the `font-display` value injected into those `@font-face` declarations server-side. When the `display` parameter is absent, Google Fonts defaults to emitting `font-display: block` in the served CSS — not `auto`, but explicitly `block`. This means the browser enforces a block period of up to 3 seconds during which all text using any of the requested font families is completely invisible to the user. On connections slower than ~5 Mbps, which encompasses a significant portion of mobile users globally, the block period frequently reaches its 3-second maximum.

The LCP consequence is identical to native FOIT: if any text element that is the Largest Contentful Paint candidate uses a Google Font, the LCP timestamp is delayed until the text becomes visible. Google's own PageSpeed Insights tooling flags `font-display: block` (which is what the missing `display` parameter produces) as a render-blocking font load and deducts LCP score accordingly. In A/B tests documented by web performance engineers, adding `&display=swap` to an existing Google Fonts URL — with no other changes — produced LCP improvements of 200–600ms on mobile 3G profiles. This is a single-parameter change that eliminates the block period entirely and moves text rendering to first paint using the system fallback font.

A complete Google Fonts optimization requires three components working together: the `&display=swap` parameter on the font URL itself; a `<link rel="preconnect" href="https://fonts.googleapis.com">` hint so the browser initiates DNS+TCP+TLS to the CSS delivery endpoint before it parses the `<link>` element; and a `<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>` hint so the browser also warms the connection to the WOFF2 file server (the `crossorigin` attribute is required here because font files are fetched as CORS requests). Without the `gstatic.com` preconnect, the WOFF2 files themselves still incur the full DNS+TCP+TLS round-trip — typically 100–250ms — even if `display=swap` eliminates FOIT. The three changes together maximize both perceived performance (FOIT eliminated) and actual font load speed (connection overhead reduced).

---

## Detection Logic (AST)

1. **Identify `HtmlElement` nodes with `tagName === "link"` and `rel="stylesheet"`.** The AST walker finds all `HtmlElement` nodes. For each, inspect the `HtmlAttribute` list for an attribute with `name === "rel"` and value `"stylesheet"`. Nodes matching this shape are stylesheet link candidates.

2. **Inspect the `href` attribute value for the Google Fonts hostname.** On each candidate node, locate the `HtmlAttribute` with `name === "href"`. If the attribute value string contains `fonts.googleapis.com`, the node is a Google Fonts `<link>` element and is subject to this rule.

3. **Parse the `href` URL query string for the `display` parameter.** Extract the query string portion of the `href` value. If the query string does not contain `display=swap` (case-insensitive match on the value), the node is a violation. Note: `display=block`, `display=fallback`, and `display=optional` are also flagged with severity MEDIUM and a note indicating the non-swap display mode.

4. **Check for accompanying `preconnect` hints.** Walk all sibling `HtmlElement` nodes in the same parent context. If no sibling has `rel="preconnect"` with `href` containing `fonts.googleapis.com`, emit an additional MEDIUM-severity advisory. If no sibling has `rel="preconnect"` with `href` containing `fonts.gstatic.com`, emit a separate MEDIUM-severity advisory for the missing WOFF2 origin preconnect.

**Why regex fails here:** A regex matching `fonts.googleapis.com` without `display=swap` would produce false positives on URLs that use `display=swap` but with different parameter ordering (e.g., `?family=Inter&display=swap&subset=latin`). Parsing URL query strings with regex is brittle — the `display` parameter may appear in any position, be URL-encoded, or be present in a Liquid variable interpolation within the href. The AST walker resolves the `HtmlAttribute.value` as a typed node (potentially a `LiquidDrop` or a concatenated string), enabling structured URL parsing separate from raw pattern matching.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Google Fonts loaded without &display=swap, without preconnect
  hints, and using a render-blocking <link> placement.

  Failure mode 1: Absent display parameter — Google servers return @font-face
  blocks with font-display: block. Text is invisible for up to 3 seconds on
  slow connections. This is not browser-default behavior; Google explicitly
  sets block when the parameter is absent.

  Failure mode 2: No preconnect to fonts.googleapis.com — the CSS request
  incurs full DNS+TCP+TLS overhead (~150ms) at discovery time, delaying
  even the start of the @font-face CSS fetch until after HTML parsing has
  already begun building the render tree.

  Failure mode 3: No preconnect to fonts.gstatic.com — the WOFF2 files
  incur their own DNS+TCP+TLS round-trip, adding another 100–250ms
  to actual font load time regardless of display mode.

  Failure mode 4: Two font families loaded in separate requests instead of
  a single combined URL — doubles the CSS request count unnecessarily.
{%- endcomment -%}

<head>
  <meta charset="UTF-8">
  <title>{{ page_title }}</title>

  <link
    rel="stylesheet"
    href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700&family=Playfair+Display:ital,wght@0,700;1,400"
  >

  {{ content_for_header }}

  {{ 'base.css' | asset_url | stylesheet_tag }}
</head>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add &display=swap to the Google Fonts URL and add preconnect hints
  for both fonts.googleapis.com and fonts.gstatic.com.

  Change 1: &display=swap appended to the combined Google Fonts URL. Google's
  servers now emit font-display: swap in every @font-face block in the
  returned CSS. Text is rendered immediately in the fallback system font;
  no FOIT window occurs.

  Change 2: preconnect to fonts.googleapis.com warms DNS+TCP+TLS for the
  CSS delivery endpoint. The stylesheet fetch begins immediately when the
  browser parser reaches the <link rel="stylesheet"> element.

  Change 3: preconnect to fonts.gstatic.com with crossorigin attribute.
  Font files are fetched as anonymous CORS requests; crossorigin is required
  for the preconnect to cover the correct connection pool. Without it,
  the browser opens a second non-CORS connection that is not reused for
  the actual font fetch.

  Change 4: Both font families combined into a single URL — one CSS request
  instead of two, halving the stylesheet fetch round-trip count.
{%- endcomment -%}

<head>
  <meta charset="UTF-8">
  <title>{{ page_title }}</title>

  <!-- Warm DNS+TCP+TLS for the Google Fonts CSS endpoint -->
  <link rel="preconnect" href="https://fonts.googleapis.com">

  <!-- Warm DNS+TCP+TLS for the WOFF2 file server; crossorigin required -->
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

  <!-- Combined URL with display=swap: eliminates FOIT for both families -->
  <link
    rel="stylesheet"
    href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700&family=Playfair+Display:ital,wght@0,700;1,400&display=swap"
  >

  {{ content_for_header }}

  {{ 'base.css' | asset_url | stylesheet_tag }}
</head>
```

---

## References

- [Google Fonts — `display` parameter documentation](https://developers.google.com/fonts/docs/css2#use_font-display)
- [web.dev — Optimize Google Fonts loading](https://web.dev/articles/font-best-practices#google_fonts)
- [MDN — `<link rel="preconnect">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preconnect)
- [web.dev — Avoid Flash of Invisible Text (FOIT)](https://web.dev/articles/avoid-invisible-text)
- [Google — Largest Contentful Paint (LCP)](https://web.dev/articles/lcp)
