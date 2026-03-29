# LIQ-003 | `= 0` instead of `== 0` in `{% if %}` — always true

**Category:** Performance

**Severity:** 🔴 CRITICAL

**CWE:** CWE-697 — Incorrect Comparison

**Affects:** Any `{% if %}` or `{% elsif %}` tag that uses a single `=` operator for comparison

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

Liquid does not support assignment expressions inside `{% if %}` conditions. The single `=` character in a Liquid `{% if %}` tag is not a valid operator — Liquid's condition parser expects one of: `==`, `!=`, `>`, `>=`, `<`, `<=`, `contains`. When a developer writes `{% if product.variants.size = 0 %}`, the behavior is undefined at the parser level: some Liquid runtime versions interpret the expression as a truthy non-nil assignment result, others may raise a parse error, and still others silently ignore the operator and evaluate the entire condition as true. In all cases, the intended equality comparison is never performed.

The concrete user-facing impact depends on where the pattern appears. In sale badge logic (`{% if product.compare_at_price = 0 %}`), the badge renders for every product regardless of whether it is on sale. In inventory gating (`{% if product.variants.size = 0 %}`), the "no variants" branch renders for all products including those with variants. In cart state rendering (`{% if cart.item_count = 0 %}`), the empty cart message displays even when the cart has items. These are silent failures — no Liquid error is logged, no exception is raised, and the storefront continues to render. The bug typically survives code review because `=` and `==` are visually similar at a glance, especially in code formatted with surrounding whitespace.

The class of error maps directly to CWE-697 (Incorrect Comparison): the code performs a comparison operation using the wrong operator, causing incorrect branch evaluation. Unlike languages where `if (x = 0)` has well-defined assignment-in-condition semantics, Liquid has no such construct. The only correct equality operator in Liquid conditionals is `==`. This rule fires on any `{% if %}` or `{% elsif %}` tag where a single `=` appears between two operands.

---

## Detection Logic (AST)

1. **Identify `LiquidTag` nodes where `name === "if"` or `name === "elsif"`.** The AST walker collects all `LiquidTag` nodes with these names. Both `{% if %}` and `{% elsif %}` can contain comparison expressions and are equally susceptible to the incorrect operator pattern.

2. **Parse the condition expression within `LiquidTag.markup`.** Each `LiquidTag[if]` or `LiquidTag[elsif]` node exposes its condition as a `LiquidCondition` node in the AST. The condition node contains a `left` operand, an `operator` token, and a `right` operand. The walker inspects the `operator` property.

3. **Inspect the `operator` token value.** If `operator === "="` (single equals, as a string token), the rule fires. The valid comparison operators are `==`, `!=`, `>`, `>=`, `<`, `<=`, and `contains`. A single `=` is never a valid comparison operator in Liquid and always indicates a typo for `==`.

4. **Classify severity by operand type.** If the `right` operand is a numeric literal `0` or `1`, severity is CRITICAL — this pattern most commonly appears in inventory, cart count, or price comparisons where incorrect evaluation has direct commercial impact. If the right operand is a string literal or boolean, severity is HIGH — the comparison still fails, but the impact domain is narrower.

**Why regex fails here:** A regex matching `{%\s*if\s+.*[^!=<>]=(?!=)` would produce false positives on valid Liquid syntax such as `{% if product.handle contains "sale" %}` (no `=` involved but adjacent characters match) and false negatives when whitespace is inconsistent. The AST operator token is unambiguous: the parser tokenizes `=` and `==` as distinct tokens, making the check a direct property comparison with zero false-positive risk.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Single = used as comparison operator in multiple if/elsif branches.

  Failure mode 1: inventory gate — {% if product.variants.size = 0 %} fires
  for ALL products, not just those with zero variants. "No variants available"
  message renders on products with 10 variants.

  Failure mode 2: cart state — {% if cart.item_count = 0 %} renders the
  empty cart UI even when the customer has 5 items in the cart. Add-to-cart
  confirmation and subtotal are hidden.

  Failure mode 3: sale badge — {% if product.compare_at_price = 0 %} renders
  the sale badge for every product in the store, including full-price items.
  This is a legal liability in price advertising regulations (FTC, UK CPRs).

  Failure mode 4: elsif chain — all branches use =, so the first branch
  always evaluates as true, and subsequent elsif branches are never reached.
{%- endcomment -%}

{%- comment -%} Inventory gate — renders for ALL products -%}
{% if product.variants.size = 0 %}
  <p class="no-variants">No variants available.</p>
{% elsif product.variants.size = 1 %}
  <p class="single-variant">One option available.</p>
{% else %}
  <p class="multi-variant">Choose your option below.</p>
{% endif %}

{%- comment -%} Cart state — empty cart UI shows with items present -%}
{% if cart.item_count = 0 %}
  <div class="cart-empty">
    <p>Your cart is empty.</p>
    <a href="{{ routes.all_products_collection_url }}">Continue shopping</a>
  </div>
{% else %}
  <div class="cart-items">
    {% for item in cart.items %}
      <div class="cart-item">
        <span>{{ item.title }}</span>
        <span>{{ item.final_price | money }}</span>
      </div>
    {% endfor %}
  </div>
{% endif %}

{%- comment -%} Sale badge — renders on all products including full-price -%}
{% if product.compare_at_price = 0 %}
  <span class="badge badge--sale">Sale</span>
{% endif %}
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace every single = comparison operator with ==.

  Change 1: product.variants.size == 0 — now correctly evaluates to true
  only when the variants array is empty. The elsif chain reaches the correct
  branch for each product.

  Change 2: cart.item_count == 0 — correctly shows empty cart UI only when
  the cart is empty. Cart items render when item_count > 0.

  Change 3: product.compare_at_price > product.price — more robust check:
  verifies the compare_at_price is actually set AND higher than current price,
  which is the correct condition for a genuine sale badge.
{%- endcomment -%}

{%- comment -%} Inventory gate — correctly gates on variant count -%}
{% if product.variants.size == 0 %}
  <p class="no-variants">No variants available.</p>
{% elsif product.variants.size == 1 %}
  <p class="single-variant">One option available.</p>
{% else %}
  <p class="multi-variant">Choose your option below.</p>
{% endif %}

{%- comment -%} Cart state — correctly shows empty state only when empty -%}
{% if cart.item_count == 0 %}
  <div class="cart-empty">
    <p>Your cart is empty.</p>
    <a href="{{ routes.all_products_collection_url }}">Continue shopping</a>
  </div>
{% else %}
  <div class="cart-items">
    {% for item in cart.items %}
      <div class="cart-item">
        <span>{{ item.title }}</span>
        <span>{{ item.final_price | money }}</span>
      </div>
    {% endfor %}
  </div>
{% endif %}

{%- comment -%}
  Sale badge — correct guard: compare_at_price must exist AND exceed price.
  Avoids false positives on products with no compare_at_price set (returns nil).
{%- endcomment -%}
{% if product.compare_at_price > product.price %}
  <span class="badge badge--sale">Sale</span>
{% endif %}
```

---

## References

- [CWE-697 — Incorrect Comparison](https://cwe.mitre.org/data/definitions/697.html)
- [Shopify Liquid — Operators](https://shopify.dev/docs/api/liquid/basics#operators)
- [Shopify Liquid — `if` tag](https://shopify.dev/docs/api/liquid/tags/if)
- [Shopify Theme Check — LiquidTag condition parsing](https://shopify.dev/docs/storefronts/themes/tools/theme-check)
