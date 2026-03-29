# PERF-014 | `font_face` without `font_display: 'swap'`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Layout files, section files, snippets that generate `@font-face` CSS via Shopify's `font_face` filter

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Shopify's `font_face` Liquid filter generates a CSS `@font-face` block from a font object retrieved via `shop.metafields`, a theme font setting, or the `settings_schema.json` font picker. The filter accepts a `font_display` keyword argument that maps directly to the CSS `font-display` descriptor inside the generated block. When `font_display` is omitted, the Shopify Liquid runtime defaults to emitting no `font-display` descriptor at all, which causes browsers to fall back to the UA default behavior. In Chrome and Firefox, the UA default for `font-display` is equivalent to `auto` — a heuristic that typically resolves to `block` on slow connections, making all text rendered in the custom font invisible for up to 3 seconds while the font file transfers from Shopify's CDN. This 3-second window is the Flash of Invisible Text (FOIT) interval.

The LCP impact of FOIT is direct and measurable. If the Largest Contentful Paint element is a text node — a heading, a product title, a promotional banner — rendered using the custom font, the browser's LCP timestamp is recorded only when the text becomes visible. On a 3G connection (1.6 Mbps, typical for mobile users in emerging markets), a 40KB WOFF2 font file takes approximately 200ms to transfer; however, because `font-display: block` holds text invisible for the full block period regardless of actual transfer time, the LCP is delayed by the entire FOIT window, not just the transfer time. In Google's CrUX data, themes with FOIT-inducing font loading strategies consistently score 200–800ms worse on LCP than equivalent themes with `font-display: swap`.

The `font_display: 'swap'` argument instructs the browser to render text immediately using the best available fallback system font, then swap to the custom font when the WOFF2 file has loaded. The visual swap (Flash of Unstyled Text, FOUT) is perceptible but does not affect LCP — the text node is visible from first paint, and the LCP clock stops at first paint of the text element. Shopify's official font loading documentation mandates `font_display: 'swap'` as the required argument for any `font_face` invocation in a Theme Store-eligible theme.

---

## Detection Logic (AST)

1. **Identify `LiquidFilter` nodes named `font_face`.** The AST walker traverses all `LiquidVariable` nodes and inspects the `filters` array on each. Any filter with `name === "font_face"` is a candidate for inspection.

2. **Inspect the filter's `args` array for a `font_display` named argument.** Each `LiquidFilter` node carries an `args` array of positional and named arguments. Walk the `args` array looking for a `NamedArgument` node where `name === "font_display"`. If no such named argument exists, the call is a violation.

3. **If `font_display` is present, inspect its value node.** The `NamedArgument.value` is a `String` or `LiquidVariable` node. If the value is the string literal `'swap'`, the call is compliant. If the value is any other string literal (`'block'`, `'fallback'`, `'optional'`, `'auto'`), emit a warning with severity MEDIUM noting non-swap display mode. If the value is a variable reference, emit a LOW-severity advisory that the value must be verified at runtime.

4. **Classify severity by call site context.** A `font_face` call inside a `<style>` block that is rendered in a layout file's `<head>` is HIGH severity (affects every page render). A `font_face` call inside a section-scoped `<style>` tag is also HIGH (sections can be used on any page). There is no safe call site for `font_face` without `font_display: 'swap'` in a storefront context.

**Why regex fails here:** A regex scanning for `font_face` without `font_display` cannot distinguish between `font_face` used as a Liquid filter and the string `font_face` appearing in a CSS comment or a JavaScript variable name. More critically, a regex cannot parse the filter's argument list to confirm whether `font_display` is present as a named argument versus a positional value — a pattern like `font_face.*font_display` would match cases where `font_display` appears in a comment inside the same `{% %}` block. The AST walker resolves `LiquidFilter.args` as a typed list of `NamedArgument` and `PositionalArgument` nodes, making argument presence and value checks exact and unambiguous.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: font_face filter called without font_display: 'swap'.

  Failure mode 1: No font-display descriptor is emitted in the @font-face block.
  Chrome and Firefox default to font-display: auto which behaves like 'block'
  on slow connections — text is invisible for up to 3 seconds (FOIT).

  Failure mode 2: LCP text elements are invisible during the FOIT window.
  If the hero heading or product title uses this font, the LCP timestamp is
  delayed by the entire 3-second block period on 3G connections.

  Failure mode 3: Multiple font weights are all loaded blocking. A theme that
  calls font_face for regular, bold, and italic variants without swap multiplies
  the FOIT risk — any single slow font load blocks the entire text layer.

  Failure mode 4: The body_font_bold variant is also missing font_display,
  meaning headings and body text are both subject to FOIT simultaneously.
{%- endcomment -%}

{%- assign body_font = settings.type_body_font -%}
{%- assign body_font_bold = body_font | font_modify: 'weight', 'bold' -%}
{%- assign heading_font = settings.type_header_font -%}

<style>
  {{ body_font | font_face }}
  {{ body_font_bold | font_face }}
  {{ heading_font | font_face }}

  :root {
    --font-body-family: {{ body_font.family }}, {{ body_font.fallback_families }};
    --font-body-style: {{ body_font.style }};
    --font-body-weight: {{ body_font.weight }};
    --font-heading-family: {{ heading_font.family }}, {{ heading_font.fallback_families }};
  }

  body {
    font-family: var(--font-body-family);
    font-style: var(--font-body-style);
    font-weight: var(--font-body-weight);
  }

  h1, h2, h3, h4 {
    font-family: var(--font-heading-family);
  }
</style>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Pass font_display: 'swap' to every font_face invocation.

  Change 1: font_display: 'swap' emits font-display: swap in the @font-face
  block. The browser renders text immediately in the fallback system font
  and swaps to the custom font when the WOFF2 file finishes loading.
  Text is never invisible — FOIT is eliminated entirely.

  Change 2: All three font variants (body, bold, heading) receive the swap
  argument. Missing it on even one variant reintroduces FOIT for that weight.

  Change 3: preload link hints for the primary body and heading font files
  should accompany this — add <link rel="preload" as="font" crossorigin>
  in the layout <head> to reduce the time-to-swap from visible fallback
  to the custom font, minimizing the FOUT window duration.
{%- endcomment -%}

{%- assign body_font = settings.type_body_font -%}
{%- assign body_font_bold = body_font | font_modify: 'weight', 'bold' -%}
{%- assign heading_font = settings.type_header_font -%}

<style>
  {{ body_font | font_face: font_display: 'swap' }}
  {{ body_font_bold | font_face: font_display: 'swap' }}
  {{ heading_font | font_face: font_display: 'swap' }}

  :root {
    --font-body-family: {{ body_font.family }}, {{ body_font.fallback_families }};
    --font-body-style: {{ body_font.style }};
    --font-body-weight: {{ body_font.weight }};
    --font-heading-family: {{ heading_font.family }}, {{ heading_font.fallback_families }};
  }

  body {
    font-family: var(--font-body-family);
    font-style: var(--font-body-style);
    font-weight: var(--font-body-weight);
  }

  h1, h2, h3, h4 {
    font-family: var(--font-heading-family);
  }
</style>
```

---

## References

- [Shopify — `font_face` filter](https://shopify.dev/docs/api/liquid/filters/font_face)
- [Shopify — Font settings best practices](https://shopify.dev/docs/storefronts/themes/architecture/settings/fonts)
- [MDN — CSS `font-display` descriptor](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display)
- [web.dev — Avoid Flash of Invisible Text (FOIT)](https://web.dev/articles/avoid-invisible-text)
- [Google — Largest Contentful Paint (LCP) optimization](https://web.dev/articles/optimize-lcp)
