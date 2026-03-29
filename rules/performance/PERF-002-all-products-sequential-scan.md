# PERF-002 | `all_products[handle]` — sequential O(n) scan

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** Any template or snippet using `all_products[handle]` — product pages, cross-sell sections, cart recommendations, search results

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`all_products` is a Shopify Liquid global that maps product handles to product objects using bracket-accessor syntax: `all_products['my-product-handle']`. The bracket syntax is visually identical to a hash lookup and is commonly assumed to be O(1). It is not. In Shopify's Liquid runtime, `all_products[handle]` performs a sequential linear scan of the store's full product catalog until the matching handle is encountered. On a store with 1,000 products, a single call takes 50–200ms of server-side render time, depending on catalog structure and edge node load. The scan does not terminate early on the first match in all runtime versions — behavior varies by Shopify's Liquid execution engine revision, but no documented guarantee of early-exit optimization exists.

When `all_products[handle]` appears inside a `{% for %}` loop — even a short one — the penalty multiplies per iteration. A cross-sell snippet that resolves 5 related product handles via `all_products[handle]` in a loop produces 250ms–1s of cumulative scan overhead per page render. On high-traffic product pages, this is not amortized: each HTTP request re-executes the Liquid render in full, with no memoization of prior `all_products` scans across requests. At 100 requests per second, a single product page with 5 `all_products` lookups in a loop generates 100 full catalog scans per second on the edge node. Shopify's documented render timeout of 10–30 seconds does not protect against accumulated slow renders that individually stay under the threshold — they compound into LCP failures and degraded Apdex scores.

The correct replacement is `collections`-based product access: curate a "related products" or "cross-sell" collection in Shopify admin, then access `section.settings.collection.products` or use a metafield to store an explicit `collection` reference. For cases where a product must be resolved by handle without a collection context, Shopify's Storefront API provides O(1) indexed product lookups via GraphQL that do not execute inside the Liquid render cycle. The `all_products` global exists for backward compatibility with pre-OS2.0 themes and is absent from all Shopify reference theme implementations (Dawn, Craft, Horizon) as of 2024.

---

## Detection Logic (AST)

1. **Identify `LiquidVariable` nodes with bracket-accessor syntax on `all_products`.** The AST walker targets `LiquidVariable` nodes whose `expression` is a `MemberExpression` (or equivalent bracketed access) where the base object identifier resolves to `all_products`. The bracket argument may be a string literal (`all_products['handle']`), a variable (`all_products[product_handle]`), or a chained expression (`all_products[item.handle]`).

2. **Check for `LiquidTag[assign]` context.** If the `all_products[handle]` access is the right-hand side of an `{% assign %}` tag, record the assigned variable name. Trace forward uses of that variable to understand the full scope of the violation — a single assign followed by multiple uses still represents one scan, but assigns inside a loop represent N scans.

3. **Traverse ancestors for `LiquidTag[for]` containment.** Walk the ancestor chain from the `all_products[handle]` node upward. If any ancestor is a `LiquidTagOpen` with `name === "for"`, flag as loop-amplified and report the estimated scan count based on the loop's iterable (if statically determinable from the `for` tag markup — e.g., `for handle in cross_sell_handles` where `cross_sell_handles` is a known-length array).

4. **Classify severity.** Any `all_products[handle]` access is at minimum HIGH. Any access inside a `LiquidTag[for]` ancestor is CRITICAL. Multiple `all_products` accesses in the same template without an intervening `{% paginate %}` or collection-scope guard are reported as a single CRITICAL violation with a count.

**Why regex fails here:** A pattern matching `all_products\[` would match uses inside `{% comment %}` blocks, inside `{% raw %}` sections, and inside `{% if %}` branches that are never executed (e.g., `{% if false %}`). It cannot determine whether the access site is inside a `{% for %}` loop to apply severity escalation. The AST walker can filter comment and raw nodes entirely, and can traverse parent nodes to detect the loop context, providing accurate severity classification without manual triage.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: all_products[handle] used in a cross-sell loop on a product page.

  Failure mode 1: O(n) scan per call — each all_products[handle] access
  scans the full product catalog linearly. Five handles = five full scans.
  On a 1,000-product store this is 250–1,000ms of pure lookup overhead.

  Failure mode 2: Loop placement — the lookup is inside a {% for %} loop,
  so the catalog is scanned once per iteration. Shopify's runtime does not
  cache the all_products map between iterations within the same request.

  Failure mode 3: No nil guard — if a handle in cross_sell_handles has been
  deleted or renamed, all_products[handle] returns nil. Accessing nil.title
  throws a Liquid render error that short-circuits the entire section output.

  Failure mode 4: Handle sourced from metafield string list — metafield values
  are not validated against the live product catalog, so stale handles cause
  both wasted scans and nil errors simultaneously.
{%- endcomment -%}

