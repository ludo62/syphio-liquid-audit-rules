# PERF-010 | jQuery loaded as a global dependency

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** `theme.liquid` and any template loading jQuery via `script_tag` or `<script src>` — impacts all pages globally

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

jQuery 3.7 minified and gzipped is 87KB. Loaded synchronously in the `<head>` — as the overwhelming majority of jQuery-dependent themes load it, because theme JavaScript that uses `$()` must wait for jQuery to be available — this represents 87KB of parser-blocking JavaScript that must be fully downloaded, parsed, compiled, and executed before the HTML parser can continue. On a mid-range Android device (the median device class globally for Shopify merchant customers), JavaScript parse and compile time for 87KB of code is 10–30ms. Combined with the network fetch time on a 4G connection, the total time cost is 200–400ms of Total Blocking Time attributable solely to jQuery. This single dependency is frequently the largest contributor to TBT on jQuery-dependent themes.

Every API that jQuery provided is now natively available in all modern browsers without polyfills. `$(selector)` is `document.querySelector(selector)` or `document.querySelectorAll(selector)`. `$.ajax()` is `fetch()`. `$(el).addClass()` / `$(el).removeClass()` is `el.classList.add()` / `el.classList.remove()`. `$(el).on('event', handler)` is `el.addEventListener('event', handler)`. `$(document).ready(fn)` is `document.addEventListener('DOMContentLoaded', fn)` or simply placing scripts before `</body>`. `$.extend()` is `Object.assign()` or the spread operator. The browser compatibility floor for all of these APIs is Internet Explorer 11 (querySelector, classList, fetch with a polyfill) or fully evergreen browsers (classList, fetch natively). As of 2024, IE11 represents less than 0.5% of global web traffic, and Shopify's own theme SDK dropped IE11 support in 2022. There is no remaining browser compatibility argument for jQuery in a Shopify theme context.

Shopify's reference theme ecosystem has been jQuery-free since Dawn 1.0 (2021). Craft (2023) and Horizon (2024) also ship zero jQuery. The Shopify Theme Store increasingly rejects themes that load jQuery globally on every page, citing the 87KB parsing overhead and the availability of native equivalents. In Theme Store review feedback published in the Shopify Partners community, jQuery global loading is cited as the most frequent performance feedback item for High severity themes. Themes that use jQuery in isolated sections can confine the dependency to those sections by dynamically importing a module-scoped jQuery import — but the correct long-term approach is migration to native APIs, which is a one-time refactor with no ongoing maintenance burden.

---

## Detection Logic (AST)

1. **Identify `HtmlRawElement[script]` nodes with a `src` attribute pointing to jQuery.** The AST walker collects all `HtmlRawElement` nodes where `name === "script"`. For each, inspect `HtmlAttribute` children for `name === "src"`. If the `src` value (or its `LiquidVariable` component after filter evaluation) contains `"jquery"`, `"jQuery"`, or `"jQuery.min"` (case-insensitive), this is a jQuery load candidate.

2. **Identify `LiquidVariable` nodes with `script_tag` filter chained after `asset_url` on a jQuery filename.** For `LiquidVariable` nodes containing a `LiquidFilter` chain, check whether the input string literal (the asset filename) contains `"jquery"` (case-insensitive) and the chain includes an `asset_url` filter followed by `script_tag`. This catches both the `asset_url | script_tag` form and manual `<script src>` forms.

3. **Check for global scope: confirm the load is in `theme.liquid` or equivalent global layout.** Determine whether the jQuery load site is in a global layout template (`theme.liquid`, `password.liquid`) or in a section/snippet. Global layouts load on every page — this is the highest impact form. Section-scoped jQuery loads (inside a `<script>` in a section file) are a lower severity variant but still violate the rule.

4. **Detect `$` and `jQuery` usage in inline scripts as a dependency signal.** In `HtmlRawElement[script]` nodes with no `src` attribute (inline scripts), inspect the `RawMarkup` child's text for references to `$(` or `jQuery(` or `$.ajax` patterns. These confirm jQuery dependency and should be reported alongside the jQuery load violation as context for the migration scope.

**Why regex fails here:** A regex matching `jquery` in script `src` attributes would match commented-out script tags, strings inside JavaScript template literals that reference "jquery" as a string (not a load), and documentation strings. More importantly, it cannot determine whether the jQuery load is in a global layout (high severity — every page affected) versus in a section-scoped script (lower severity — only that section affected). The AST walker resolves the template file's type from the parse context and can classify scope at the structural level, not just the textual level.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: jQuery loaded globally in theme.liquid as a synchronous
  parser-blocking script, used throughout the theme.
  Context: A theme migrated from a pre-OS2.0 codebase that relied on
  jQuery for all DOM interactions.

  Failure mode 1: 87KB parser-blocking load on every page — product pages,
  collection pages, cart, checkout, blog, article. The TBT penalty is paid
  universally, even on pages where jQuery is never called.

  Failure mode 2: theme.js depends on jQuery being present — it accesses
  $ immediately at module scope. Both scripts are synchronous, so load order
  is guaranteed, but the combined blocking cost is 87KB + theme.js size.

  Failure mode 3: jQuery's $ conflicts with other global libraries if
  third-party apps also inject jQuery (a common occurrence with Shopify
  apps). Version conflicts cause silent failures or require $.noConflict().

  Failure mode 4: Theme Store rejection — global jQuery loading is flagged
  in the automated performance review pipeline as a HIGH severity violation.
{%- endcomment -%}

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>
  {{ content_for_header }}
  {{ 'theme.css' | asset_url | stylesheet_tag }}
  {{ 'jquery.min.js' | asset_url | script_tag }}
  {{ 'theme.js' | asset_url | script_tag }}
