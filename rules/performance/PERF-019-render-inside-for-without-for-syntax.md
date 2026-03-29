# PERF-019 | `{% render %}` inside `{% for %}` without `for:` syntax

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Section files and templates that render product grids, collection listings, cart line items, or any repeated snippet inside a `{% for %}` loop

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Shopify's `{% render %}` tag operates with strict scope isolation: each invocation creates an independent render context, instantiates the snippet's variable scope, resolves the snippet file from the theme asset graph, and writes the output into a separate output buffer that is then merged into the parent template's output stream. This is not a function call or a template include in the traditional sense — it is a sandboxed sub-render. When `{% render 'snippet', product: product %}` is placed inside a `{% for product in collection.products %}` loop, each loop iteration performs a full snippet instantiation cycle. For a product grid displaying 24 items, this produces 24 independent render calls. For a cart drawer rendering 8 line items, this produces 8 render calls. Each render call has non-zero overhead: snippet lookup in the compiled template cache, argument binding, scope object allocation, and output buffer management.

Shopify's `{% render %}` tag provides a `for:` syntax specifically designed to eliminate this overhead: `{% render 'snippet' for collection.products as product %}`. This syntax performs a single render call and iterates the array inside the snippet's render context. The snippet receives each iteration's object as the `as`-aliased variable, and the standard `forloop` object is available within the snippet for index tracking (`forloop.index`, `forloop.first`, `forloop.last`). The single-render approach reduces N snippet instantiations to 1, eliminating N-1 rounds of scope allocation and buffer management. For a 24-product grid with a moderately complex product card snippet (image, title, price, variant picker, badges), the render-call reduction produces measurable TTFB improvements in the range of 20–80ms on Shopify's edge rendering layer. For server-side rendered sections where TTFB directly affects LCP, this compounds with other performance factors.

The `for:` syntax also enables a cleaner architectural separation: the parent template is responsible for providing the array; the snippet is responsible for rendering one item. This pattern is the recommended approach in Shopify's official render tag documentation and is used throughout the Dawn reference theme for product card and collection card rendering. The `with:` syntax (passing a single object) and the `for:` syntax (passing an array) are the two intended usage patterns for `{% render %}`; using neither and instead relying on a parent `{% for %}` loop is the anti-pattern this rule targets.

---

## Detection Logic (AST)

1. **Identify `LiquidTag[render]` nodes that are descendants of `LiquidTag[for]` nodes.** The AST walker traverses all `LiquidTag` nodes with `name === "render"`. For each, walk the ancestor chain. If any ancestor is a `LiquidTagOpen` node with `name === "for"`, the render call is inside a for loop and is a candidate for this rule.

2. **Inspect the `render` tag's markup for presence of `for:` syntax.** Parse the `LiquidTag[render]` node's markup string (or its structured argument list if the AST parser has resolved it). If the markup contains the `for:` keyword argument — indicating that the `for:` iteration syntax is already in use — the node is compliant; skip it. Only `render` calls using the simple `'snippet', variable: value` argument form inside a `for` loop are violations.

3. **Inspect the `for` ancestor's iterable expression.** From the `LiquidTag[for]` ancestor, extract the iterable object (e.g., `collection.products`, `cart.items`, `section.blocks`). This is the array that should be passed to the `for:` argument in the recommended fix. Include this information in the diagnostic message to provide an actionable fix.

4. **Classify severity by loop bound.** If the `for` tag iterates over `collection.products`, `collections`, `cart.items`, `search.results`, or any paginated array — where loop count can exceed 12 — severity is MEDIUM. If the iterable is `section.blocks` (maximum 50 blocks, typically 3–10), severity is LOW. Include the loop variable's source in the diagnostic message.

