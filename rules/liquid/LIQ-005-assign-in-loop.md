# LIQ-005 | `{% assign %}` inside a `{% for %}` loop

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Any template or snippet using `{% assign %}` within the body of a `{% for %}` loop

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{% assign %}` in Shopify Liquid allocates a new string or object reference on the Liquid memory heap and binds it to a variable name in the current scope. When `{% assign %}` appears inside a `{% for %}` loop, the allocation is repeated on every iteration of the loop. For a collection with 100 products, a single `{% assign %}` inside the loop creates 100 discrete heap allocations — each one triggering Liquid's variable resolution pipeline, which includes expression parsing, filter chain execution (if any filters are applied), and string interning. Shopify's Liquid runtime does not perform loop-invariant code motion or dead-code elimination; it executes every tag on every pass through the loop body, unconditionally.

The memory impact compounds when the assigned value is derived from a filter chain. A pattern such as `{% assign badge_label = 'products.badge.sale' | t | upcase %}` inside a 48-product loop executes the translation filter lookup and the `upcase` string allocation 48 times, producing 48 intermediate string objects. The translation filter (`| t`) performs a locale hash lookup on each invocation — not a cached result. For themes targeting multiple locales with large translation files, this is a measurable per-request overhead that scales linearly with collection size.

Beyond memory: the most common logical error produced by this pattern is the "only last item survives" bug. When a developer uses `{% assign %}` inside a loop intending to accumulate values across iterations — building a comma-separated string, summing prices, collecting handles — `{% assign %}` overwrites the variable on each iteration rather than appending to it. The variable's value after the loop reflects only the last iteration. This is a silent logic failure: no error is raised, and the output appears valid (it contains a real product title or price), but it is the wrong value. The correct primitive for accumulation is `{% capture %}` + string concatenation, or pre-computing the accumulated value before the loop using Liquid's `| join` filter.

---

## Detection Logic (AST)

1. **Identify all `LiquidTag[name=for]` nodes.** The AST walker collects every `LiquidTag` where `name === "for"`. These represent loop blocks whose body is a candidate for invariant-assign analysis.

2. **Traverse the loop body for `LiquidTag[name=assign]` children.** For each `LiquidTag[for]`, the walker recursively descends through all child nodes in the loop body (between the `LiquidTagOpen[for]` and `LiquidTagClose[endfor]` nodes). Any `LiquidTag[name=assign]` found at any nesting depth within this range is a candidate violation.

3. **Inspect the right-hand expression of each `LiquidTag[assign]` for loop-variable references.** The `LiquidTag[assign].markup` string is parsed to extract the right-hand expression. The walker checks whether the expression references the loop variable identifier declared in the `{% for variable in collection %}` tag. If the expression contains no reference to the loop variable, the assign is loop-invariant (the same value would be produced on every iteration) — severity is HIGH. If the expression references the loop variable, the assign is loop-dependent — severity is MEDIUM (intentional per-item computation, still potentially wasteful if the result is used in accumulation).

4. **Check for accumulation pattern.** If the loop-dependent `{% assign %}` variable name matches a variable that was assigned before the loop and is referenced after the loop, flag it as a probable accumulation error — the developer likely intended `{% capture %}` but used `{% assign %}`.

**Why regex fails here:** A regex scanning for `{%\s*assign\s+` cannot determine whether the match is inside a `{% for %}` block without tracking brace nesting state across multiple lines. Liquid templates have arbitrary levels of nested `{% for %}` and `{% if %}` blocks, and a regex cannot maintain a nesting stack. The AST provides the parent-child relationship directly: the walker checks `node.parent` ancestry for a `LiquidTag[for]` node, making the nesting depth irrelevant to the detection logic.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Multiple {% assign %} inside {% for %} loops.

  Failure mode 1: Loop-invariant assign — `sale_label` is the same on every
  iteration. It is allocated and discarded 48 times. The translation filter
  lookup runs 48 times. Result: unnecessary heap churn.

  Failure mode 2: Accumulation bug — `all_handles` is intended to collect
  all product handles into a comma-separated string. {% assign %} overwrites
  it on each iteration. After the loop, `all_handles` contains only the
  last product's handle. The developer expected all 48 handles.

  Failure mode 3: Filter chain inside loop — `display_price` runs a 3-filter
  chain (minus, money, prepend) on every iteration. Each filter creates an
  intermediate string allocation. 48 products × 3 filters = 144 allocations.
{%- endcomment -%}

{%- comment -%} Intended: collect all handles. Actual: only last handle. -%}
{% assign all_handles = '' %}

{% for product in collection.products %}
  {%- comment -%} Invariant: same value every iteration. Allocates 48 times. -%}
  {% assign sale_label = 'products.badge.sale' | t | upcase %}

  {%- comment -%} Accumulation bug: overwrites, does not append. -%}
  {% assign all_handles = product.handle %}

  {%- comment -%} Filter chain: 3 allocations × 48 iterations = 144 total. -%}
  {% assign display_price = product.compare_at_price
    | minus: product.price
    | money
    | prepend: 'Save '
  %}

  <div class="product-card">
    <span class="badge">{{ sale_label }}</span>
    <h3>{{ product.title }}</h3>
    <p>{{ display_price }}</p>
  </div>
{% endfor %}

<p>Handles: {{ all_handles }}</p>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Hoist loop-invariant assigns above the loop. Fix accumulation with
  capture + join. Keep loop-dependent assigns inside (they are acceptable).

  Change 1: sale_label hoisted above the loop — allocated once, read 48 times.
  Translation filter runs once. No per-iteration overhead.

  Change 2: all_handles built using | map and | join before the loop.
  This is the correct Liquid idiom for collecting values across an array —
  no loop required, no accumulation bug possible.

  Change 3: display_price remains inside the loop because it depends on
  the loop variable (product). This is acceptable — it is genuinely per-item.
  The filter chain is reduced to the minimum necessary.
{%- endcomment -%}

{%- comment -%}
  Invariant: hoisted above loop. Allocated once. t filter runs once.
{%- endcomment -%}
{% assign sale_label = 'products.badge.sale' | t | upcase %}

{%- comment -%}
  Correct accumulation: map extracts handles, join concatenates with separator.
  No loop needed. No overwrite bug. Executes in a single Liquid operation.
{%- endcomment -%}
{% assign all_handles = collection.products | map: 'handle' | join: ', ' %}

{% for product in collection.products %}
  {%- comment -%}
    Loop-dependent assign: depends on product (loop variable). Stays inside.
    Unavoidable per-item computation — this is acceptable usage.
  {%- endcomment -%}
  {% assign display_price = product.compare_at_price
    | minus: product.price
    | money
    | prepend: 'Save '
  %}

  <div class="product-card">
    <span class="badge">{{ sale_label }}</span>
    <h3>{{ product.title }}</h3>
    <p>{{ display_price }}</p>
  </div>
{% endfor %}

<p>Handles: {{ all_handles }}</p>
```

---

## References

- [Shopify Liquid — `assign` tag](https://shopify.dev/docs/api/liquid/tags/assign)
- [Shopify Liquid — `capture` tag](https://shopify.dev/docs/api/liquid/tags/capture)
- [Shopify Liquid — `map` filter](https://shopify.dev/docs/api/liquid/filters/map)
- [Shopify Liquid — `join` filter](https://shopify.dev/docs/api/liquid/filters/join)
- [Shopify — Performance best practices for Liquid](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
