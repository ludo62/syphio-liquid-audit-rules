# PERF-011 | `section.settings` accessed inside `{% for %}` loop

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Collection page sections, product grid sections, article list sections — any section template that iterates over a collection or list inside a `{% for %}` loop while accessing `section.settings`

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`section.settings` is the Liquid object that exposes a section's schema-defined settings values. Each property access — `section.settings.show_vendor`, `section.settings.columns_desktop`, `section.settings.color_scheme` — performs a hash lookup in the settings store for the current section context. While each individual lookup is fast (sub-millisecond in isolation), the lookup is not cached between calls within a single render. When settings are accessed inside a `{% for %}` loop over a product collection, the same setting key is looked up anew on every iteration. For a collection page showing 24 products with 5 settings accessed per product card render, this produces 120 redundant setting lookups per page render. On a section with 48 products (the maximum per paginated page) and 8 settings accesses per card, this is 384 redundant lookups.

The cumulative rendering overhead of 384 redundant hash lookups is measured in tens of microseconds — not milliseconds. Individually, this rule is the lowest severity among the PERF series. However, it appears almost universally in themes that access `section.settings` without discipline, and it compounds with other loop-body inefficiencies: if the same loop also accesses `section.blocks` on every iteration, or performs `| where` filtering inside the loop (PERF-012), or uses `all_products[handle]` (PERF-002), the cumulative overhead across all N iterations becomes material. The pattern is also a code quality signal — it indicates that the developer treats `section.settings` as a free-access global rather than a hash that should be resolved once. In themes with 20+ settings per section (common in highly configurable commercial themes), the total lookup count per product card can reach 15–20, making loop-body settings access a measurable contributor to render time on large collections.

The fix is a single-line `{% assign %}` hoist above the `{% for %}` loop for each setting that is accessed inside the loop body. `{% assign show_vendor = section.settings.show_vendor %}` reduces the lookup count from N (one per iteration) to 1 (once before the loop). The assigned variable is a primitive (boolean, integer, string, or color) — it is lightweight to store and costs no allocation overhead. This pattern is used consistently in Shopify's Dawn theme: all section settings used inside `{% for %}` loops over collections are hoisted to `{% assign %}` variables in the loop's preamble.

---

## Detection Logic (AST)

1. **Identify all `LiquidTag[for]` blocks in section files.** The AST walker collects `LiquidTagOpen` nodes where `name === "for"` within section template files (`.liquid` files under the `sections/` directory, identified by file path). The loop body scope extends from the `LiquidTagOpen[for]` to its matching `LiquidTagClose[endfor]`.

2. **Within each loop body, find `LiquidVariable` or `LiquidTag` nodes referencing `section.settings.*`.** Traverse all descendant nodes of the `LiquidTagOpen[for]` node. Identify any `MemberExpression` (or equivalent) where the root object is `section`, the first member is `settings`, and a second member (the setting key) follows. Record each unique setting key accessed inside the loop.

3. **Check whether the setting key is pre-assigned above the loop.** For each unique setting key identified in step 2, traverse the loop's preceding siblings (nodes that appear before the `LiquidTagOpen[for]` node in the same parent scope). If a `LiquidTag[assign]` node exists whose right-hand side is the same `section.settings.[key]` expression, the setting is already hoisted — exclude it from the violation.

4. **Report violation with iteration count estimate.** If the loop's iterable can be statically analyzed (e.g., `for product in collection.products` — collection size is dynamic but typically 24–48 per page with pagination), report the unhoisted setting keys and their estimated lookup count (minimum: the number of iterations times the number of unhoisted accesses). This gives developers a concrete cost estimate in the violation message.

**Why regex fails here:** A regex matching `section\.settings\.\w+` inside a `{% for %}` block requires the regex to track block boundaries — `{% for %}` open and `{% endfor %}` close — as scope delimiters. In templates where `{% for %}` loops contain nested `{% if %}`, `{% unless %}`, `{% capture %}`, or inner `{% for %}` blocks, the regex must correctly identify which settings access is inside the outer loop vs. in a nested block. It also cannot determine whether a given setting access is already covered by an upstream `{% assign %}` hoist. The AST walker handles all nesting via recursive descent with explicit scope boundaries from the parsed block structure.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: section.settings accessed repeatedly inside a {% for %} loop.
  Context: A product grid section on a collection page — 24+ products per page.

  Failure mode 1: show_vendor, columns_desktop, columns_mobile, show_rating,
  and show_quick_add are each looked up from the settings hash on every
  iteration. For 24 products × 5 settings = 120 redundant lookups per render.

  Failure mode 2: section.settings.color_scheme is used to build a CSS class
  string on every iteration — string concatenation + settings lookup per card.

  Failure mode 3: section.settings.image_ratio conditional is evaluated per
  card — a boolean setting resolved 24 times instead of 1.

  Failure mode 4: This pattern scales linearly with product count. On a
  collection page using 48 products per page: 288+ redundant lookups.
{%- endcomment -%}

