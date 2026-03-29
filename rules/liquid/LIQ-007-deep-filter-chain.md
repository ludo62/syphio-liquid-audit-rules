# LIQ-007 | Filter chain > 6 levels deep

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Any `{{ }}` output expression or `{% assign %}` right-hand side with more than 6 chained `LiquidFilter` nodes

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

Each filter in a Liquid filter chain creates a new intermediate value on the Liquid memory heap. The filter receives the output of the previous filter as its input, processes it, and returns a new string or object. The previous intermediate value becomes eligible for garbage collection, but in Shopify's synchronous single-threaded Liquid runtime, garbage collection does not occur mid-expression — all intermediate values for a given filter chain remain allocated simultaneously until the expression completes. A chain of 8 filters on a string therefore holds 8 string objects in memory at the same time during evaluation, even though only the final output is used.

The computational cost compounds inside `{% for %}` loops. A 7-filter chain applied inside a loop over 50 products creates 350 intermediate string allocations per request. If any of those filters performs a non-trivial operation — `| replace:`, `| remove:`, `| split:` — each one runs string search or tokenization on the intermediate value, not on the original. This is because Liquid does not perform filter fusion or lazy evaluation. Each filter is a discrete function call with a discrete input and output, evaluated eagerly in order. There is no compiler optimization pass that collapses `| downcase | replace: ' ', '-'` into a single regex substitution — they execute as two separate function calls.

Beyond performance: deeply chained filters have a debugging problem that is particularly acute in Liquid's type system. If any filter in the chain receives an unexpected input type — most commonly `nil` from an unset metafield or a product attribute that hasn't been populated — every subsequent filter in the chain silently receives `nil` as input and returns `nil` or an empty string as output. A 7-filter chain where the third filter receives nil produces blank output with no indication of where the chain broke. Breaking long chains into intermediate `{% assign %}` variables makes each step inspectable and makes nil propagation visible — the developer can insert `{% if intermediate_var != blank %}` guards between steps.

---

## Detection Logic (AST)

1. **Identify `LiquidVariableOutput` nodes and `LiquidTag[name=assign]` right-hand expressions.** The AST walker traverses all `LiquidVariableOutput` nodes (output expressions `{{ }}`) and all `LiquidTag[name=assign]` nodes where the right-hand side is a `LiquidVariable` with a filter chain. Both are sites where filter chains are evaluated.

2. **Count `LiquidFilter` child nodes on each expression.** Each `LiquidVariable` node exposes a `filters` array containing all `LiquidFilter` nodes in the chain in order. The walker reads `LiquidVariable.filters.length`. If `filters.length > 6`, the expression is a violation.

3. **Inspect for loop parent context.** The walker checks whether the `LiquidVariableOutput` or `LiquidTag[assign]` node's ancestor chain contains a `LiquidTag[name=for]` node. If so, severity is upgraded from MEDIUM to HIGH — the allocation multiplier effect applies.

4. **Classify chain composition for fix guidance.** The walker inspects whether the filter chain mixes type-transformation filters (`| split`, `| first`, `| last`, `| map`) with string-formatting filters (`| upcase`, `| replace`, `| strip`). Mixed chains are candidates for split into two `{% assign %}` intermediates at the type boundary. Pure string chains are candidates for reduction by combining logically equivalent filters.

**Why regex fails here:** A regex counting pipe `|` characters within `{{ }}` delimiters would miscount filters that include pipe characters in their string arguments — for example `| replace: '|', '-'` (replacing a pipe character) would be counted as an additional filter by a naive regex. The AST filter chain is a parsed array of `LiquidFilter` nodes with argument lists, not a pipe-delimited string, making the count unambiguous regardless of filter argument content.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Filter chains exceeding 6 levels, including one inside a loop.

  Failure mode 1: 8-filter chain for URL slug — holds 8 intermediate strings
  in memory simultaneously. Runs string search 5 times on increasingly
  processed intermediates. Nil propagation from filter 3 onward if input
  is blank produces silent empty output.

  Failure mode 2: 7-filter chain inside a for loop over 48 products.
  48 iterations × 7 allocations = 336 intermediate string objects per request.
  If product.title is nil for any product (draft/hidden), the entire chain
  returns blank with no warning.

  Failure mode 3: Debugging is impossible — if any filter returns nil,
  the final output is blank and there is no way to identify which step failed
  without decomposing the chain manually.
{%- endcomment -%}

