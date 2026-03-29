# PERF-012 | `| where` on array > 100 elements

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Any template applying `| where` to `collections.all.products`, `collection.products` on large collections, variant arrays, metafield list values, or any Liquid array with 100+ elements

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

The `| where` filter performs a linear O(n) scan of its input array, iterating over every element and comparing a named property against a target value. For arrays of 100 or fewer elements, the overhead is negligible. For arrays derived from product catalogs — `collections.all.products` (potentially 10,000+ items), a large collection's `collection.products` (up to 50 items per paginated page, but unbounded before pagination), or variant arrays on configurable products (up to 100 variants per product) — the scan cost accumulates. More critically, `| where` is not lazily evaluated: it must materialize the complete input array before scanning it. Applied to `collections.all.products`, this means the full product catalog is instantiated in the edge node's memory heap before the first comparison is made. The filter produces no output savings at the input stage — a catalog with 5,000 products and 1 matching item still requires all 5,000 product objects to be instantiated and evaluated.

When `| where` is chained with other filters on large arrays, the memory pressure compounds. `collections.all.products | where: "product_type", "Shoes" | sort: "price" | limit: 12` performs three sequential operations, each on the fully materialized output of the previous: load all products (5,000 objects), filter to type "Shoes" (allocate a filtered array of, say, 200 objects), sort the 200-element array (allocate a sorted copy), then take 12. The peak memory at step 2 is the original 5,000-object array plus the 200-object filtered copy — both are live simultaneously because Liquid does not stream or garbage-collect intermediate results during a filter chain. For metafield list values (e.g., a list of product references stored in a metafield), `| where` may also be applied to arrays of 50–200 elements; individually small, but applied inside a loop the cost multiplies.

The correct approach depends on the use case. For product type or tag filtering on a collection, Shopify's native collection URL parameters (`?filter.p.product_type=Shoes`) apply the filter at the database level before any Liquid evaluation occurs — zero product objects are instantiated for products that don't match the filter. For variant filtering (finding a specific variant by option values), native Liquid provides `product.variants` as a pre-scoped array and `product.selected_or_first_available_variant` covers the most common use case without any scanning. For metafield list filtering, the correct solution is to model the data differently — use a `collection_reference` metafield pointing to a pre-curated collection rather than a list of product references to be post-filtered.

---

## Detection Logic (AST)

1. **Identify all `LiquidFilter` nodes with `name === "where"`.** The AST walker collects every `LiquidFilter` node in the template tree where the filter name is `"where"`. These appear in `LiquidVariable` expression chains and in `LiquidTag[assign]` right-hand sides.

2. **Trace the filter's input array to determine its source size.** For each `| where` filter node, walk backward along the filter chain to the base input expression. If the base is `collections.all` (or `collections.all.products`), flag as CRITICAL — the input is the full unbounded product catalog. If the base is `collection.products` without an upstream `| limit` or `{% paginate %}` boundary, flag as HIGH — the collection may have 100+ products. If the base is a variable, trace the variable's `{% assign %}` source to determine the array origin.

3. **Check for upstream `{% paginate %}` boundary.** If the `| where` filter is inside a `{% paginate %}` block and the paginate tag's iterable is the same collection, the collection is already page-bounded (20–50 items per page). In this case, the effective input to `| where` is at most 50 items — flag as LOW severity (informational) rather than HIGH.

4. **Classify by filter chain context.** If `| where` is the first filter in a chain that also includes `| sort`, `| sort_natural`, or another `| where`, report the full chain as compounding the issue — each additional filter adds a materialized intermediate array allocation. If `| where` appears inside a `LiquidTag[for]` ancestor (nested loop), escalate to CRITICAL — see PERF-003 interaction.

**Why regex fails here:** A regex matching `\|\s*where` identifies every `| where` usage, including valid uses on small arrays (e.g., `section.blocks | where: "type", "product-card"` — always fewer than 50 blocks). The rule only applies to large arrays, and "large" requires tracing the filter's input to its array source — a task that involves following assignment chains and recognizing specific Shopify global objects (`collections.all`, `collection.products`). This chain-following requires AST traversal of assignment nodes, not text pattern matching. A regex producing a violation on `section.blocks | where:` would generate false positives for every theme using block filtering — a near-universal pattern.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: | where filter applied to collections.all.products —
  triggers a full catalog load before any filtering occurs.
  Context: A "shop by category" section on the homepage showing
  product previews for each product type.

  Failure mode 1: collections.all.products instantiates the full product
  catalog into memory before | where begins scanning. On a 2,000-product
  store, this is 2,000 product objects allocated for a filter that may
  match only 20.

  Failure mode 2: The filter runs inside a {% for %} loop over product types.
  Each iteration performs a full catalog load and scan — see PERF-003.
  Five product types = five full catalog instantiations.

  Failure mode 3: Chained | sort after | where allocates a second array
  (the sorted subset) while the original filtered array is still in scope.
  Memory peak: full catalog + filtered subset + sorted copy, all live.

  Failure mode 4: | limit: 3 does not prevent the upstream load — it only
  truncates the output. The 2,000-product scan still happens in full.
{%- endcomment -%}