<div
  class="collection-grid"
  data-section-id="{{ section.id }}"
  style="--grid-columns-desktop: {{ section.settings.columns_desktop }}; --grid-columns-mobile: {{ section.settings.columns_mobile }};"
>
  {%- for product in collection.products -%}
    <div class="product-card color-{{ section.settings.color_scheme }}">
      {%- if product.featured_image != blank -%}
        <div class="product-card__media product-card__media--{{ section.settings.image_ratio }}">
          {{
            product.featured_image
            | image_url: width: 500
            | image_tag: loading: 'lazy', alt: product.featured_image.alt
          }}
        </div>
      {%- endif -%}

      <div class="product-card__info">
        {%- if section.settings.show_vendor -%}
          <p class="product-card__vendor">{{ product.vendor }}</p>
        {%- endif -%}

        <h3 class="product-card__title">{{ product.title }}</h3>

        <p class="product-card__price">{{ product.price | money }}</p>

        {%- if section.settings.show_rating and product.metafields.reviews.rating != blank -%}
          <div class="product-card__rating">
            {{ product.metafields.reviews.rating.value | round: 1 }} / 5
          </div>
        {%- endif -%}

        {%- if section.settings.show_quick_add and product.available -%}
          <button class="product-card__quick-add btn" data-product-id="{{ product.id }}">
            {{ 'products.product.add_to_cart' | t }}
          </button>
        {%- endif -%}
      </div>
    </div>
  {%- endfor -%}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Hoist all section.settings accesses to {% assign %} variables above
  the {% for %} loop. Each setting is resolved exactly once per render,
  regardless of the collection's product count.

  Change 1: Five settings hoisted to five {% assign %} variables. Inside the
  loop, variable access replaces settings hash lookups — 120 lookups become 5.

  Change 2: color_scheme and image_ratio hoisted to strings — the CSS class
  computation is also moved outside the loop (applied to the wrapper element).
  The inner cards inherit the class context from the wrapper via CSS cascade.

  Change 3: columns_desktop and columns_mobile remain on the wrapper element
  as CSS custom properties — they only need to be evaluated once, at the
  wrapper level, not once per card.

  Change 4: Readability improves — the loop body reads assigned variables
  rather than long chained object accessors.
{%- endcomment -%}

{%- comment -%}Hoist all section.settings used inside the loop — one lookup each.{%- endcomment -%}
{%- assign show_vendor = section.settings.show_vendor -%}
{%- assign show_rating = section.settings.show_rating -%}
{%- assign show_quick_add = section.settings.show_quick_add -%}
{%- assign image_ratio = section.settings.image_ratio -%}
{%- assign color_scheme = section.settings.color_scheme -%}

<div
  class="collection-grid color-{{ color_scheme }}"
  data-section-id="{{ section.id }}"
  style="
    --grid-columns-desktop: {{ section.settings.columns_desktop }};
    --grid-columns-mobile: {{ section.settings.columns_mobile }};
  "
>
  {%- for product in collection.products -%}
    {%- comment -%}
      Inside the loop: all variables are pre-resolved scalars.
      No section.settings lookups occur below this line.
    {%- endcomment -%}
    <div class="product-card">
      {%- if product.featured_image != blank -%}
        <div class="product-card__media product-card__media--{{ image_ratio }}">
          {{-
            product.featured_image
            | image_url: width: 500
            | image_tag:
              loading: 'lazy',
              widths: '250,500,750',
              width: product.featured_image.width,
              height: product.featured_image.height,
              alt: product.featured_image.alt
          -}}
        </div>
      {%- endif -%}

      <div class="product-card__info">
        {%- if show_vendor -%}
          <p class="product-card__vendor">{{ product.vendor }}</p>
        {%- endif -%}

        <h3 class="product-card__title">{{ product.title }}</h3>

        <p class="product-card__price">{{ product.price | money }}</p>

        {%- if show_rating and product.metafields.reviews.rating != blank -%}
          <div class="product-card__rating">
            {{ product.metafields.reviews.rating.value | round: 1 }} / 5
          </div>
        {%- endif -%}

        {%- if show_quick_add and product.available -%}
          <button
            class="product-card__quick-add btn"
            data-product-id="{{ product.id }}"
            data-variant-id="{{ product.selected_or_first_available_variant.id }}"
          >
            {{ 'products.product.add_to_cart' | t }}
          </button>
        {%- endif -%}
      </div>
    </div>
  {%- endfor -%}
</div>
```

---

## References

- [Shopify — `section.settings` object](https://shopify.dev/docs/api/liquid/objects/section#section-settings)
- [Shopify — Dawn theme: settings hoisting pattern](https://github.com/Shopify/dawn/blob/main/sections/main-collection-product-grid.liquid)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify — Section schema settings](https://shopify.dev/docs/storefronts/themes/architecture/sections/section-schema#settings)
- [Liquid — `assign` tag](https://shopify.dev/docs/api/liquid/tags/assign)
