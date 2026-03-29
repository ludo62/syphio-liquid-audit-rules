# PERF-009 | Inline `<script>` without `defer`/`async` on non-critical path

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** `theme.liquid`, section files, snippet files — any context where inline `<script>` blocks appear in `<head>` or early `<body>` outside structured data or critical initialization

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Inline `<script>` blocks — JavaScript written directly between `<script>` and `</script>` tags without a `src` attribute — are unconditionally parser-blocking. Unlike external scripts, inline scripts cannot use the `defer` or `async` attributes; the HTML specification explicitly states that these attributes have no effect on scripts without a `src`. When the HTML parser encounters an inline `<script>` block, it must stop parsing, hand control to the JavaScript engine, wait for the script to execute fully, and only then resume parsing the remaining HTML. There is no mechanism to defer this execution. The only way to make inline script execution non-blocking is to restructure the code: move it into an external file that can be loaded with `defer`, or restructure the logic to run inside a `DOMContentLoaded` event listener placed just before `</body>`.

In Shopify themes, inline `<script>` blocks in the `<head>` most commonly serve three purposes: passing Liquid data to JavaScript (product variant JSON, cart data, section settings), loading analytics/tracking initialization code, and performing DOM manipulation such as applying a theme color or font class before first paint. Of these, only the last category — scripts that must run before the first paint to prevent a flash of unstyled content — has a valid reason to be in the `<head>` as a blocking inline script. Data serialization (passing Liquid data to JavaScript) should be done via `<script type="application/json">` tags, which are not executed by the JavaScript engine and are non-blocking. Analytics initialization code should always be deferred until after the DOM is interactive — tracking missed in the first 100ms of page load is negligible compared to the TBT impact of blocking initialization.

Shopify's Theme Store review process flags inline `<script>` blocks in `<head>` that contain DOM manipulation (the DOM is not available in `<head>`) and analytics/tracking initialization (always deferrable). Shopify's own Liquid rendering pipeline provides the `content_for_header` global, which Shopify uses to inject its own scripts in a controlled, non-blocking manner. Theme code should not attempt to replicate this mechanism with its own `<head>` inline scripts. Any inline `<script>` not of `type="application/ld+json"` (structured data) or `type="application/json"` (data serialization) warrants inspection to determine whether it can be moved to just before `</body>` or refactored into a deferred external file.

---

## Detection Logic (AST)

1. **Identify `HtmlRawElement[script]` nodes.** The AST walker collects all `HtmlRawElement` nodes where `name === "script"`. `HtmlRawElement` is the correct AST node type for `<script>...</script>` elements, as script content is treated as raw markup (not parsed as HTML or Liquid). The `RawMarkup` child node contains the script's content.

2. **Filter to inline scripts: check for absence of `src` attribute.** For each `HtmlRawElement[script]` node, inspect its `HtmlAttribute` children. If no attribute with `name === "src"` is present, the script is inline. External scripts (`src` present) have their own rule pathway (PERF-008 for `script_tag`, PERF-008 for manual `<script>` without `defer`).

3. **Filter out exempt `type` values.** Check `HtmlAttribute` children for `name === "type"`. If `value === "application/ld+json"` or `value === "application/json"`, the element is not executed as JavaScript — exclude from the violation. `type="module"` scripts are deferred by default and may be flagged with a lower severity (informational) depending on their content.

4. **Classify by document position.** Determine whether the `HtmlRawElement[script]` node's ancestor chain contains an `HtmlElement[head]` (in `<head>` — CRITICAL impact, parser-blocked before any body content) or `HtmlElement[body]` (in `<body>` — HIGH impact, parser-blocked at the point of encounter). Inspect the `RawMarkup` child's text content for patterns indicating DOM manipulation (`document.querySelector`, `getElementById`, `document.body`) — in `<head>`, DOM manipulation will fail silently or throw an error since the DOM is not yet built.