{%- assign product_types = 'Shoes,Bags,Accessories,Outerwear,Basics' | split: ',' -%}

<section class="shop-by-category" data-section-id="{{ section.id }}">
  <h2>{{ section.settings.heading }}</h2>

  <div class="shop-by-category__grid">
    {%- for type in product_types -%}
      {%- comment -%}
        Full catalog loaded on EVERY iteration of this loop.
        Five iterations = five full catalog instantiations.
      {%- endcomment -%}
      {%- assign type_products = collections.all.products
        | where: "product_type", type
        | sort: "published_at"
        | limit: 3
      -%}

      {%- if type_products.size > 0 -%}
        <div class="category-preview">
          <h3 class="category-preview__title">{{ type }}</h3>
          <div class="category-preview__products">
            {%- for product in type_products -%}
              <a href="{{ product.url }}" class="category-preview__item">
                {{
                  product.featured_image
                  | image_url: width: 300
                  | image_tag: loading: 'lazy', alt: product.featured_image.alt
                }}
                <span>{{ product.title }}</span>
              </a>
            {%- endfor -%}
          </div>
          <a href="/collections/{{ type | handleize }}" class="category-preview__link">
            {{ 'sections.shop_by_category.view_all' | t }}
          </a>
        </div>
      {%- endif -%}
    {%- endfor -%}
  </div>
</section>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace collections.all | where with collection-per-type resolution.
  Each product type is curated as its own collection in Shopify admin.
  collections[handle] performs a single indexed collection lookup — no
  catalog scan, no full product instantiation.

  Change 1: Merchant curates one collection per product type. The section
  uses a list of collection handles (configurable via section blocks) rather
  than a hardcoded product_type filter over the full catalog.

  Change 2: collections[handle].products returns only the products in that
  specific collection — database-filtered before Liquid evaluation.
  `| limit: 3` is applied to this already-scoped array.

  Change 3: If the shop structure does not support per-type collections,
  use Shopify's collection URL filter parameter (?filter.p.product_type=Shoes)
  to redirect users to a filtered collection URL — filtering at the DB level.

  Change 4: section.blocks replaces the hardcoded product_types array,
  enabling merchants to configure which categories appear and in what order.
{%- endcomment -%}

<section class="shop-by-category" data-section-id="{{ section.id }}">
  <h2>{{ section.settings.heading }}</h2>

  <div class="shop-by-category__grid">
    {%- for block in section.blocks -%}
      {%- comment -%}
        collections[handle] — O(1) indexed collection lookup.
        No product catalog scan. Only products in this collection are loaded.
      {%- endcomment -%}
      {%- assign category_collection = collections[block.settings.collection_handle] -%}

      {%- if category_collection != blank and category_collection.products_count > 0 -%}
        {%- assign preview_products = category_collection.products | limit: 3 -%}

        <div class="category-preview" {{ block.shopify_attributes }}>
          <h3 class="category-preview__title">
            {{ block.settings.title | default: category_collection.title }}
          </h3>
          <div class="category-preview__products">
            {%- for product in preview_products -%}
              <a href="{{ product.url }}" class="category-preview__item">
                {{-
                  product.featured_image
                  | image_url: width: 300
                  | image_tag:
                    loading: 'lazy',
                    widths: '150,300,450',
                    width: product.featured_image.width,
                    height: product.featured_image.height,
                    alt: product.featured_image.alt
                -}}
                <span>{{ product.title }}</span>
              </a>
            {%- endfor -%}
          </div>
          <a href="{{ category_collection.url }}" class="category-preview__link">
            {{ 'sections.shop_by_category.view_all' | t }}
          </a>
        </div>
      {%- endif -%}
    {%- endfor -%}
  </div>
</section>
```

---

## References

- [Shopify — `where` filter](https://shopify.dev/docs/api/liquid/filters/where)
- [Shopify — Collection filtering via URL parameters](https://shopify.dev/docs/storefronts/themes/navigation-search/filtering/storefront-filtering)
- [Shopify — `collections` object: handle-based access](https://shopify.dev/docs/api/liquid/objects/collections)
- [Shopify — Collection object: products](https://shopify.dev/docs/api/liquid/objects/collection#collection-products)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify — Section blocks schema](https://shopify.dev/docs/storefronts/themes/architecture/sections/section-schema#blocks)
