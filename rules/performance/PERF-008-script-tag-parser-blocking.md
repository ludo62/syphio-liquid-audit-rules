# PERF-008 | `{{ 'file.js' | asset_url | script_tag }}` without `defer`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** All templates and layouts using the `script_tag` filter to load JavaScript assets — `theme.liquid`, section files, snippet files

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

The `script_tag` filter generates a synchronous `<script src="..."></script>` element — no `async` or `defer` attribute is included in the generated markup, and the filter provides no mechanism to add them. A synchronous script tag encountered by the browser's HTML parser is parser-blocking: the parser halts, the browser dispatches a network request for the script file, waits for the full response, executes the script in the JavaScript engine, and only then resumes HTML parsing. Every millisecond the script takes to download and execute is a millisecond of zero forward progress on HTML parsing. Render-blocking behavior directly inflates First Contentful Paint (FCP) and Total Blocking Time (TBT) — two of the five Core Web Vitals metrics evaluated by PageSpeed Insights and used as Google ranking signals.

On a typical Shopify theme that uses `script_tag` for 3–5 JavaScript files — a slider library, a form validator, an analytics integration, a cookie consent script, and a custom theme script — the cumulative parser-blocking time is the sum of all network round-trips plus execution times for each file. On a 4G connection with 80ms round-trip time and files averaging 30KB minified, each script adds approximately 160–300ms of blocking time. Five scripts: 800ms–1.5s of cumulative parser-blocking before any content below the script tags is rendered. Shopify's CDN edge delivers assets with very low latency, but even sub-100ms per-file network times accumulate to significant blocking budgets across 5 files. The Total Blocking Time "Good" threshold is under 200ms total across the page load.

Shopify's Theme Store requirements, updated in 2023, mandate that all non-critical JavaScript use `defer` or `async` loading. The `script_tag` filter cannot be updated to emit these attributes — it is a filter with fixed output. Replacing `script_tag` with a manually constructed `<script defer src="{{ 'file.js' | asset_url }}"></script>` is the only correct remediation. `defer` is preferred over `async` for theme scripts because `defer` preserves execution order (scripts execute in document order after HTML parsing completes) while `async` does not guarantee order and can cause dependency failures when one script depends on another. Scripts that genuinely cannot be deferred — those that must execute synchronously before the DOM exists, such as a "no-flash" theme color script — are the rare exception and must be documented as such.

---

## Detection Logic (AST)

1. **Identify `LiquidFilter` nodes with `name === "script_tag"`.** The AST walker traverses all `LiquidFilter` nodes in the template tree. Any filter node where `name === "script_tag"` is a candidate violation — the filter cannot produce `defer` or `async` output regardless of arguments.

2. **Confirm the filter is in a `LiquidVariable` output expression.** `script_tag` is an output filter, appearing in `{{ 'file.js' | asset_url | script_tag }}` patterns. Verify the filter chain is inside a `LiquidVariable` node (not a `LiquidTag[assign]` context), confirming that the output is rendered directly into the HTML stream.

3. **Classify by location in the document.** Determine whether the `LiquidVariable` node's ancestor chain includes an `HtmlElement[head]` node (script in `<head>` — blocks parser immediately on encounter) or an `HtmlElement[body]` node (script in `<body>` — blocks parser at the point of encounter, still harmful but impact depends on position). Report the location as context in the violation message.

4. **Identify the file being loaded.** Extract the first filter in the chain — typically `asset_url` — and trace its input to identify the filename. This enables the fix suggestion to name the specific file: `<script defer src="{{ 'file.js' | asset_url }}"></script>`.

