# LIQ-014 | `| strip_html` on an already plain-text field

**Category:** Performance

**Severity:** 🟢 LOW

**Affects:** All templates referencing `product.title`, `variant.title`, `product.handle`, `variant.sku`, `product.vendor`, `product.type`, `collection.title`,
`collection.handle`

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`| strip_html` is implemented in Liquid as a two-pass operation: first, HTML entities are decoded (`&amp;` → `&`, `&lt;` → `<`, etc.), then a tag-stripping regex removes anything matching `<[^>]+>`. Both passes execute unconditionally regardless of whether the input contains any HTML markup. On plain-text fields — `product.title`, `variant.title`, `product.handle`, `variant.sku`, `product.vendor`, `product.type`, `collection.title`, and `collection.handle` — the Shopify storefront API guarantees that values are stored without HTML markup. The filter runs the full parsing pipeline and returns the input unchanged. On a product page with 50+ Liquid expressions, redundant `strip_html` calls across title, vendor, and SKU fields accumulate into measurable CPU overhead per render cycle on Shopify's shared rendering infrastructure.

The more consequential issue is correctness. Plain-text fields accept and store characters that are syntactically significant in HTML: `<`, `>`, and `&`. A product titled `"Cable Gauge 14 < 20 AWG"` or a vendor named `"R&D Supplies"` will be mutated by `strip_html`. The entity-decode pass converts stored `&amp;` sequences to `&`, and the tag-stripping pass removes content matching `<[^>]+>`. For the title `"Cable Gauge 14 < 20 AWG"`, the `<` character causes `< 20 AWG>` to be matched as an HTML tag and stripped — the rendered output becomes `"Cable Gauge 14"`. This is a silent data-corruption bug: no error is raised, the truncated title renders confidently, and the defect only surfaces when a merchant creates a product whose name contains a `<` character.

The pattern consistently appears when developers copy filter chains from `product.description` (a rich-text HTML field that legitimately requires `strip_html` for truncation or text comparison) and paste them onto title or SKU references without auditing the destination field's data type. This signals a fundamental misunderstanding of the Shopify data model. Theme Store reviewers flag gratuitous `strip_html` on plain-text fields as evidence of a poorly understood content model, and it has contributed to outright rejection during the Theme Store submission review process. The fix is removal — not substitution — of the filter.

---

## Detection Logic (AST)

1. **Identify `LiquidVariable` nodes.** Walk the full AST for all `LiquidVariable` nodes (output expressions `{{ ... }}`) and the right-hand sides of `LiquidTag` `assign` statements.
2. **Collect the filter chain.** For each `LiquidVariable`, extract the ordered list of `LiquidFilter` child nodes from the variable's filter chain.
3. **Check for `strip_html` filter presence.** If any `LiquidFilter` has `name === 'strip_html'`, proceed to step 4.
4. **Resolve the root object and property.** Traverse the `LiquidVariable`'s expression to its root `VariableLookup` node. Follow the lookup chain: if the root identifier resolves to a known Shopify object (`product`, `variant`, `collection`, `line_item`, `cart`, `customer`) and the next lookup segment is a statically known plain-text property (`title`, `handle`, `sku`, `vendor`, `type`, `barcode`, `name`, `email`), emit a LOW violation with the field name in the diagnostic message.
5. **Classify data-corruption variant.** If the field is one that commonly holds user-entered free text with special characters (`title`, `vendor`, `name`), annotate the violation with the data-corruption sub-class to elevate operator awareness.

**Why regex fails here:** A regex matching `strip_html` on the same line as `product.title` correctly catches `{{ product.title | strip_html }}` but cannot distinguish `{{ product.title | strip_html }}` (violation) from `{{ product.title_tag | strip_html }}` (`title_tag` is an HTML string — the filter is appropriate). It also cannot resolve aliased variables: `{% assign t = product %} {{ t.title | strip_html }}` requires tracking the `assign` binding across lines and following the object reference chain — a scope-tracking operation that requires AST traversal with a symbol table, not line-by-line pattern matching.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: strip_html applied to plain-text fields throughout a product card.
  product.title, product.vendor, variant.sku, and product.handle are all stored
  as plain text by Shopify's API — no HTML tags will ever appear in these values.
  The filter runs a full entity-decode + tag-strip pipeline on each call, consuming
  CPU for zero benefit. A title like "Size S < L" is silently truncated to "Size S".
{%- endcomment -%}

{%- assign product_title = product.title | strip_html | escape -%}
{%- assign vendor_name   = product.vendor | strip_html | upcase -%}
{%- assign display_sku   = variant.sku | strip_html | default: 'N/A' -%}
{%- assign clean_handle  = product.handle | strip_html -%}

<article class="product-card" data-handle="{{ clean_handle }}">
  <header class="product-card__header">
    <h2 class="product-card__title">{{ product_title }}</h2>
    <span class="product-card__vendor">{{ vendor_name }}</span>
  </header>

  <div class="product-card__meta">
    {%- if display_sku != 'N/A' -%}
      <span class="product-card__sku">SKU: {{ display_sku }}</span>
    {%- endif -%}

    {{- /* handle is a URL slug — strip_html on it achieves nothing and risks corruption */ -}}
    <a href="/products/{{ clean_handle }}" class="product-card__link">
      View {{ product_title }}
    </a>
  </div>
</article>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Remove strip_html from all plain-text fields. These fields are stored without
  HTML by Shopify's API contract — strip_html is a no-op at best and a data mutator
  at worst. escape is the correct output filter for HTML attribute contexts only.
  In HTML body context, Liquid auto-escapes variable output, so no filter is needed.
  product.handle is a URL-safe slug by definition — neither strip_html nor escape
  is appropriate; it can be interpolated directly into URL path segments.
{%- endcomment -%}

{%- assign product_title = product.title -%}
{%- assign vendor_name   = product.vendor | upcase -%}
{%- assign display_sku   = variant.sku | default: 'N/A' -%}
{%- assign clean_handle  = product.handle -%}

<article class="product-card" data-handle="{{ clean_handle | escape }}">
  <header class="product-card__header">
    {{- /* product.title is plain text; auto-escaped in HTML body context */ -}}
    <h2 class="product-card__title">{{ product_title }}</h2>
    <span class="product-card__vendor">{{ vendor_name }}</span>
  </header>

  <div class="product-card__meta">
    {%- if display_sku != 'N/A' -%}
      <span class="product-card__sku">SKU: {{ display_sku }}</span>
    {%- endif -%}

    {{- /* handle is a URL-safe slug — safe to interpolate directly in path segments */ -}}
    <a href="/products/{{ clean_handle }}" class="product-card__link">
      View {{ product_title }}
    </a>
  </div>
</article>
```

---

## References

- [Shopify Liquid: strip_html filter](https://shopify.dev/docs/api/liquid/filters/strip_html)
- [Shopify Liquid: product object — field types](https://shopify.dev/docs/api/liquid/objects/product)
- [Shopify Liquid: variant object — field types](https://shopify.dev/docs/api/liquid/objects/variant)
- [Shopify Theme Check: checks reference](https://shopify.dev/docs/themes/tools/theme-check/checks)
