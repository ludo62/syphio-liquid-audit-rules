# LIQ-008 | `{{ content_for_header }}` missing or duplicated in layout

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** `layout/theme.liquid` and any custom layout files in `/layout/`

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{{ content_for_header }}` is a Shopify magic variable that serves as the injection point for all platform-level JavaScript that Shopify's infrastructure must deliver to every storefront page. Its content is generated server-side by Shopify's rendering pipeline and includes: the Shopify Analytics pixel initialization, the App Bridge JavaScript required by all embedded apps, ScriptTag payloads registered by installed apps via the Admin API, the Theme Editor preview bar JavaScript (enabling live editing without a full reload), checkout session token scripts, and the Shopify global object (`window.Shopify`) initialization that every Shopify app relies on for storefront context. This content is not accessible via any other Liquid variable and cannot be manually reconstructed.

When `{{ content_for_header }}` is absent from the layout, every installed Shopify app that uses ScriptTag or App Embeds fails silently. The cart AJAX API may not initialize correctly, causing add-to-cart interactions to fail. Discount apps that intercept price rendering will not load. Tracking pixels (Facebook CAPI, Google GA4) will not fire. Shopify's own analytics will stop recording sessions. The Theme Editor's live preview bar will not render, breaking the merchant's ability to edit the theme via the Customize interface. None of these failures produce a visible error to the visitor — the page renders, but all app-dependent functionality is broken. The merchant will typically discover the issue only after reporting from customers or noticing a sudden drop in analytics data.

When `{{ content_for_header }}` is duplicated — a common artifact of merging two theme versions or copying a layout file from a different theme — every platform-level script is injected twice. Event listeners registered by app scripts fire twice per event. Cart mutation handlers double-trigger, potentially submitting two API requests for one user action. Analytics events (add-to-cart, page view, purchase) are double-counted in every connected analytics platform. The Shopify global object initialization runs twice, which in some SDK versions produces console errors or overrides previously set properties. `{{ content_for_header }}` must appear exactly once in the `<head>` element, before any theme-defined JavaScript.

---

## Detection Logic (AST)

1. **Scope detection to layout files.** The rule applies only to files in the `/layout/` directory. Template files (`/templates/`) and section files (`/sections/`) do not require `{{ content_for_header }}` directly. The walker identifies the file path and skips non-layout files.

2. **Identify `LiquidVariableOutput` nodes where the expression resolves to `content_for_header`.** Within layout files, the walker traverses all `LiquidVariableOutput` nodes and inspects the `expression` property. A `LiquidVariableOutput` where `expression.name === "content_for_header"` (a bare variable reference with no filters) is the target node. Filter presence on `content_for_header` is itself a secondary violation — the variable must not be filtered.

3. **Count occurrences per file.** The walker accumulates all matching `LiquidVariableOutput[content_for_header]` nodes into a count. If `count === 0`, the rule fires as MISSING (CRITICAL severity). If `count > 1`, the rule fires as DUPLICATED (HIGH severity) with source positions of all occurrences reported.

4. **Verify placement within `<head>` element.** For layouts where `count === 1` (correct count), the walker verifies that the `LiquidVariableOutput[content_for_header]` node's parent chain includes an `HtmlElement[tag=head]` node. If it appears outside `<head>` (in `<body>` or at the root), a placement violation is flagged — scripts loaded in `<body>` rather than `<head>` can cause race conditions with app initialization code.

**Why regex fails here:** A regex matching `\{\{\s*content_for_header\s*\}\}` and counting occurrences would report false positives on occurrences inside `{% comment %}` blocks or inside `{% if %}` branches where the variable is conditionally rendered (a pattern that itself indicates a logic error). The AST walker inspects the node's parent chain to exclude `LiquidTag[comment]` descendants from the count, ensuring only live output occurrences are counted.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Two scenarios shown — one missing, one duplicated.
  Both scenarios produce distinct failure modes.

  Scenario A (missing): content_for_header omitted entirely.
  - All app ScriptTags fail to load.
  - Cart AJAX API does not initialize.
  - Theme Editor preview bar is absent — merchant cannot use Customize.
  - Analytics pixels (GA4, Facebook CAPI, Shopify Analytics) do not fire.
  - Discount apps, loyalty apps, review apps all break silently.

  Scenario B (duplicated): content_for_header appears twice.
  - All platform scripts execute twice per page load.
  - Cart event listeners double-fire — potential double add-to-cart API calls.
  - Analytics events double-count — conversion rate reporting corrupted.
  - window.Shopify object initialized twice — SDK conflicts in some app versions.
{%- endcomment -%}

{%- comment -%} Scenario A: content_for_header missing entirely -%}
<!DOCTYPE html>
<html lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    {%- comment -%} content_for_header absent — all app scripts fail silently -%}
    <title>{{ page_title }}</title>
    {{ 'theme.css' | asset_url | stylesheet_tag }}
  </head>
  <body>
    {% section 'header' %}
    <main>{{ content_for_layout }}</main>
    {% section 'footer' %}
    {{ 'theme.js' | asset_url | script_tag }}
  </body>
</html>

{%- comment -%} Scenario B: content_for_header duplicated -%}
<!DOCTYPE html>
<html lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    {{ content_for_header }}
    <title>{{ page_title }}</title>
    {%- comment -%} Second occurrence — from a copy-paste or merge error -%}
    {{ content_for_header }}
    {{ 'theme.css' | asset_url | stylesheet_tag }}
  </head>
  <body>
    {% section 'header' %}
    <main>{{ content_for_layout }}</main>
    {% section 'footer' %}
  </body>
</html>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: content_for_header appears exactly once, inside <head>,
  as the first meaningful tag after <meta charset>.

  Placement rationale: Shopify's platform scripts must execute before any
  theme JavaScript. Placing content_for_header before stylesheet_tag and
  before any custom <script> tags ensures app initialization completes
  before theme code runs. This prevents race conditions in app bridge setup.

  The <meta charset> must remain first in <head> per HTML spec (within first
  1024 bytes). content_for_header comes immediately after.
{%- endcomment -%}

<!DOCTYPE html>
<html lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    {%- comment -%}
      content_for_header: exactly one occurrence, in <head>, before theme assets.
      Injects: Analytics, App Bridge, ScriptTags, Theme Editor bar, window.Shopify.
    {%- endcomment -%}
    {{ content_for_header }}

    <title>{{ page_title }}</title>
    <meta name="description" content="{{ page_description | escape }}">

    {{ 'theme.css' | asset_url | stylesheet_tag }}

    {%- comment -%}
      Preconnect to Shopify CDN for image and asset performance.
      Placed after content_for_header so app-injected preconnects don't conflict.
    {%- endcomment -%}
    <link rel="preconnect" href="https://cdn.shopify.com" crossorigin>
  </head>
  <body>
    {% section 'announcement-bar' %}
    {% section 'header' %}

    <main id="main-content" role="main" tabindex="-1">
      {{ content_for_layout }}
    </main>

    {% section 'footer' %}

    {{ 'theme.js' | asset_url | script_tag }}
  </body>
</html>
```

---

## References

- [Shopify — `content_for_header` global variable](https://shopify.dev/docs/api/liquid/objects/content_for_header)
- [Shopify — Layout files](https://shopify.dev/docs/storefronts/themes/architecture/layouts)
- [Shopify — App embeds and ScriptTag API](https://shopify.dev/docs/apps/build/online-store/script-tag-legacy)
- [Shopify Theme Check — RequiredLayoutThemeObject rule](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/required-layout-theme-object)
