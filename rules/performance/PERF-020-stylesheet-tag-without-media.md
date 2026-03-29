# PERF-020 | CSS loaded via `stylesheet_tag` without `media` attribute

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Layout files, section files, and snippets that use the `stylesheet_tag` filter to load CSS assets

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

The `stylesheet_tag` Liquid filter generates a `<link rel="stylesheet" href="...">` element from an asset URL. It is a convenience wrapper that accepts a single input (the asset URL string) and produces a complete stylesheet link element. The filter does not accept any named arguments — there is no `media:` parameter, no `crossorigin:` parameter, no `fetchpriority:` parameter. The generated `<link>` element always omits the `media` attribute, which causes every browser to treat the stylesheet as a universal, render-blocking resource applicable to all media types and all viewports. This is correct behavior for a theme's main stylesheet; it is incorrect behavior for stylesheets that are only relevant to a specific media type (print), a specific viewport width (desktop-only or mobile-only CSS), or a specific interaction state (dark mode).

A stylesheet without a `media` attribute is render-blocking by definition: the browser must fetch and fully parse it before it can proceed with rendering. For a print stylesheet (typically 5–15KB), loading it without `media="print"` adds its full parse time to the critical render path even on screen media where it has no effect. For a desktop-only stylesheet loaded on a mobile device, the browser fetches and parses CSS rules that apply to zero elements in the current viewport. In aggregate, themes that load 3–5 CSS files without media differentiation can inflate Time to First Paint by 50–200ms compared to themes that correctly scope each stylesheet to its media context. The browser also cannot preload the correct critical CSS if all stylesheets are treated equally.

The `stylesheet_tag` filter is not fixable — it does not support `media` attribute injection. The correct fix is to replace `stylesheet_tag` with a manually constructed `<link>` element that includes the appropriate `media` attribute. For the primary theme stylesheet, `media="all"` (or omitting the attribute entirely) is correct and produces no regression. For print stylesheets, `media="print"` is required. For stylesheets conditionally relevant to specific breakpoints, `media="(min-width: 990px)"` or `media="(max-width: 749px)"` scopes the browser's fetch-and-parse work to contexts where the CSS applies. Note that stylesheets with a `media` attribute that does not match the current context are still fetched (to handle viewport resize and orientation change) but are fetched at lower priority and do not block rendering.

---

## Detection Logic (AST)

1. **Identify `LiquidFilter` nodes named `stylesheet_tag`.** The AST walker traverses all `LiquidVariable` nodes and inspects the `filters` array. Any filter with `name === "stylesheet_tag"` is a candidate for this rule. Collect the full filter chain to determine what URL is being passed (e.g., whether it originates from `asset_url`, a CDN URL, or a Liquid variable).

2. **Inspect the filter's `args` array for a `media` named argument.** Because `stylesheet_tag` does not support named arguments, the `args` array will always be empty. The violation is structural — the filter itself is incapable of emitting a `media` attribute. Every use of `stylesheet_tag` where a `media` scope is semantically relevant is a violation.

3. **Classify the stylesheet by URL pattern to determine severity.** If the URL string (or the variable feeding the filter chain) contains `print`, `narrow`, `mobile`, `wide`, `desktop`, or similar media-scope vocabulary, the omitted `media` attribute is a confirmed functional regression — severity MEDIUM. For a generic `theme.css` or `base.css` without a media-scope indicator, emit LOW advisory noting that `stylesheet_tag` cannot support future media scoping without a manual `<link>` refactor.

4. **Check for multiple `stylesheet_tag` calls in the same layout head.** If three or more `stylesheet_tag` calls appear in a single layout file's `<head>`, emit an aggregate MEDIUM finding noting that the inability to set `media` attributes on any of them means the browser treats all as equally render-blocking, regardless of their actual applicability to the current media context.