{%- assign cross_sell_handles = product.metafields.custom.cross_sell_handles.value -%}

<div class="cross-sell" data-section-id="{{ section.id }}">
  <h2>{{ 'products.cross_sell.heading' | t }}</h2>

  <div class="cross-sell__grid">
    {%- for handle in cross_sell_handles -%}
      {%- assign cross_product = all_products[handle] -%}

      <div class="cross-sell__item">
        <a href="{{ cross_product.url }}">
          <img
            src="{{ cross_product.featured_image | image_url: width: 300 }}"
            alt="{{ cross_product.title }}"
          >
          <p class="cross-sell__title">{{ cross_product.title }}</p>
          <p class="cross-sell__price">{{ cross_product.price | money }}</p>
        </a>
        <button data-product-id="{{ cross_product.id }}">
          {{ 'products.cross_sell.add_to_cart' | t }}
        </button>
      </div>
    {%- endfor -%}
  </div>
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace all_products[handle] loop with a curated collection resolved
  via a section setting or product metafield of type `collection_reference`.

  Change 1: Use a `collection_reference` metafield instead of a list of handles.
  Shopify resolves the collection object at the database level — no Liquid-side
  catalog scan is required. The collection.products array is pre-indexed.

  Change 2: `| limit: 4` scopes the product iteration to the display count.
  No additional filtering over the full catalog occurs.

  Change 3: Nil guard on cross_sell_collection prevents render errors when the
  metafield is not set. A feature-flag pattern is used rather than a bare access.

  Change 4: image_tag with widths generates proper srcset — each cross-sell
  image is served at the correct size per breakpoint without manual srcset markup.
{%- endcomment -%}

{%- comment -%}
  Requires a metafield definition:
  namespace: custom, key: cross_sell_collection, type: collection_reference
{%- endcomment -%}
{%- assign cross_sell_collection = product.metafields.custom.cross_sell_collection.value -%}

<div class="cross-sell" data-section-id="{{ section.id }}">
  <h2>{{ 'products.cross_sell.heading' | t }}</h2>

  {%- if cross_sell_collection != blank -%}
    {%- assign cross_sell_products = cross_sell_collection.products | limit: 4 -%}

    <div class="cross-sell__grid">
      {%- for cross_product in cross_sell_products -%}
        {%- unless cross_product.id == product.id -%}
          <div class="cross-sell__item">
            <a href="{{ cross_product.url }}">
              {{-
                cross_product.featured_image
                | image_url: width: 300
                | image_tag:
                  loading: 'lazy',
                  widths: '150,300,450',
                  alt: cross_product.featured_image.alt
              -}}
              <p class="cross-sell__title">{{ cross_product.title }}</p>
              <p class="cross-sell__price">{{ cross_product.price | money }}</p>
            </a>
            <button
              class="cross-sell__add"
              data-product-id="{{ cross_product.id }}"
              data-variant-id="{{ cross_product.selected_or_first_available_variant.id }}"
            >
              {{ 'products.cross_sell.add_to_cart' | t }}
            </button>
          </div>
        {%- endunless -%}
      {%- endfor -%}
    </div>
  {%- endif -%}
</div>
```

---

## References

- [Shopify — `all_products` object (legacy)](https://shopify.dev/docs/api/liquid/objects/all_products)
- [Shopify — Product metafields: collection_reference type](https://shopify.dev/docs/apps/build/custom-data/metafields/list-of-data-types)
- [Shopify — Performance: avoid all_products](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify — Dawn theme (no all_products usage)](https://github.com/Shopify/dawn)
- [Shopify Storefront API — product query by handle](https://shopify.dev/docs/api/storefront/latest/queries/product)
