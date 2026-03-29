# PERF-001 | `collections.all` outside paginated context

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** Homepage sections, collection list templates, any template referencing `collections.all` outside a `{% paginate %}` block

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`collections.all` is a Shopify global object that exposes every product in the store as a single flat iterable. It is not a database cursor or a lazy stream — when the Liquid runtime evaluates `collections.all`, Shopify's edge rendering layer immediately instantiates every product object in the catalog, including all associated variants, images, options, and metafields. Outside a `{% paginate %}` block, there is no page-size cap applied. For a store with 1,000 products, each with 5 variants and 10 metafields, a single unpaginated reference to `collections.all` can allocate 50–200MB on the Shopify edge node's per-request memory heap. This allocation happens synchronously, blocking the render thread for the duration of the object instantiation.

Shopify enforces a server-side Liquid render timeout of 10–30 seconds per request. Unpaginated `collections.all` on catalogs above 500 products reliably triggers this timeout, causing the storefront to return a 500 error for every visitor simultaneously — including during promotional events when traffic spikes compound the memory pressure. The `{% paginate %}` tag, when wrapping a `collections.all` iteration, applies a server-side page cursor that limits instantiation to the current page's product window (default 20 items, maximum 50). Without it, the full catalog load is unavoidable regardless of how many products the `{% for %}` loop actually renders.

The pattern appears most frequently in homepage "featured products" sections built by developers who reach for `collections.all` as a convenience rather than connecting the section to a specific curated collection via `section.settings.collection`. In Shopify's documented Theme Store rejection history, unpaginated `collections.all` in a homepage section is cited as a cause of automated performance rejection. The correct remediation is to replace `collections.all` with `section.settings.collection` (merchant-configurable) or a hardcoded specific collection handle, and to apply `{% paginate %}` whenever iteration over a large, unbounded collection is required.

---

## Detection Logic (AST)

1. **Identify `LiquidVariable` and `LiquidTag` nodes that reference `collections.all`.** The AST walker traverses all `LiquidVariable` nodes (output expressions `{{ }}`) and `LiquidTag` nodes (tag expressions `{% %}`). In each, inspect the `LiquidVariable.expression` or the `LiquidTag.markup` for a member-access chain where the base object is `collections` and the member is `all` — specifically, an `AssignmentExpression`, `MemberExpression`, or raw markup containing `collections.all`.

2. **Traverse ancestors to locate a `LiquidTag[paginate]` boundary.** For each candidate node, walk the AST ancestor chain. If any ancestor is a `LiquidTagOpen` node with `name === "paginate"`, the reference is within a paginated context and is not a violation. The paginate tag enforces a server-side page cursor that caps the instantiation window.

3. **Flag the reference if no `paginate` ancestor exists.** Any `collections.all` reference whose ancestor chain contains no `LiquidTag[paginate]` node is a violation regardless of whether the reference is inside a `{% for %}` loop, an `{% assign %}`, or a direct output `{{ collections.all.size }}` — all forms trigger the full catalog load.

4. **Classify severity by loop context.** If the `collections.all` reference is inside a `LiquidTag[for]` ancestor that is itself not inside `LiquidTag[paginate]`, severity remains CRITICAL. If the reference is a standalone property access (`collections.all.size`, `collections.all | map: "title"`), severity is also CRITICAL — these still trigger full instantiation.

**Why regex fails here:** A pattern matching `collections\.all` would flag valid uses inside `{% paginate collections.all by 24 %}` blocks, generating false positives on correctly paginated code. The regex has no mechanism to check whether the match site is a descendant of a paginate tag boundary — that requires parent-chain traversal. The AST walker can follow the ancestor chain from any node upward to detect the `LiquidTagOpen[paginate]` boundary with exact scope awareness.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: collections.all referenced outside any {% paginate %} block
  on a homepage featured section.

  Failure mode 1: Full catalog instantiation — on a store with 1,000+ products,
  this single assign statement triggers 50–200MB of heap allocation before any
  template output is produced. The edge node has no page cursor to apply.

  Failure mode 2: Chained filters on the full array — `| where` and `| limit`
  run AFTER the full catalog is loaded into memory. `limit` does not prevent
  the upstream load; it only truncates the output array.

  Failure mode 3: Cascading 500 — when the render timeout fires, every visitor
  receives a 500 simultaneously. High-traffic flash sales are the most common
  trigger. The storefront goes dark for all users, not just heavy sessions.

  Failure mode 4: Developer intent mismatch — the intent is to show 4 products,
  but 10,000 are loaded to find them. The correct fix is a curated collection.
{%- endcomment -%}

{%- assign featured_products = collections.all.products
  | where: "product_type", "Featured"
  | limit: 4
-%}

<section class="featured-products" data-section-id="{{ section.id }}">
  <h2 class="featured-products__heading">{{ section.settings.title }}</h2>

  <div class="featured-products__grid">
    {%- for product in featured_products -%}
      <div class="product-card">
        <a href="{{ product.url }}">
          {%- if product.featured_image -%}
            <img
              src="{{ product.featured_image | image_url: width: 400 }}"
              alt="{{ product.featured_image.alt | escape }}"
            >
          {%- endif -%}
          <h3>{{ product.title }}</h3>
          <span>{{ product.price | money }}</span>
        </a>
      </div>
    {%- endfor -%}
  </div>
</section>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace collections.all with a merchant-configurable section setting
  that resolves to a specific collection handle.

  Change 1: section.settings.collection is a `collection` type setting — Shopify
  resolves this to a single Collection object with no catalog-wide scan. Only
  the products belonging to that collection are loaded.

  Change 2: `| limit: 4` is applied to collection.products, which is already
  scoped to the specific collection. No full-catalog load occurs.

  Change 3: A nil guard on the collection setting prevents rendering errors
  when a merchant has not yet configured the section.

  Change 4: When full catalog iteration IS genuinely required (e.g., a sitemap
  section), wrap in {% paginate collections.all by 20 %} to apply the server-side
  page cursor and limit per-request instantiation to 20 product objects.
{%- endcomment -%}

{%- assign featured_collection = section.settings.collection -%}

<section class="featured-products" data-section-id="{{ section.id }}">
  <h2 class="featured-products__heading">{{ section.settings.title }}</h2>

  {%- if featured_collection != blank -%}
    {%- assign featured_products = featured_collection.products | limit: 4 -%}

    <div class="featured-products__grid">
      {%- for product in featured_products -%}
        <div class="product-card">
          <a href="{{ product.url }}">
            {%- if product.featured_image -%}
              {{
                product.featured_image
                | image_url: width: 400
                | image_tag:
                  loading: 'lazy',
                  widths: '200,400,600',
                  alt: product.featured_image.alt
              }}
            {%- endif -%}
            <h3>{{ product.title }}</h3>
            <span>{{ product.price | money }}</span>
          </a>
        </div>
      {%- endfor -%}
    </div>
  {%- else -%}
    {%- comment -%}Placeholder shown in theme editor before collection is selected.{%- endcomment -%}
    <p class="featured-products__placeholder">{{ 'sections.featured_products.no_collection' | t }}</p>
  {%- endif -%}
</section>
```

---

## References

- [Shopify — `collections` global object](https://shopify.dev/docs/api/liquid/objects/collections)
- [Shopify — `paginate` tag](https://shopify.dev/docs/api/liquid/tags/paginate)
- [Shopify — Section schema: collection setting type](https://shopify.dev/docs/storefronts/themes/architecture/sections/section-schema#collection)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify Theme Check — UnusedAssign / performance linting](https://shopify.dev/docs/storefronts/themes/tools/theme-check)
