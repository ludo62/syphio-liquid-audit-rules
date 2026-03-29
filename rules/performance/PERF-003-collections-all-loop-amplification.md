# PERF-003 | `collections.all` inside a `{% for %}` loop

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** Any template or section iterating over a loop body that references `collections.all` — homepage sections, collection list pages, mega-menus, dynamic navigation

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

When `collections.all` is referenced inside the body of a `{% for %}` loop, Shopify's Liquid runtime re-evaluates the `collections.all` global on each loop iteration. Unlike a compiled language where a global reference is resolved once and cached by the optimizer, Liquid's interpreted execution model re-enters the global object resolution path every time the expression is evaluated. For a loop iterating N times over a store with M products, this produces N full catalog instantiations: N × M product objects allocated, traversed, and then released per request. For a loop over 10 navigation items on a store with 500 products, this means 5,000 product objects instantiated and discarded in a single page render.

The memory pressure compounds when filters are chained on `collections.all` inside the loop body. Each application of `| where`, `| sort`, or `| map` to `collections.all` produces a fully materialized intermediate array — a fresh allocation of the filtered or sorted result set, per iteration. A loop body containing `collections.all | where: "product_type", type_value | sort: "price"` performs three allocations per iteration: the raw catalog load, the filtered array, and the sorted copy. At 10 iterations, this is 30 array allocations of varying sizes, all heap-resident simultaneously during the render of a single request. Shopify's engineering blog has documented this "spiral of doom" pattern as a primary cause of flash-sale-triggered render timeouts, where the combination of high concurrency and per-request heap exhaustion causes cascading 500 errors across all edge nodes serving the storefront.

The correct remediation requires understanding why `collections.all` appears in the loop. In navigation mega-menu scenarios, developers use it to find a collection by handle for each nav item — this should be replaced with `collections[handle]` outside the loop, or with a precomputed `{% assign %}` of the specific collection before the loop. In "all categories" listing scenarios, `collections` (the collection list object) should be iterated directly, which does not trigger a full product catalog load. In any scenario where products from the full catalog must be filtered per loop iteration, the work should be moved out of Liquid entirely into a Storefront API call that executes before the render phase.

---

## Detection Logic (AST)

1. **Identify all `LiquidTag[for]` nodes in the template.** The AST walker collects every `LiquidTagOpen` node where `name === "for"`. For each, record the start and end boundary of the `{% for %}...{% endfor %}` block to define the loop body scope.

2. **Within each loop body scope, search for `collections.all` references.** Traverse all descendant nodes of the `LiquidTagOpen[for]` node. Inspect `LiquidVariable` expression trees and `LiquidTag` markup strings for any `MemberExpression` where the base is `collections` and the accessed member is `all`. Both dot-notation (`collections.all`) and equivalent forms must be matched.

3. **Distinguish loop-external assignments from loop-internal references.** If a `LiquidTag[assign]` node whose right-hand side contains `collections.all` appears as a sibling before the `LiquidTagOpen[for]` node (not a descendant of it), that assignment is loop-external and is a PERF-001 violation rather than PERF-003. PERF-003 applies only when the `collections.all` reference is a descendant of the `LiquidTagOpen[for]` node.

4. **Classify by filter chain complexity.** If the `collections.all` reference is the base of a `LiquidFilter` chain containing `where`, `sort`, `sort_natural`, `map`, or `uniq`, mark the violation as compounded — each filter adds an additional heap allocation per iteration. Report the filter chain length as context in the violation message.

**Why regex fails here:** A regex matching `collections\.all` inside a `for` block would require the regex to track the `{% for %}` opening and `{% endfor %}` closing tags as block boundaries — this requires stateful context that flat-pattern matching cannot provide. Nested `{% for %}` loops, `{% for %}` loops containing `{% if %}` blocks with embedded `collections.all`, and multiline filter chains that span several source lines all produce false negatives or false positives under regex. The AST walker handles all these cases through recursive descendant traversal with exact scope boundaries derived from the parsed block structure.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: collections.all accessed inside a {% for %} loop body.
  Context: a mega-menu rendering navigation categories with product counts.

  Failure mode 1: N catalog instantiations — for every nav_item in
  main_menu.links (typically 6–12 items), collections.all is fully loaded.
  On a store with 500 products, this is 6–12 × 500 product object allocations.

  Failure mode 2: Chained filters inside the loop — `| where` materializes
  a filtered copy of the full catalog per iteration. Two filters = three
  allocations per iteration (raw load + filtered + result assignment).

  Failure mode 3: Render timeout amplification — mega-menus render on every
  page of the storefront. This pattern doesn't affect one page; it degrades
  every page simultaneously, including checkout redirect pages.

  Failure mode 4: No early exit — even if the `| where` filter finds zero
  matches on the first iteration, the subsequent iterations still perform
  the full catalog scan. There is no short-circuit.
{%- endcomment -%}