**Why regex fails here:** A pattern matching `script_tag` as a filter name would match commented-out code inside `{% comment %}` blocks, match filter names in `{% assign %}` statements that never produce output, and cannot determine whether the matched filter output lands in the `<head>` or `<body>`. The AST walker differentiates `LiquidVariable` output nodes from assignment nodes, skips nodes inside `LiquidTag[comment]` ancestors, and can traverse parent elements to locate the `<head>` or `<body>` context through `HtmlElement` ancestor inspection.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: script_tag filter used to load JavaScript assets in theme.liquid.
  This is the default pattern in many legacy Shopify themes and starter kits.

  Failure mode 1: All five script_tag calls are parser-blocking. The browser
  halts HTML parsing at each one. Total blocking time: 800ms–1.5s on 4G
  before any body content is painted.

  Failure mode 2: script_tag generates <script src="..."></script> with no
  defer or async attribute. There is no filter argument to change this.
  The filter's output format is hardcoded in Shopify's Liquid runtime.

  Failure mode 3: Theme Store rejection — the automated review pipeline
  checks for parser-blocking scripts. Multiple script_tag calls in <head>
  result in a performance violation that blocks Theme Store submission.

  Failure mode 4: Execution order is accidental — scripts happen to load in
  source order only because they're all blocking. Switching to async would
  break dependencies. The fix requires deliberate defer sequencing.
{%- endcomment -%}

<!DOCTYPE html>
<html lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ page_title }}</title>

    {{ content_for_header }}

    {{ 'theme.css' | asset_url | stylesheet_tag }}

    {{ 'vendor/jquery.min.js' | asset_url | script_tag }}
    {{ 'vendor/splide.min.js' | asset_url | script_tag }}
    {{ 'vendor/fancybox.min.js' | asset_url | script_tag }}
    {{ 'theme.js' | asset_url | script_tag }}
    {{ 'cart.js' | asset_url | script_tag }}
  </head>

  <body class="template-{{ template.name }}">
    <a class="skip-link" href="#MainContent">
      {{ 'accessibility.skip_to_content' | t }}
    </a>

    {% sections 'header-group' %}

    <main id="MainContent" role="main" tabindex="-1">
      {{ content_for_layout }}
    </main>

    {% sections 'footer-group' %}
  </body>
</html>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace all script_tag filter calls with manual <script defer src="...">
  construction. The defer attribute is the critical change — it instructs
  the browser to fetch the script without blocking the parser and to execute
  it only after HTML parsing completes, in document order.

  Change 1: <script defer src="{{ 'file.js' | asset_url }}"> replaces the
  script_tag filter entirely. defer preserves execution order — all deferred
  scripts execute in the order they appear in the document.

  Change 2: jQuery is removed entirely (see PERF-010). All APIs used by
  the theme scripts have native equivalents supported since 2019.

  Change 3: Vendor libraries (splide, fancybox) are also deferred. They
  must be listed before theme.js in document order because deferred scripts
  execute in order — theme.js depending on splide will find it initialized.

  Change 4: type="module" is an alternative to defer for ES module scripts —
  modules are deferred by default. For non-module scripts, explicit defer
  is required.
{%- endcomment -%}

<!DOCTYPE html>
<html lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ page_title }}</title>

    {{ content_for_header }}

    {{ 'theme.css' | asset_url | stylesheet_tag }}

    {%- comment -%}
      defer: scripts are fetched in parallel (non-blocking) and execute
      in document order after the HTML parser finishes. No parser blocking.
    {%- endcomment -%}
    <script defer src="{{ 'vendor/splide.min.js' | asset_url }}"></script>
    <script defer src="{{ 'vendor/fancybox.min.js' | asset_url }}"></script>
    <script defer src="{{ 'theme.js' | asset_url }}"></script>
    <script defer src="{{ 'cart.js' | asset_url }}"></script>
  </head>

  <body class="template-{{ template.name }}">
    <a class="skip-link" href="#MainContent">
      {{ 'accessibility.skip_to_content' | t }}
    </a>

    {% sections 'header-group' %}

    <main id="MainContent" role="main" tabindex="-1">
      {{ content_for_layout }}
    </main>

    {% sections 'footer-group' %}
  </body>
</html>
```

---

## References

- [Shopify — `script_tag` filter (legacy documentation)](https://shopify.dev/docs/api/liquid/filters/script_tag)
- [Shopify — Theme Store requirements: script loading](https://shopify.dev/docs/storefronts/themes/store/requirements)
- [Shopify — Theme performance: JavaScript](https://shopify.dev/docs/storefronts/themes/best-practices/performance#javascript)
- [MDN — `<script>`: defer attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#defer)
- [web.dev — Efficiently load third-party JavaScript](https://web.dev/articles/efficiently-load-third-party-javascript)
- [Google — Core Web Vitals: Total Blocking Time](https://web.dev/articles/tbt)