**Why regex fails here:** A regex matching `<script>` without a `src` attribute must distinguish `<script type="application/ld+json">` (valid, non-blocking) from `<script>` (JavaScript, blocking). The `type` attribute may appear before or after other attributes, on the same line or a different line, with or without quotes. Multi-line inline script tags in Liquid templates also interleave with Liquid `{{ }}` and `{% %}` expressions within attribute values, making attribute parsing by regex fragile. The AST represents `type` as a discrete `HtmlAttribute` node — presence and value checking is a direct property access, not a pattern match.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Multiple inline <script> blocks in <head> performing
  non-critical initialization that should be deferred.
  Context: theme.liquid head section of a Shopify theme.

  Failure mode 1: Analytics script blocks the parser on every page load.
  The tracking call is not time-sensitive — a 100ms delay in tracking
  startup is imperceptible to analytics accuracy but visible to LCP.

  Failure mode 2: Cart data serialization uses an executable <script> block
  rather than <script type="application/json">. The JS engine parses and
  compiles the script even though it only defines a variable.

  Failure mode 3: DOM manipulation in the third script will fail or produce
  errors — document.getElementById fires before the body exists.

  Failure mode 4: Cumulative blocking time for three inline scripts,
  even if each executes in 5ms, is still 15ms+ of parser-blocking time
  added on top of any external scripts in the <head>.
{%- endcomment -%}

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>

  {{ content_for_header }}

  <script>
    window.analyticsConfig = {
      storeId: '{{ shop.id }}',
      currency: '{{ cart.currency.iso_code }}',
      locale: '{{ request.locale.iso_code }}'
    };
    (function() {
      var script = document.createElement('script');
      script.src = 'https://analytics.example.com/track.js';
      document.head.appendChild(script);
    })();
  </script>

  <script>
    window.cartData = {
      token: '{{ cart.token }}',
      itemCount: {{ cart.item_count }},
      totalPrice: {{ cart.total_price }}
    };
  </script>

  <script>
    var themeColor = '{{ settings.color_primary }}';
    document.getElementById('theme-root').style.setProperty('--color-primary', themeColor);
  </script>

  {{ 'theme.css' | asset_url | stylesheet_tag }}
</head>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Restructure inline scripts to eliminate parser blocking.

  Change 1: Cart data serialized via <script type="application/json">.
  The browser does not execute application/json script blocks — they are
  non-blocking. JavaScript can read the data via JSON.parse(el.textContent).

  Change 2: Analytics initialization moved to a deferred external script
  (analytics-init.js). The external file uses defer, so it executes after
  HTML parsing completes. The 100ms delay is imperceptible to analytics.

  Change 3: The CSS custom property initialization (color token) is moved
  to a <style> block using Liquid interpolation — no JavaScript required.
  For themes that genuinely need JS-based pre-paint initialization (e.g.,
  dark mode flash prevention), the script is kept but isolated to that
  single purpose with a comment explaining the exception.

  Change 4: window.themeConfig data object is placed just before </body>
  so it is available to deferred scripts without blocking the parser.
{%- endcomment -%}

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>

  {{ content_for_header }}

  {%- comment -%}
    CSS custom property set via <style> block — no JS required, no blocking.
  {%- endcomment -%}
  <style>
    :root {
      --color-primary: {{ settings.color_primary }};
      --color-secondary: {{ settings.color_secondary }};
    }
  </style>

  {%- comment -%}
    Non-blocking JSON data island — parsed by consuming scripts after load.
    type="application/json" is never executed by the JS engine.
  {%- endcomment -%}
  <script id="cart-data" type="application/json">
    {
      "token": {{ cart.token | json }},
      "itemCount": {{ cart.item_count }},
      "totalPrice": {{ cart.total_price }}
    }
  </script>

  {{ 'theme.css' | asset_url | stylesheet_tag }}

  {%- comment -%}
    Analytics loaded as deferred external script. Executes after HTML parse
    completes. analyticsConfig is available from the data island above.
  {%- endcomment -%}
  <script defer src="{{ 'analytics-init.js' | asset_url }}"></script>
  <script defer src="{{ 'theme.js' | asset_url }}"></script>
</head>

<body class="template-{{ template.name }}">
  {% sections 'header-group' %}

  <main id="MainContent" role="main" tabindex="-1">
    {{ content_for_layout }}
  </main>

  {% sections 'footer-group' %}

  {%- comment -%}
    Page-level config placed just before </body> — available to all deferred
    scripts without blocking the parser during head processing.
  {%- endcomment -%}
  <script>
    window.themeConfig = {
      storeId: {{ shop.id | json }},
      currency: {{ cart.currency.iso_code | json }},
      locale: {{ request.locale.iso_code | json }},
      routes: {
        cart: '{{ routes.cart_url }}',
        cartAdd: '{{ routes.cart_add_url }}'
      }
    };
  </script>
</body>
```

---

## References

- [Shopify — Theme performance: JavaScript best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance#javascript)
- [Shopify — Theme Store requirements](https://shopify.dev/docs/storefronts/themes/store/requirements)
- [MDN — `<script>`: The inline script does not support defer](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#defer)
- [web.dev — Render-blocking resources](https://web.dev/articles/render-blocking-resources)
- [web.dev — Total Blocking Time](https://web.dev/articles/tbt)
- [HTML spec — Parser-blocking scripts](https://html.spec.whatwg.org/multipage/scripting.html#parser-blocking)