**Why regex fails here:** A regex matching `{% render %}` would identify render calls but cannot determine whether the call site is inside a `{% for %}` loop without tracking block boundaries — multi-line `{% for %}` / `{% endfor %}` pairs require stateful context that a single-pass regex cannot maintain. Additionally, detecting whether the render call already uses `for:` syntax requires parsing the render tag's argument list, which may span multiple lines with whitespace-stripped markers (`{%-`). The AST walker resolves the complete ancestor chain and the render tag's typed argument structure without line-by-line scanning.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: {% render %} called once per loop iteration inside
  a {% for %} loop over collection.products.

  Failure mode 1: 24 render calls for a 24-product grid. Each call
  allocates an independent snippet context, performs snippet file lookup,
  binds the `product` argument into the new scope, and manages a separate
  output buffer. Total overhead: ~24x per-render fixed cost.

  Failure mode 2: forloop metadata (forloop.index, forloop.first,
  forloop.last) is not available inside the snippet because render
  creates an isolated scope. The snippet cannot know its position
  in the grid without re-implementing index tracking manually.

  Failure mode 3: The parent template's for loop variable (product)
  shadows and is explicitly re-passed into the snippet as a named
  argument — boilerplate that the for: syntax eliminates.

  Failure mode 4: On collection pages with dynamic pagination reset
  to 48 products, this pattern generates 48 render calls per section
  render — compounding TTFB impact on paginated collection pages.
{%- endcomment -%}

<div class="product-grid" data-section-id="{{ section.id }}">
  {%- if collection.products_count > 0 -%}
    <ul class="product-grid__list" role="list">
      {%- for product in collection.products -%}
        {%- render 'product-card',
          product: product,
          show_vendor: section.settings.show_vendor,
          show_rating: section.settings.show_rating
        -%}
      {%- endfor -%}
    </ul>

    {%- if paginate.pages > 1 -%}
      {%- render 'pagination', paginate: paginate -%}
    {%- endif -%}
  {%- else -%}
    <p class="product-grid__empty">{{ 'sections.collection.empty' | t }}</p>
  {%- endif -%}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace {% render %} inside {% for %} with the render for: syntax.

  Change 1: `{% render 'product-card' for collection.products as product %}`
  performs a single render call that iterates the products array internally.
  The per-render overhead (scope allocation, file lookup, buffer management)
  is paid once instead of once per product.

  Change 2: forloop is available inside the snippet when using for: syntax.
  The snippet can access forloop.index, forloop.first, forloop.last for
  position-dependent styling (e.g., first-card animation class, index-based
  lazy loading) without any additional argument passing.

  Change 3: Section settings shared across all card renders (show_vendor,
  show_rating) must be accessible inside the snippet. Pass them as additional
  named arguments alongside the for: argument — they are bound once and
  available for every iteration within the snippet context.

  Change 4: The outer {% for %} block and its closing {% endfor %} are
  removed entirely — the for: syntax handles iteration inside the snippet.
  The parent template remains responsible only for providing the array.
{%- endcomment -%}

<div class="product-grid" data-section-id="{{ section.id }}">
  {%- if collection.products_count > 0 -%}
    <ul class="product-grid__list" role="list">
      {%- render 'product-card' for collection.products as product,
        show_vendor: section.settings.show_vendor,
        show_rating: section.settings.show_rating
      -%}
    </ul>

    {%- if paginate.pages > 1 -%}
      {%- render 'pagination', paginate: paginate -%}
    {%- endif -%}
  {%- else -%}
    <p class="product-grid__empty">{{ 'sections.collection.empty' | t }}</p>
  {%- endif -%}
</div>
```

---

## References

- [Shopify — `render` tag: `for` syntax](https://shopify.dev/docs/api/liquid/tags/render#render-for-array-as-variable)
- [Shopify — `render` vs `include` tag differences](https://shopify.dev/docs/api/liquid/tags/render)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify Dawn — product-card render pattern](https://github.com/Shopify/dawn)
- [Shopify — `forloop` object](https://shopify.dev/docs/api/liquid/objects/forloop)