<nav class="mega-menu" aria-label="{{ 'layout.navigation.main' | t }}">
  <ul class="mega-menu__list">
    {%- for link in main_menu.links -%}
      {%- assign link_handle = link.title | handleize -%}

      {%- comment -%}
        collections.all accessed fresh on every iteration of this loop.
        No caching between iterations.
      {%- endcomment -%}
      {%- assign section_products = collections.all.products
        | where: "product_type", link.title
        | limit: 3
      -%}

      <li class="mega-menu__item">
        <a href="{{ link.url }}" class="mega-menu__link">
          {{ link.title }}
          <span class="mega-menu__count">({{ section_products.size }})</span>
        </a>

        {%- if section_products.size > 0 -%}
          <div class="mega-menu__dropdown">
            {%- for product in section_products -%}
              <a href="{{ product.url }}" class="mega-menu__product">
                {{ product.title }}
              </a>
            {%- endfor -%}
          </div>
        {%- endif -%}
      </li>
    {%- endfor -%}
  </ul>
</nav>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Move all collection resolution outside the {% for %} loop.
  Use collections[handle] to resolve individual collections by handle —
  this performs a single indexed collection lookup, not a catalog scan.

  Change 1: collections[handle] resolves a specific collection by handle
  with O(1) database-indexed lookup. It does not load the product catalog.
  Only the products belonging to that collection are instantiated.

  Change 2: The lookup is used inside the loop but references the specific
  collection handle derived from the nav link — each iteration resolves a
  different collection, not the full catalog.

  Change 3: `| limit: 3` is applied to collection.products (already scoped)
  rather than to a full-catalog filtered array.

  Change 4: A nil guard prevents rendering errors when a nav link handle
  does not correspond to an existing collection.
{%- endcomment -%}

<nav class="mega-menu" aria-label="{{ 'layout.navigation.main' | t }}">
  <ul class="mega-menu__list">
    {%- for link in main_menu.links -%}
      {%- comment -%}
        Resolve collection by handle — O(1) indexed lookup, not a catalog scan.
        collections[handle] does not load the full product catalog.
      {%- endcomment -%}
      {%- assign link_handle = link.title | handleize -%}
      {%- assign nav_collection = collections[link_handle] -%}

      <li class="mega-menu__item">
        <a href="{{ link.url }}" class="mega-menu__link">
          {{ link.title }}
          {%- if nav_collection != blank -%}
            {%- comment -%}products_count is a scalar — no product objects loaded.{%- endcomment -%}
            <span class="mega-menu__count">({{ nav_collection.products_count }})</span>
          {%- endif -%}
        </a>

        {%- if nav_collection != blank and nav_collection.products_count > 0 -%}
          {%- assign preview_products = nav_collection.products | limit: 3 -%}
          <div class="mega-menu__dropdown">
            {%- for product in preview_products -%}
              <a href="{{ product.url }}" class="mega-menu__product">
                {{ product.title }}
              </a>
            {%- endfor -%}
          </div>
        {%- endif -%}
      </li>
    {%- endfor -%}
  </ul>
</nav>
```

---

## References

- [Shopify — `collections` object and handle-based access](https://shopify.dev/docs/api/liquid/objects/collections)
- [Shopify — `paginate` tag for bounded catalog iteration](https://shopify.dev/docs/api/liquid/tags/paginate)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify — Collection object: `products_count`](https://shopify.dev/docs/api/liquid/objects/collection#collection-products_count)
- [Shopify engineering — Liquid render timeouts and spiral-of-doom patterns](https://shopify.engineering)
