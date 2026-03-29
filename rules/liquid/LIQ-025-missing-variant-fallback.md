# LIQ-025 | `{{ product.selected_variant }}` instead of `selected_or_first_available_variant`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** `templates/product.liquid`, `sections/product-information.liquid`, `sections/main-product.liquid`, and any snippet rendering product price,add-to-cart form, or inventory status

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`product.selected_variant` returns the variant object corresponding to the `?variant=` URL parameter. If the URL contains `?variant=12345678`, the property resolves to that variant object. If no `?variant=` parameter is present in the URL — which is the case for every direct product page access, every social media link, every search engine crawl, every email campaign link that does not include a variant ID, and every internal navigation from a collection page that links to `/products/handle` without a query parameter — `product.selected_variant` returns `nil`. This is documented behaviour, not a bug in Shopify's implementation. The property is intentionally nil-returning when no variant is URL-selected.

The consequence of nil propagation through variant-dependent expressions is total page dysfunction. `product.selected_variant.price` returns nil — the price display renders blank. `product.selected_variant.available` returns nil — the add-to-cart button cannot determine inventory state and its JavaScript-controlled disabled/enabled state is based on a nil value, which evaluates as falsy. `product.selected_variant.id` returns nil — the hidden `<input name="id">` in the add-to-cart form receives no value, and the form submission fails with a Shopify API error ("Variant id is required"). `product.selected_variant.sku`, `product.selected_variant.compare_at_price`, and `product.selected_variant.featured_image` all return nil. The product page renders visually incomplete on every cold load and on every search-engine crawl — directly harming SEO quality signals and conversion rates for direct-traffic visitors.

`product.selected_or_first_available_variant` implements the correct fallback chain: it returns the variant matching the `?variant=` URL parameter if present, and falls back to the first variant marked `available: true` if no URL parameter is present, and falls back to the first variant in the variants array if no variants are available. This ensures that the product page always renders with a valid variant context regardless of the URL state. Theme Store requirements mandate the use of `selected_or_first_available_variant` in product page templates; themes that use bare `selected_variant` without a nil guard are flagged during automated product page testing, which verifies correct rendering on direct URL access without a variant parameter.

---

## Detection Logic (AST)

1. **Identify `VariableLookup` nodes accessing `selected_variant`.** Walk the AST for all `VariableLookup` nodes where the lookup chain contains the property name `selected_variant` on a `product`-rooted expression (i.e., `product.selected_variant`, or a variable assigned from `product.selected_variant`).
2. **Verify the root object resolves to `product`.** Confirm that the root of the lookup chain is either the Shopify global `product` object, or a local variable whose `assign` right-hand side ultimately resolves to `product`. This excludes false-positive matches on user-defined objects named `selected_variant`.
3. **Check for nil guard.** Inspect the parent `LiquidTag` or `LiquidBranch` containing the `selected_variant` access. If the access is within a `{% if product.selected_variant %}` condition block or an equivalent nil-guard pattern (`{% unless product.selected_variant == nil %}`), classify the violation as LOW (nil is guarded) rather than HIGH (nil is unguarded and propagates to output).
4. **Check for `selected_or_first_available_variant` usage in the same file.** If `selected_or_first_available_variant` is used elsewhere in the same template, annotate the violation: "File uses both `selected_variant` and `selected_or_first_available_variant` — verify the `selected_variant` reference is intentional."
5. **Emit HIGH violation for unguarded usages.** Report the property access location, the downstream expressions that receive the nil value (`selected_variant.price`, `selected_variant.id`, etc.), and the recommended replacement.

