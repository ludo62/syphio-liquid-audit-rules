# LIQ-022 | `.css.liquid` or `.js.liquid` file — deprecated since 2025

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** Theme file structure; any theme containing files with `.css.liquid` or `.js.liquid` extensions in the `/assets/` directory

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

The `.css.liquid` and `.js.liquid` file formats were a Shopify 1.0 mechanism that allowed developers to inject Liquid expressions — primarily theme settings values such as colors, font sizes, and spacing — directly into CSS and JavaScript files. At request time, Shopify's server would process these files through the Liquid renderer, resolve `{{ settings.color_primary }}` expressions, and serve the resulting plain CSS or JavaScript to the browser. This mechanism was architecturally simple but had significant performance costs: the files could not be cached at the CDN edge because their content depended on theme settings that could change, and each request required a full Liquid render pass on the file before delivery. For themes with large `.css.liquid` files containing dozens of settings interpolations, this added measurable latency to every page load.

As of 2025, Shopify has fully removed support for `.css.liquid` file processing in the Storefront Renderer. Files with these extensions are no longer passed through the Liquid engine — they are served as raw text files with the literal Liquid syntax intact. A CSS file containing `color: {{ settings.color_primary }};` is delivered to the browser as the string `color: {{ settings.color_primary }};`, which is not valid CSS syntax. The browser ignores the malformed rule entirely. The visual consequence is that all theme-settings-driven colors, fonts, and layout values revert to the browser's default styles — a complete visual regression that appears as unstyled content. JavaScript files containing Liquid similarly deliver raw `{{ }}` syntax to the browser's JavaScript engine, which produces a syntax error and may break all JavaScript on the page.

The replacement architecture is well-established in Online Store 2.0. CSS custom properties (CSS variables) are the correct mechanism for injecting settings values into stylesheets: a `<style>` block in the Liquid layout file (`layout/theme.liquid`) defines `:root { --color-primary: {{ settings.color_primary }}; }`, and the static CSS file references `var(--color-primary)` without any Liquid. For JavaScript files that require Liquid data, the correct pattern is to pass values via `data-*` attributes on HTML elements (set in Liquid templates) and read them in the static JavaScript file using `dataset` access — never inline Liquid in a `.js` file. The `{% javascript %}` tag block available in section files provides a scoped mechanism for section-level JavaScript that can reference section settings.

---

## Detection Logic (AST)

1. **Inspect the file extension and path.** This rule operates at the file-system level as a pre-AST check. Scan all files in the theme's `/assets/` directory (and any other directories) for filenames matching the glob patterns `*.css.liquid` and `*.js.liquid`.
2. **Confirm the file contains Liquid expressions.** Open the file and check whether its content contains `{{`, `{%`, or `{%-` sequences. A `.css.liquid` file with no Liquid content is a naming error rather than a functional regression, and should be flagged as LOW (wrong extension, no active Liquid).
3. **Identify Liquid expressions in CSS property values.** For `.css.liquid` files, parse the content and locate `{{ ... }}` expressions appearing as CSS property values, variable names, or selector content. Each one represents a setting value that is no longer being resolved.
4. **Identify Liquid expressions in JavaScript.** For `.js.liquid` files, locate `{{ ... }}` and `{% ... %}` expressions anywhere in the file content. These produce invalid JavaScript syntax when served as raw text.
5. **Emit CRITICAL violation per file.** Report the filename, the number of unresolvable Liquid expressions found, and a note that the file must be migrated to the CSS custom property or data-attribute pattern.

**Why regex fails here:** This rule's primary detection (file extension matching) does not require regex — it is a glob-pattern file-system scan. However, the secondary analysis (confirming that the file contains active Liquid expressions and classifying their types) benefits from AST parsing: regex would fail to distinguish `{{ settings.color }}` inside a CSS comment (harmless) from `{{ settings.color }}` inside a CSS property value (breakage), or to correctly handle multiline `{% if %}` blocks that span multiple lines in the file. AST parsing of the Liquid content within the file provides accurate expression location data.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: assets/theme.css.liquid — a .css.liquid file that injects
  theme settings directly into CSS rules. As of 2025, this file is served as
  raw text. All {{ settings.* }} expressions appear literally in the browser's
  CSS parser output, producing invalid CSS. Colors, fonts, and spacing all
  revert to browser defaults. The file also cannot be CDN-cached.
{%- endcomment -%}
```

```css
/* assets/theme.css.liquid — NO LONGER PROCESSED AS LIQUID */
:root {
  --font-size-base: {{ settings.type_base_size }}px;
}

body {
  font-family: {{ settings.type_body_font.family }}, sans-serif;
  font-size:   {{ settings.type_base_size }}px;
  color:       {{ settings.color_body_text }};
  background:  {{ settings.color_background_1 }};
}

.btn--primary {
  background-color: {{ settings.color_button }};
  color:            {{ settings.color_button_label }};
  border-radius:    {{ settings.buttons_border_radius }}px;
}

.header {
  background: {{ settings.color_header_bg }};
  height:     {{ settings.header_height }}px;
}
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Split into two files. In layout/theme.liquid, emit a <style> block that
  defines CSS custom properties from settings values. The static assets/theme.css
  file (no .liquid extension) references var(--property-name) — it is served
  directly from Shopify CDN with full caching, no Liquid processing needed.
  The <style> block in the layout is small (custom property declarations only)
  and is inlined in the <head> — it does not block CDN caching of the CSS file.
{%- endcomment -%}
```

In `layout/theme.liquid` `<head>`:

```liquid
<style>
  :root {
    --color-body-text:    {{ settings.color_body_text }};
    --color-background-1: {{ settings.color_background_1 }};
    --color-button:       {{ settings.color_button }};
    --color-button-label: {{ settings.color_button_label }};
    --color-header-bg:    {{ settings.color_header_bg }};
    --font-size-base:     {{ settings.type_base_size }}px;
    --header-height:      {{ settings.header_height }}px;
    --btn-border-radius:  {{ settings.buttons_border_radius }}px;
  }
</style>

{{ 'theme.css' | asset_url | stylesheet_tag }}
```

In `assets/theme.css` (static — no `.liquid` extension, fully CDN-cacheable):

```css
body {
  font-family: var(--font-body), sans-serif;
  font-size: var(--font-size-base);
  color: var(--color-body-text);
  background: var(--color-background-1);
}

.btn--primary {
  background-color: var(--color-button);
  color: var(--color-button-label);
  border-radius: var(--btn-border-radius);
}

.header {
  background: var(--color-header-bg);
  height: var(--header-height);
}
```

---

## References

- [Shopify: Migrate from .css.liquid and .js.liquid](https://shopify.dev/docs/themes/architecture/assets#css-liquid-and-js-liquid-files)
- [Shopify: CSS variables in themes](https://shopify.dev/docs/themes/architecture/assets#css-variables)
- [Shopify: section javascript tag](https://shopify.dev/docs/api/liquid/tags/javascript)
- [Shopify Theme Check: AssetUrlFilters](https://shopify.dev/docs/themes/tools/theme-check/checks)
- [MDN: CSS custom properties (variables)](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