</head>

<body>
  {% sections 'header-group' %}
  <main id="MainContent">{{ content_for_layout }}</main>
  {% sections 'footer-group' %}
</body>

{%- comment -%}
  Example of jQuery usage in a section script — all of this has native equivalents.
{%- endcomment -%}
<script>
  $(document).ready(function() {
    // Product form submission
    $('#product-form').on('submit', function(e) {
      e.preventDefault();
      var formData = $(this).serialize();
      $.ajax({
        type: 'POST',
        url: '/cart/add.js',
        data: formData,
        success: function(response) {
          $('.cart-count').text(response.quantity);
          $('.cart-notification').addClass('is-visible');
        }
      });
    });

    // Accordion toggle
    $('.accordion__trigger').on('click', function() {
      $(this).toggleClass('is-active');
      $(this).next('.accordion__content').slideToggle(200);
    });
  });
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Remove jQuery entirely. Replace all jQuery APIs with native equivalents.
  All APIs used above have had native browser support since 2019.

  Change 1: jQuery removed from theme.liquid — no 87KB blocking load.
  theme.js migrated to native APIs and loaded with defer.

  Change 2: $(document).ready() → DOMContentLoaded event listener, or
  simply rely on defer attribute which guarantees DOM is ready on execution.

  Change 3: $.ajax() → fetch() with async/await. Cleaner syntax, native,
  zero additional bytes.

  Change 4: $().on() → addEventListener(). $().toggleClass() → classList.toggle().
  $().next() → nextElementSibling. All native, all supported universally.

  Change 5: CSS transitions replace $.slideToggle() — the animation is
  GPU-accelerated via CSS and requires zero JavaScript bytes.
{%- endcomment -%}

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>
  {{ content_for_header }}
  {{ 'theme.css' | asset_url | stylesheet_tag }}
  {%- comment -%}No jQuery. defer on theme.js: DOM is ready when it executes.{%- endcomment -%}
  <script defer src="{{ 'theme.js' | asset_url }}"></script>
</head>

<body>
  {% sections 'header-group' %}
  <main id="MainContent">{{ content_for_layout }}</main>
  {% sections 'footer-group' %}
</body>

{%- comment -%}
  Native equivalents — no jQuery required. This script block can be moved
  into theme.js (external, deferred) for best practice.
{%- endcomment -%}
<script>
  // DOMContentLoaded is guaranteed by defer — this listener is for inline scripts only.
  document.addEventListener('DOMContentLoaded', function () {

    // Product form — fetch() replaces $.ajax()
    var productForm = document.getElementById('product-form');
    if (productForm) {
      productForm.addEventListener('submit', async function (event) {
        event.preventDefault();
        var formData = new FormData(productForm);

        try {
          var response = await fetch('/cart/add.js', {
            method: 'POST',
            body: formData,
            headers: { 'X-Requested-With': 'XMLHttpRequest' }
          });
          var cartItem = await response.json();

          // classList.replace / textContent — replaces $().text() and $().addClass()
          var cartCount = document.querySelector('.cart-count');
          if (cartCount) cartCount.textContent = cartItem.quantity;

          var notification = document.querySelector('.cart-notification');
          if (notification) notification.classList.add('is-visible');
        } catch (error) {
          console.error('Cart add failed:', error);
        }
      });
    }

    // Accordion — classList.toggle + CSS transition replaces $.slideToggle()
    document.querySelectorAll('.accordion__trigger').forEach(function (trigger) {
      trigger.addEventListener('click', function () {
        trigger.classList.toggle('is-active');
        var content = trigger.nextElementSibling; // replaces $().next()
        if (content) {
          content.classList.toggle('is-expanded'); // CSS handles the animation
        }
      });
    });
  });
</script>
```

---

## References

- [Shopify — Dawn theme: zero jQuery](https://github.com/Shopify/dawn)
- [Shopify — Theme performance: remove jQuery](https://shopify.dev/docs/storefronts/themes/best-practices/performance#javascript)
- [You Might Not Need jQuery](https://youmightnotneedjquery.com/)
- [MDN — Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [MDN — Element.classList](https://developer.mozilla.org/en-US/docs/Web/API/Element/classList)
- [web.dev — Total Blocking Time](https://web.dev/articles/tbt)
- [Shopify — Theme Store requirements: performance](https://shopify.dev/docs/storefronts/themes/store/requirements#performance)