**Why regex fails here:** A regex matching `selected_variant` catches both `product.selected_variant` (violation) and `product.selected_or_first_available_variant` (correct — the regex would match the `selected_variant` substring within it). A more specific regex matching `selected_variant[^_]` or `selected_variant$` would avoid the substring match but still cannot determine whether the access is nil-guarded — that requires traversing the ancestor `LiquidTag` and `LiquidBranch` nodes to find a containing `if product.selected_variant` check, which is a structural parent-traversal operation unavailable to pattern matching.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: product.selected_variant is used throughout the product page
  template without nil guarding. On every direct URL access (/products/handle
  with no ?variant= parameter), selected_variant is nil. Price renders blank,
  the add-to-cart form has no variant id, available is nil (falsy — button
  may show incorrectly as sold out), and media gallery has no variant image.
  Search engines crawl product pages without ?variant= parameters — they index
  pages with blank prices and non-functional add-to-cart forms.
{%- endcomment -%}

{%- assign current_variant = product.selected_variant -%}

<div class="product__info">
  <h1>{{ product.title }}</h1>

  {{- /* nil on direct URL access — renders blank */ -}}
  <div class="product__price">
    {%- if current_variant.compare_at_price > current_variant.price -%}
      <s class="price--compare">{{ current_variant.compare_at_price | money }}</s>
    {%- endif -%}
    <span class="price">{{ current_variant.price | money }}</span>
  </div>

  <form action="/cart/add" method="post">
    {{- /* nil — form submission fails: "Variant id is required" */ -}}
    <input type="hidden" name="id" value="{{ current_variant.id }}">

    <button
      type="submit"
      {% unless current_variant.available %}disabled{% endunless %}
    >
      {{- /* available is nil — unless nil evaluates as true — button is disabled */ -}}
      Add to Cart
    </button>
  </form>

  {{- /* nil — no image renders */ -}}
  {%- if current_variant.featured_image -%}
    <img src="{{ current_variant.featured_image | image_url: width: 800 }}"
         alt="{{ current_variant.featured_image.alt | escape }}">
  {%- endif -%}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace product.selected_variant with product.selected_or_first_available_variant.
  This property falls back to the first available variant when no ?variant= URL
  parameter is present, ensuring the page always renders with a valid variant object.
  All downstream property accesses (price, id, available, featured_image) resolve
  to real values on every page load — direct URL, social share, or search crawl.
  The variant selector JavaScript updates the form and price display when the
  customer selects a different variant; the initial server render is always valid.
{%- endcomment -%}

{%- assign current_variant = product.selected_or_first_available_variant -%}

<div class="product__info">
  <h1>{{ product.title }}</h1>

  {{- /* Always resolves — falls back to first available variant price */ -}}
  <div class="product__price">
    {%- if current_variant.compare_at_price > current_variant.price -%}
      <s class="price--compare">{{ current_variant.compare_at_price | money }}</s>
    {%- endif -%}
    <span class="price">{{ current_variant.price | money }}</span>
  </div>

  <form action="/cart/add" method="post">
    {{- /* Always a valid integer — form submission succeeds on first load */ -}}
    <input type="hidden" name="id" value="{{ current_variant.id }}">

    <button
      type="submit"
      {% unless current_variant.available %}disabled{% endunless %}
    >
      {%- if current_variant.available -%}
        Add to Cart
      {%- else -%}
        Sold Out
      {%- endif -%}
    </button>
  </form>

  {{- /* Resolves to first available variant's image on direct page load */ -}}
  {%- if current_variant.featured_image -%}
    <img src="{{ current_variant.featured_image | image_url: width: 800 }}"
         alt="{{ current_variant.featured_image.alt | default: product.title | escape }}">
  {%- endif -%}
</div>
```

---

## References

- [Shopify Liquid: product.selected_or_first_available_variant](https://shopify.dev/docs/api/liquid/objects/product#product-selected_or_first_available_variant)
- [Shopify Liquid: product.selected_variant](https://shopify.dev/docs/api/liquid/objects/product#product-selected_variant)
- [Shopify Theme Store: product page requirements](https://shopify.dev/docs/themes/store/requirements#product-pages)
- [Shopify Theme Check: checks reference](https://shopify.dev/docs/themes/tools/theme-check/checks)
- [Shopify: Variant selection — best practices](https://shopify.dev/docs/themes/product-merchandising/variants)
