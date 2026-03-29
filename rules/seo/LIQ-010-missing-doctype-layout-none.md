# LIQ-010 | `{% layout none %}` without explicit `<!DOCTYPE html>`

**Category:** SEO

**Severity:** 🟡 MEDIUM

**Affects:** Any template in `/templates/` that uses `{% layout none %}` and renders full HTML documents

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{% layout none %}` instructs Shopify's Liquid renderer to bypass the theme's `layout/theme.liquid` file entirely, rendering the template as a self-contained document. This is the correct approach for AJAX endpoints that return partial HTML fragments, for custom JSON or XML responses, or for standalone pages (custom landing pages, AMP pages, email templates) that need a completely different document structure from the main theme layout. The problem arises when a template that renders a complete HTML document — with `<html>`, `<head>`, and `<body>` elements — omits the `<!DOCTYPE html>` declaration that all browsers require to engage standards-compliant rendering mode.

Without `<!DOCTYPE html>`, every browser that receives the document activates quirks mode. In quirks mode, the browser reverts to pre-standards rendering behavior from the Internet Explorer 5 era. The impact on layout is concrete and browser-specific: the CSS box model changes (padding and border are included in the declared width in some browsers, excluded in others), font size algorithms diverge from the CSS specification, certain flexbox and grid properties behave differently, and `vertical-align` on table cells uses quirks-specific calculations. A page that renders correctly in standards mode may have broken column alignment, incorrect button sizing, or font rendering inconsistencies in quirks mode, and these issues vary across browser vendors — making them difficult to reproduce and debug.

Googlebot's HTML parser documents quirks mode pages differently from standards mode pages. When Googlebot detects missing doctype, it flags the page as malformed HTML in the URL Inspection tool. Malformed HTML status suppresses rich snippet eligibility — structured data (JSON-LD, microdata) on a quirks mode page may not be extracted for rich results eligibility. For standalone landing pages using `{% layout none %}` with schema markup (product schema, FAQ schema, breadcrumb schema), the absence of `<!DOCTYPE html>` can prevent the structured data from being used in Google Search results, directly reducing click-through rate from organic search.

---

## Detection Logic (AST)

1. **Identify `LiquidTag[name=layout]` nodes with a `none` argument.** The AST walker traverses all `LiquidTag` nodes where `name === "layout"`. The `markup` property of this tag is inspected — if it equals `"none"` (the bare identifier), the file is a layout-bypassing template. Files without `{% layout none %}` are out of scope for this rule.

2. **Scan the template for an `HtmlDoctype` node.** Within the same file, the walker checks for the presence of an `HtmlDoctype` node — the AST representation of `<!DOCTYPE html>`. If no `HtmlDoctype` node exists anywhere in the template file, the rule proceeds to step 3.

3. **Determine whether the template renders an HTML document.** Not all `{% layout none %}` templates render full HTML — some return JSON, XML, or plain text. The walker checks for the presence of `HtmlElement[tag=html]` or `HtmlElement[tag=body]` nodes. If these are present, the template is rendering a full HTML document without a doctype, and the rule fires. If neither is present, the template is a non-HTML response and the doctype requirement does not apply.

4. **Classify template type for fix guidance.** If the template contains `HtmlElement[tag=head]` with `HtmlElement[tag=script]` or `HtmlElement[tag=link]` children, it is a standalone page (likely a custom landing page). If it contains `HtmlRawElement[tag=script]` with `type=application/json`, it may be an AJAX partial — the walker re-evaluates whether a doctype is needed. Only full-document templates (those with `<html lang>`, `<head>`, and `<body>`) are flagged.

**Why regex fails here:** A regex checking for the absence of `<!DOCTYPE html>` within the first N lines of a file would misfire on templates that begin with Liquid tags (`{% layout none %}`, `{%- comment -%}`, `{% assign %}`) before the doctype — the doctype may appear on line 5 or later, past a character offset limit, yet still be valid. The AST `HtmlDoctype` node is position-independent; its presence or absence in the node tree is unambiguous regardless of where in the source it appears.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: {% layout none %} with a full HTML document but no <!DOCTYPE html>.

  Failure mode 1: All browsers render in quirks mode.
  CSS box model: padding included in declared width (IE5 behavior).
  Font sizing: browser-specific non-standard algorithms apply.
  Flexbox: some properties behave differently from CSS spec in quirks mode.

  Failure mode 2: Googlebot flags page as malformed HTML.
  JSON-LD schema on this page (Product schema below) may not be extracted
  for rich results. Loss of rich snippet eligibility in Google Search.

  Failure mode 3: lang attribute on <html> is absent — screen readers
  cannot determine the document language for correct pronunciation engine
  selection. EAA 2025 compliance risk for EU markets.
{%- endcomment -%}

{% layout none %}
<html>
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    {{ content_for_header }}
    <title>{{ page.title }} — {{ shop.name }}</title>
    {{ 'landing-page.css' | asset_url | stylesheet_tag }}

    <script type="application/ld+json">
      {
        "@context": "https://schema.org",
        "@type": "Product",
        "name": {{ product.title | json }},
        "offers": {
          "@type": "Offer",
          "price": "{{ product.price | money_without_currency }}",
          "priceCurrency": "{{ cart.currency.iso_code }}"
        }
      }
    </script>
  </head>
  <body>
    <main>
      <h1>{{ page.title }}</h1>
      <div class="page-content">{{ page.content }}</div>
    </main>
  </body>
</html>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add <!DOCTYPE html> as the first declaration after {% layout none %}.

  Change 1: <!DOCTYPE html> on the first content line after the layout tag.
  This forces standards mode in all browsers. The CSS box model, font sizing,
  and flexbox behavior all conform to the CSS specification.

  Change 2: lang="{{ request.locale.iso_code }}" on <html>.
  Declares the document language — required for screen reader pronunciation
  accuracy and for EAA 2025 / WCAG 2.1 Success Criterion 3.1.1 compliance.

  Change 3: JSON-LD structured data is now in a standards-mode document.
  Googlebot will correctly parse and extract the Product schema for rich results.
{%- endcomment -%}

{% layout none %}
<!DOCTYPE html>
<html lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    {%- comment -%}
      content_for_header required even in layout-none templates —
      injects analytics and app scripts for this standalone page.
    {%- endcomment -%}
    {{ content_for_header }}

    <title>{{ page.title | escape }} — {{ shop.name | escape }}</title>
    <meta name="description" content="{{ page.metafields.seo.description.value | default: page.content | strip_html | truncate: 155 | escape }}">

    {{ 'landing-page.css' | asset_url | stylesheet_tag }}

    {%- comment -%}
      JSON-LD in a standards-mode document — correctly extracted by Googlebot
      for Product rich results eligibility in Google Search.
    {%- endcomment -%}
    <script type="application/ld+json">
      {
        "@context": "https://schema.org",
        "@type": "Product",
        "name": {{ product.title | json }},
        "offers": {
          "@type": "Offer",
          "price": "{{ product.price | money_without_currency }}",
          "priceCurrency": {{ cart.currency.iso_code | json }}
        }
      }
    </script>
  </head>
  <body>
    <main id="main-content" role="main" tabindex="-1">
      <h1>{{ page.title }}</h1>
      <div class="page-content">{{ page.content }}</div>
    </main>
  </body>
</html>
```

---

## References

- [Shopify Liquid — `layout` tag](https://shopify.dev/docs/api/liquid/tags/layout)
- [WHATWG — Quirks mode specification](https://quirks.spec.whatwg.org/)
- [Google Search Central — Structured data guidelines](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)
- [WCAG 2.1 — Success Criterion 3.1.1 Language of Page](https://www.w3.org/TR/WCAG21/#language-of-page)
- [MDN — Quirks mode and standards mode](https://developer.mozilla.org/en-US/docs/Web/HTML/Quirks_Mode_and_Standards_Mode)
