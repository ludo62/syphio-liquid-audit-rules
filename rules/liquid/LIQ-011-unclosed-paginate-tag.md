# LIQ-011 | `{% paginate %}` without `{% endpaginate %}`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Collection templates, search templates, and any template using `{% paginate %}` to limit array output

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{% paginate %}` opens a pagination context that configures how Shopify's Liquid runtime loads collection or array data for the current request. Its primary function is to impose a per-page item limit on the data loaded from Shopify's database layer into the Liquid memory heap. When `{% paginate collection.products by 24 %}` is declared, Shopify's rendering pipeline loads only 24 product objects into memory for the current page, regardless of how many products exist in the collection. The `{% endpaginate %}` closing tag signals the end of this context scope. Without the closing tag, Shopify's runtime behavior at the data loading boundary is undefined — in observed cases, the pagination limit is not applied, and the full collection is loaded into the edge node's memory heap for the request.

For stores with large catalogs — 500 to 50,000+ products in a collection — a missing `{% endpaginate %}` on a collection template causes each page request to allocate the full product catalog into Liquid memory. The Shopify edge node allocates product objects, variant arrays, image arrays, and metafield namespaces for every product in the collection simultaneously. At 500 products with 5 variants each, this is 2,500 variant objects, 500+ image objects, and all associated metafield data loaded per request. Shopify imposes memory limits on Liquid rendering contexts; exceeding these limits causes the page to time out or return an error response. The degradation is also asymmetric: small collections are unaffected, so the bug may go unnoticed in development but causes production failures as the catalog grows.

The `paginate` drop object — which exposes `paginate.pages`, `paginate.current_page`, `paginate.previous`, `paginate.next`, and `paginate.parts` — is only available as a populated object within the `{% paginate %}...{% endpaginate %}` block scope. Outside this block, `paginate` is nil or returns default/empty values. This means all pagination UI — "Previous", "Next" buttons, page number links, total page counts — renders as blank or broken when the block is unclosed. A missing `{% endpaginate %}` therefore produces two simultaneous failures: full catalog memory allocation, and broken pagination UI, both visible to all users of the collection page.

---

## Detection Logic (AST)

1. **Collect all `LiquidTag[name=paginate]` nodes.** The AST walker traverses the template tree and collects every `LiquidTag` where `name === "paginate"`. These are `LiquidTagOpen` nodes that begin a scoped block requiring a matching `LiquidTagClose[name=endpaginate]`.

2. **Apply stack-based open/close matching.** For each `LiquidTag[paginate]` encountered, a stack entry is pushed with the tag's source position and the paginated object expression (e.g., `collection.products`). For each `LiquidTag[endpaginate]` encountered, the stack is popped. If the stack is non-empty after full traversal of the template, each remaining entry is an unclosed `{% paginate %}` — a violation is filed for each, with source position and paginated object reported.

3. **Verify `paginate` object usage within the block scope.** Within correctly closed `{% paginate %}...{% endpaginate %}` blocks, the walker checks whether the `paginate` variable is referenced (via `LiquidVariableOutput` nodes or `LiquidTag[if]` conditions referencing `paginate.pages`). If the block contains a `{% for %}` loop over the paginated collection but no `paginate` variable usage, a secondary advisory violation is filed: the developer may have forgotten to add pagination UI controls.

4. **Detect orphaned `{% endpaginate %}` tags.** If a `LiquidTag[endpaginate]` is encountered when the stack is empty, it is an orphaned closing tag with no matching open. This is flagged as a distinct violation — it indicates a template where the opening `{% paginate %}` was removed but the closing tag was left in place, which may cause unexpected scoping behavior.

**Why regex fails here:** A regex counting `{%\s*paginate\s+` and `{%\s*endpaginate\s*%}` occurrences and comparing them cannot handle templates where both tags appear but in incorrect nesting order, or where one tag appears inside a `{% comment %}` block. The AST stack-based matcher handles ordering, nesting, and comment exclusion correctly, and provides exact source positions for both the unclosed opening tag and the location where `endpaginate` was expected.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: {% paginate %} without {% endpaginate %}.

  Failure mode 1: Pagination limit not applied — Shopify loads the full
  collection into memory. For 1,000+ products, this exceeds edge node
  memory limits and causes request timeouts or 500 errors.

  Failure mode 2: paginate object is nil outside the block scope.
  paginate.pages, paginate.next, paginate.previous all return nil.
  Pagination UI (page numbers, next/prev links) renders blank.

  Failure mode 3: The for loop iterates over all products because the
  paginate block scope never closed — the `by 24` limit is not enforced
  for the for loop's data source.
{%- endcomment -%}

{% paginate collection.products by 24 %}

<div class="collection-header">
  <h1>{{ collection.title }}</h1>
  <p>{{ collection.products_count }} products</p>
</div>

<div class="product-grid">
  {% for product in collection.products %}
    <div class="product-card">
      {{
        product.featured_image
        | image_url: width: 400
        | image_tag:
          widths: '200, 400',
          sizes: '(min-width: 990px) 25vw, 50vw',
          loading: 'lazy',
          alt: product.featured_image.alt
      }}
      <h2>{{ product.title }}</h2>
      <p>{{ product.price | money }}</p>
      <a href="{{ product.url }}">View product</a>
    </div>
  {% endfor %}
</div>

{%- comment -%} paginate object is nil here — navigation renders blank -%}
{% if paginate.pages > 1 %}
  <nav class="pagination" aria-label="Collection pagination">
    {{ paginate | default_pagination }}
  </nav>
{% endif %}

{%- comment -%} endpaginate missing — pagination context never closed -%}
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add {% endpaginate %} after all content that depends on the
  pagination context, including the pagination navigation UI.

  Change 1: endpaginate placed after the pagination nav — the paginate
  object is fully populated for all code within the block scope.

  Change 2: paginate.pages > 1 guard prevents rendering empty <nav> on
  single-page collections. The paginate object correctly returns page count.

  Change 3: The for loop now operates within the paginate scope — the
  `by 24` limit is enforced. Only 24 product objects are loaded per request
  regardless of collection size.
{%- endcomment -%}

{% paginate collection.products by 24 %}

<div class="collection-header">
  <h1>{{ collection.title }}</h1>
  {%- comment -%}
    paginate.items returns the total item count across all pages —
    available because we are inside the paginate block scope.
  {%- endcomment -%}
  <p>{{ paginate.items }} products</p>
</div>

<div class="product-grid">
  {%- comment -%}
    for loop inside paginate block — only 24 products loaded into memory.
    Full catalog is not allocated. Edge node memory stays within bounds.
  {%- endcomment -%}
  {% for product in collection.products %}
    <div class="product-card">
      {{
        product.featured_image
        | image_url: width: 400
        | image_tag:
          widths: '200, 400',
          sizes: '(min-width: 990px) 25vw, 50vw',
          loading: 'lazy',
          alt: product.featured_image.alt
      }}
      <h2><a href="{{ product.url }}">{{ product.title }}</a></h2>
      <p>{{ product.price | money }}</p>
    </div>
  {% endfor %}
</div>

{%- comment -%}
  paginate object is populated within block scope.
  paginate.pages, paginate.current_page, paginate.next, paginate.previous
  all return correct values here.
{%- endcomment -%}
{% if paginate.pages > 1 %}
  <nav class="pagination" aria-label="{{ 'general.pagination.label' | t }}">
    {{ paginate | default_pagination }}
  </nav>
{% endif %}

{%- comment -%}
  endpaginate: closes the pagination context. Required.
  All code below this line does not have access to the paginate object.
{%- endcomment -%}
{% endpaginate %}
```

---

## References

- [Shopify Liquid — `paginate` tag](https://shopify.dev/docs/api/liquid/tags/paginate)
- [Shopify Liquid — `paginate` object](https://shopify.dev/docs/api/liquid/objects/paginate)
- [Shopify — Collection pagination best practices](https://shopify.dev/docs/storefronts/themes/architecture/templates/collection)
- [Shopify Theme Check — MissingEndtag rule](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/missing-endtag)