**Why regex fails here:** A regex matching `stylesheet_tag` cannot determine whether the filter is the terminal filter in a chain (where the output goes directly to HTML) or an intermediate step (rare but possible). More importantly, identifying whether a stylesheet is media-scoped requires understanding the semantic content of the URL string being passed — a task that requires context from the broader filter chain and variable assignments that feed into the stylesheet URL. The AST walker resolves the complete filter chain from the `LiquidVariable` node, including all upstream `assign` nodes that supply the URL input.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: stylesheet_tag used for all CSS assets including a print
  stylesheet and a desktop-only layout file.

  Failure mode 1: print.css loaded without media="print". The browser fetches
  and parses print styles on every screen page load, blocking initial render
  even though print CSS never applies to screen media. Adds ~10–30ms to
  Time to First Paint on every page load.

  Failure mode 2: component-desktop.css (desktop-only layout overrides) loaded
  without media="(min-width: 990px)". Mobile devices fetch and parse CSS
  rules for zero applicable elements — wasted bandwidth and parse time.

  Failure mode 3: stylesheet_tag cannot be fixed with a named argument —
  there is no media: parameter available. The only remedy is replacing
  the filter call with a manual <link> element.

  Failure mode 4: All three stylesheets are treated as equal render-blocking
  resources. The browser's resource scheduler cannot deprioritize any of
  them relative to the primary theme CSS.
{%- endcomment -%}

<!doctype html>
<html lang="{{ request.locale.iso_code }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  {{ content_for_header }}

  {{ 'base.css' | asset_url | stylesheet_tag }}
  {{ 'component-desktop.css' | asset_url | stylesheet_tag }}
  {{ 'print.css' | asset_url | stylesheet_tag }}
  {{ 'theme.css' | asset_url | stylesheet_tag }}
</head>
<body>
  {{ content_for_layout }}
</body>
</html>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace stylesheet_tag with manual <link> elements to enable
  media attribute scoping on each stylesheet.

  Change 1: base.css and theme.css retain media="all" (or no media attribute)
  — they are legitimately render-blocking for all media types. The manual
  <link> construction matches what stylesheet_tag would emit but now allows
  future media scoping without a filter change.

  Change 2: component-desktop.css gets media="(min-width: 990px)". Mobile
  browsers still fetch the file (for viewport resize handling) but at
  lower priority and without blocking initial render. Parse time is deferred
  until after First Contentful Paint on mobile.

  Change 3: print.css gets media="print". The browser fetches it at the
  lowest priority (after all screen-media resources) and does not block
  rendering for any screen page load. On screen media the file has zero
  render impact.

  Change 4: asset_url filter is still used for CDN URL generation —
  only the terminal stylesheet_tag filter is replaced with the manual
  <link> element construction. The CDN URL benefit is preserved.
{%- endcomment -%}

<!doctype html>
<html lang="{{ request.locale.iso_code }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  {{ content_for_header }}

  <!-- Primary stylesheet: render-blocking for all media, correct -->
  <link rel="stylesheet" href="{{ 'base.css' | asset_url }}" media="all">

  <!-- Desktop layout overrides: non-blocking on mobile viewports -->
  <link rel="stylesheet" href="{{ 'component-desktop.css' | asset_url }}" media="(min-width: 990px)">

  <!-- Print styles: fetched at lowest priority, never blocks screen rendering -->
  <link rel="stylesheet" href="{{ 'print.css' | asset_url }}" media="print">

  <!-- Theme overrides: render-blocking for all media -->
  <link rel="stylesheet" href="{{ 'theme.css' | asset_url }}" media="all">
</head>
<body>
  {{ content_for_layout }}
</body>
</html>
```

---

## References

- [Shopify — `stylesheet_tag` filter](https://shopify.dev/docs/api/liquid/filters/stylesheet_tag)
- [MDN — `<link>` media attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/link#media)
- [web.dev — Defer non-critical CSS](https://web.dev/articles/defer-non-critical-css)
- [Google — Eliminate render-blocking resources (Lighthouse)](https://developer.chrome.com/docs/lighthouse/performance/render-blocking-resources/)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