{%- comment -%} 8-filter chain: excessive allocations, nil-unsafe -%}
{% assign product_slug = product.title
  | downcase
  | strip
  | replace: ' ', '-'
  | remove: "'"
  | remove: '.'
  | remove: ','
  | truncate: 60, ''
  | prepend: '/products/'
%}

<div class="product-grid">
  {% for product in collection.products %}
    {%- comment -%}
      7-filter chain inside a loop: 48 × 7 = 336 allocations per request.
      Nil propagation: if product.vendor is blank, | append and beyond return blank.
    {%- endcomment -%}
    {% assign display_label = product.vendor
      | downcase
      | capitalize
      | append: ' — '
      | append: product.title
      | truncate: 80
      | strip
      | escape
    %}
    <div class="product-card">
      <p>{{ display_label }}</p>
      <span>{{ product.price | money }}</span>
    </div>
  {% endfor %}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Decompose chains longer than 6 filters into named intermediates.

  Change 1: product_slug split into two assigns at the logical boundary.
  First assign: normalize the string (downcase, strip, character removal).
  Second assign: transform the normalized value (truncate, prepend).
  Each assign is nil-checkable independently. Maximum 5 filters per chain.

  Change 2: display_label decomposed into vendor_normalized + display_label.
  vendor_normalized guards against nil vendor before string concatenation.
  Loop allocation reduced: 2 assigns × 48 iterations, but each assign has
  ≤ 3 filters — 6 total allocations per iteration vs. 7 previously.
  The nil guard also prevents the blank output failure mode.
{%- endcomment -%}

{%- comment -%}
  Step 1: normalize title characters — 5 filters, within limit.
{%- endcomment -%}
{% assign product_slug_raw = product.title
  | downcase
  | strip
  | replace: ' ', '-'
  | remove: "'"
  | remove: '.'
%}

{%- comment -%}
  Step 2: shape the normalized value — 3 filters, well within limit.
  If product_slug_raw is blank, the guard below prevents a broken URL.
{%- endcomment -%}
{% if product_slug_raw != blank %}
  {% assign product_slug = product_slug_raw | remove: ',' | truncate: 60, '' | prepend: '/products/' %}
{% endif %}

<div class="product-grid">
  {% for product in collection.products %}
    {%- comment -%}
      Guard against nil vendor before building the display label.
      Nil propagation is now explicit and handleable.
    {%- endcomment -%}
    {% if product.vendor != blank %}
      {% assign vendor_normalized = product.vendor | downcase | capitalize %}
    {% else %}
      {% assign vendor_normalized = '' %}
    {% endif %}

    {%- comment -%}
      display_label: 3 filters, nil-safe, within limit.
    {%- endcomment -%}
    {% assign display_label = vendor_normalized
      | append: ' — '
      | append: product.title
      | truncate: 80
    %}

    <div class="product-card">
      <p>{{ display_label | strip | escape }}</p>
      <span>{{ product.price | money }}</span>
    </div>
  {% endfor %}
</div>
```

---

## References

- [Shopify Liquid — Filters reference](https://shopify.dev/docs/api/liquid/filters)
- [Shopify — Performance best practices for Liquid](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
- [Shopify Liquid — `assign` tag (intermediate variables)](https://shopify.dev/docs/api/liquid/tags/assign)
- [Shopify — Liquid nil handling](https://shopify.dev/docs/api/liquid/basics#nil)
