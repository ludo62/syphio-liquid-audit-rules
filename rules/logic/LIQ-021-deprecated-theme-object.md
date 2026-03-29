# LIQ-021 | Deprecated Liquid `theme` object used

**Category:** Merchant Logic

**Severity:** 🟠 HIGH

**Affects:** All templates, layouts, and snippets referencing the `theme` Liquid object; particularly `theme.id`, `theme.name`, `theme.role`

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

The `theme` Liquid object was introduced in Shopify's first-generation theme architecture to expose properties about the currently rendered theme: `theme.id` (an integer), `theme.name` (a string), and `theme.role` (one of `main`, `unpublished`, `demo`). These properties were primarily used for two purposes: conditional logic to detect whether a theme was in "live" versus "preview" mode, and generating absolute URLs to theme assets by interpolating `theme.id` into Shopify's CDN URL pattern. Both use cases are brittle with the modern Shopify infrastructure. `theme.id` is not guaranteed to be stable across theme duplication, and `theme.role` returns values that are unreliable in the context of Online Store 2.0's preview and editor modes, where the same theme can run in multiple simultaneous roles.

As of the Online Store 2.0 architecture, Shopify officially deprecated the `theme` object and flagged it in Theme Check as a deprecation violation. In modern Shopify's rendering pipeline, `theme.id` returns an unpredictable value depending on whether the request originates from a published theme render, a Theme Editor preview iframe, a development theme, or a password-protected storefront preview link. Conditional logic like `{% if theme.role == 'main' %}` to detect live versus preview mode is functionally broken in these contexts — the `role` property does not reliably distinguish editor sessions from live publishes. Themes in the Theme Store submission pipeline are automatically rejected if Theme Check reports `theme` object usage.

The replacements are context-specific. For detecting whether the storefront is being viewed inside the Theme Editor (the most common use case of `theme.role`), the correct object is `request.design_mode` — a boolean that is `true` only when the page renders inside the Theme Editor iframe. For store-level information such as store name, currency, or timezone, the `shop` object provides stable, accurate data. For theme-specific configuration that was previously toggled by `theme.name`, the correct approach is to use `settings`-based feature flags defined in `settings_schema.json`. Asset URLs should use `| asset_url` against a known filename in `/assets/`, not constructed from `theme.id`.

---

## Detection Logic (AST)

1. **Identify `LiquidVariable` and `VariableLookup` nodes with root identifier `theme`.** Walk the AST for all `LiquidVariable` nodes and `LiquidTag` (assign, if, unless, case) condition expressions. Extract all `VariableLookup` nodes whose root identifier is the string `theme`.
2. **Verify the lookup is against the Liquid `theme` object, not a user-defined variable.** Check whether `theme` is defined as a local variable via a prior `assign` or `capture` tag in the current scope chain. If so, skip — the reference is to a user-defined variable, not the deprecated Shopify object. If no local definition exists, it resolves to the Shopify global `theme` object.
3. **Inspect the property access.** Record which property is accessed: `theme.id`, `theme.name`, `theme.role`, or bare `theme` (used as a truthy object). Each property has a distinct replacement recommendation.
4. **Classify by usage pattern.** `theme.role` used in a conditional: HIGH — direct behavioral dependency on deprecated data. `theme.id` used in URL construction: HIGH — CDN URL construction via deprecated mechanism. `theme.name` used in a conditional: MEDIUM — string comparison against theme name is fragile regardless of deprecation.
5. **Report the recommended replacement in the diagnostic.** `theme.role` → `request.design_mode`; `theme.id` → `| asset_url` filter; `theme.name` → `settings`-based flag.

**Why regex fails here:** A regex matching `theme\.` would flag JavaScript expressions like `document.theme.color` or user-defined Liquid variables named `theme_settings` that happen to contain the substring `theme.`. Distinguishing the Shopify global `theme` object from user-defined variables named `theme` requires AST-level scope resolution: checking whether the root identifier `theme` was previously defined via `assign` or `capture` in the current scope chain, which is a symbol-table operation unavailable to pattern matching.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: The theme object is used for live/preview detection, asset URL
  construction, and theme-name-based feature flags. All three usages are deprecated.
  theme.role is unreliable in Theme Editor, preview, and development contexts.
  theme.id changes after theme duplication and is not stable for CDN URL building.
  theme.name comparisons are brittle — name changes break conditional logic silently.
{%- endcomment -%}

{%- comment -%} Deprecated: live vs preview detection via theme.role {%- endcomment -%}
{%- if theme.role == 'main' -%}
  {%- assign analytics_enabled = true -%}
{%- else -%}
  {%- assign analytics_enabled = false -%}
{%- endif -%}

{%- comment -%} Deprecated: asset URL via theme.id interpolation {%- endcomment -%}
{%- assign custom_css_url = 'https://cdn.shopify.com/s/files/1/0000/0000/t/' | append: theme.id | append: '/assets/custom.css' -%}
<link rel="stylesheet" href="{{ custom_css_url }}">

{%- comment -%} Deprecated: feature toggle via theme.name string comparison {%- endcomment -%}
{%- if theme.name == 'Meridian 2.0' -%}
  {%- render 'meridian-hero' -%}
{%- endif -%}

<p>Theme: {{ theme.name }} (ID: {{ theme.id }}, Role: {{ theme.role }})</p>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace all theme object usages with their supported equivalents.
  request.design_mode is the correct boolean for Theme Editor detection.
  | asset_url generates stable CDN URLs from filenames in /assets/ — no theme.id
  needed. Feature flags belong in settings_schema.json and are accessed via the
  settings object. Displaying theme metadata in output is unnecessary in production;
  if needed for debugging, scope it to design_mode sessions only.
{%- endcomment -%}

{%- comment -%} Supported: Theme Editor detection via request.design_mode {%- endcomment -%}
{%- if request.design_mode -%}
  {%- assign analytics_enabled = false -%}
{%- else -%}
  {%- assign analytics_enabled = true -%}
{%- endif -%}

{%- comment -%} Supported: asset URL via | asset_url filter {%- endcomment -%}
<link rel="stylesheet" href="{{ 'custom.css' | asset_url }}">

{%- comment -%} Supported: feature toggle via settings flag defined in settings_schema.json {%- endcomment -%}
{%- if settings.enable_hero_block -%}
  {%- render 'meridian-hero' -%}
{%- endif -%}

{%- comment -%} Debug output scoped to Theme Editor only — not emitted in production {%- endcomment -%}
{%- if request.design_mode -%}
  <!-- Design mode active: {{ shop.name }} -->
{%- endif -%}
```

---

## References

- [Shopify Liquid: theme object — deprecation notice](https://shopify.dev/docs/api/liquid/objects/theme)
- [Shopify Liquid: request object — design_mode](https://shopify.dev/docs/api/liquid/objects/request#request-design_mode)
- [Shopify Liquid: asset_url filter](https://shopify.dev/docs/api/liquid/filters/asset_url)
- [Shopify Theme Check: DeprecatedLiquidObject](https://shopify.dev/docs/themes/tools/theme-check/checks/deprecated-liquid-object)
- [Shopify: Online Store 2.0 architecture](https://shopify.dev/docs/themes/architecture)
