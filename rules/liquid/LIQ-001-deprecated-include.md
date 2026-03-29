# LIQ-001 | `{% include %}` deprecated — replace with `{% render %}`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** All templates and sections that reference snippet files via `{% include %}`

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{% include %}` was the original Shopify snippet inclusion tag, introduced in Liquid 1.0. It operates by sharing the full parent template scope with the included partial — every variable defined in the calling template is readable and writable inside the snippet. This design makes it impossible for Shopify's edge rendering layer to cache snippet output independently, because the cache key would need to encode the entire parent variable state on every request. `{% render %}`, introduced with Shopify Online Store 2.0 in 2021, enforces strict scope isolation: the snippet receives only the variables explicitly passed as arguments. This allows Shopify's CDN to cache snippet output keyed solely to those arguments, decoupling snippet performance from full-page render cycles.

The scope-sharing behavior of `{% include %}` has concrete memory implications. On each render, Shopify's Liquid runtime must serialize the full parent variable table and pass it into the snippet's execution context. For templates with deep object graphs — `product.variants`, `collection.products`, metafield namespaces — this serialization copies the entire object tree into the snippet's heap allocation. A theme with 12 `{% include %}` calls in `theme.liquid` and a product page that loads a collection with 50 products will serialize the product and collection objects 12 times per request. `{% render %}` passes only references to the explicitly declared arguments, eliminating the redundant allocation.

`{% include %}` was formally deprecated in Shopify's Theme Check 1.x as a `DeprecatedTag` violation. The Shopify Theme Store automated review pipeline runs Theme Check against all submitted themes; any theme emitting a `DeprecatedTag` violation for `{% include %}` is rejected without human review. As of 2023, Shopify has also removed `{% include %}` from its official Dawn theme reference implementation. Themes still using `{% include %}` in production are running code that Shopify has explicitly marked for removal in future Liquid runtime versions.

---

## Detection Logic (AST)

1. **Identify `LiquidTag` nodes where `name === "include"`.** The AST walker traverses all `LiquidTag` nodes in the parsed template tree. The `name` property of a `LiquidTag` node contains the tag identifier string. Any node where `name` equals `"include"` (case-sensitive, as Liquid tag names are lowercase-only) is a candidate violation. No attribute filtering is required — there is no valid modern use of `{% include %}`.

2. **Extract the snippet name from the tag's markup.** The `LiquidTag.markup` string contains the first argument to `{% include %}` — the snippet filename (without `.liquid` extension). This is extracted to generate the replacement `{% render %}` suggestion with the correct filename preserved.

3. **Inspect passed arguments for scope-reliant patterns.** If `LiquidTag.markup` contains no explicit variable arguments (bare `{% include 'snippet' %}`) versus explicit arguments (`{% include 'snippet', var: value %}`), this informs the fix suggestion. Bare includes are most likely relying on implicit parent scope access — flagged as higher remediation complexity.

4. **Classify by nesting context.** If the `LiquidTag[include]` node's parent chain contains a `LiquidTag[name=for]` node, severity is elevated: scope serialization occurs on every loop iteration, multiplying the memory impact by the array length.

**Why regex fails here:** A regex matching `{%\s*include\s+` would produce false positives on commented-out code inside `{% comment %}` blocks. It would also fail to distinguish bare includes (scope-dependent, harder migration) from parameterized includes. The AST walker can inspect the `LiquidTag` node's parent to determine whether it is a child of a `LiquidTag[comment]` node, excluding commented code from violation reporting entirely.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: {% include %} used throughout a product template.

  Failure mode 1: Implicit scope access — the product-badge snippet reads
  `product.metafields.custom.sale_badge` without it being passed explicitly.
  This silently breaks if the snippet is ever called from a context where
  `product` is named differently or not present.

  Failure mode 2: Loop amplification — {% include %} inside {% for %} serializes
  the full parent scope (product, collection, shop, request) on every iteration.
  For 48 collection products, this is 48 full-scope serializations per request.

  Failure mode 3: Theme Store rejection — DeprecatedTag violation causes
  automated rejection. No human review is triggered.
{%- endcomment -%}

{% assign featured = collection.products | first %}

<div class="product-grid">
  {%- comment -%}
    This include reads `featured` from parent scope implicitly.
    The snippet has no way to declare its dependencies.
  {%- endcomment -%}
  {% include 'product-featured-card' %}

  <div class="product-grid__items">
    {% for product in collection.products %}
      {%- comment -%}
        Full parent scope (collection, featured, shop, request, template)
        is serialized into each snippet call. 48 products = 48 serializations.
      {%- endcomment -%}
      {% include 'product-card', show_vendor: true %}

      {% if product.available == false %}
        {% include 'product-badge', badge_type: 'sold-out' %}
      {% endif %}
    {% endfor %}
  </div>

  {% include 'pagination' %}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace all {% include %} with {% render %}.

  Change 1: Pass `featured` explicitly to 'product-featured-card'.
  The snippet now declares its dependency. It cannot accidentally read or
  mutate any other parent-scope variable.

  Change 2: Use {% render 'product-card' for collection.products as product %}.
  This is render's native loop syntax — Shopify optimizes this into a single
  snippet context with iteration, avoiding per-item scope construction overhead.

  Change 3: Pass all required variables as named arguments.
  If a variable is not passed, the snippet receives nil — a loud failure
  during development rather than a silent scope leak in production.
{%- endcomment -%}

{% assign featured = collection.products | first %}

<div class="product-grid">
  {%- comment -%}
    `featured` passed explicitly — snippet scope is fully declared at call site.
  {%- endcomment -%}
  {% render 'product-featured-card', product: featured %}

  <div class="product-grid__items">
    {%- comment -%}
      render's `for` syntax iterates without re-entering parent scope per item.
      `show_vendor` passed as a constant argument — shared across all iterations.
    {%- endcomment -%}
    {% render 'product-card' for collection.products as product,
      show_vendor: true
    %}
  </div>

  {%- comment -%}
    Pagination rendered with explicit paginate object reference.
    The snippet cannot accidentally mutate paginate in parent scope.
  {%- endcomment -%}
  {% render 'pagination', paginate: paginate %}
</div>
```

---

## References

- [Shopify — Render tag documentation](https://shopify.dev/docs/api/liquid/tags/render)
- [Shopify Theme Check — DeprecatedTag rule](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/deprecated-tag)
- [Shopify — Migrating from `include` to `render`](https://shopify.dev/docs/storefronts/themes/architecture/snippets)
- [Shopify Dawn — Reference theme (no include usage)](https://github.com/Shopify/dawn)
