# LIQ-016 | Nested `{% unless %}` creating cognitive confusion

**Category:** Merchant Logic

**Severity:** 🟢 LOW

**Affects:** All templates; particularly logic-heavy section files, cart templates, and conditional display blocks

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`{% unless condition %}` is syntactic sugar for `{% if condition == false %}`. While a single top-level `{% unless %}` is readable and expressively clear, nesting `{% unless %}` inside another `{% unless %}` block creates compounded double-negative logic that imposes significant cognitive load during code review and maintenance. To evaluate `{% unless A %}{% unless B %}...`, a developer must mentally expand both negations simultaneously: "if not A, and also if not B, then...". This is equivalent to `{% if A == false and B == false %}`, which is clearer in one glance. The cognitive cost compounds further when either condition is itself a negated expression using `!=` or when the `{% unless %}` block contains `{% elsif %}` branches.

The maintenance risk is concrete. When a developer modifies the outer condition from a negated form to a positive form during a feature change, they must correctly invert the inner condition as well — a two-site change that is easy to miss and produces no runtime error when missed. The resulting logic inversion silently changes behavior: content that should display is hidden, or content that should be hidden is displayed. Syphio flags nested `{% unless %}` (an `{% unless %}` `LiquidTag` whose parent chain in the Liquid AST contains another `{% unless %}` `LiquidTag`) as a maintainability violation regardless of depth. Theme Check does not currently flag this pattern; it is enforced only by Syphio and equivalent custom linters.

The correct refactor is to collapse nested `{% unless %}` blocks into a single `{% if %}` condition using explicit `and` / `or` operators and positive or negated terms as appropriate. This produces a single decision point that can be read and validated without mental expansion of double negatives. Where the logic genuinely requires two separate guards (e.g., for readability or comment annotation), the inner block should be written as `{% if not_condition_b %}` — but using `{% if %}` with a positive or explicitly negated condition, not `{% unless %}`.

---

## Detection Logic (AST)

1. **Identify all `LiquidTag` nodes with tag name `unless`.** Walk the AST and collect every `unless` tag node.
2. **Traverse the ancestor chain for each `unless` node.** For each collected `unless` tag, walk up the AST parent chain through `LiquidTag`, `LiquidBranch`, and `LiquidRawTag` nodes.
3. **Detect a containing `unless` ancestor.** If any ancestor node in the parent chain is also a `LiquidTag` with `tag.name === 'unless'`, a nesting violation exists.
4. **Classify by depth.** A single level of nesting (one `unless` inside another) is a LOW violation. Two or more levels of nesting (an `unless` inside an `unless` inside an `unless`) should be escalated to MEDIUM severity, as the logic is practically unverifiable without execution.
5. **Report both the inner and outer `unless` positions.** Include line numbers for both tags in the diagnostic to allow the developer to locate the full nesting structure in context.

**Why regex fails here:** A regex can identify lines containing `{% unless %}` and attempt to track nesting depth by counting open and close tags sequentially. However, regex cannot correctly handle `{% unless %}` blocks that contain intervening `{% comment %}`, `{% raw %}`, or nested `{% if %}` blocks that themselves contain `{% unless %}` — the pairing of open and close tags requires a stack-based parser that respects block boundaries. AST traversal operates on already-parsed, correctly-nested node trees where parent-child relationships are structurally guaranteed, making ancestor detection O(depth) and structurally correct.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Nested unless blocks create compounding double-negative logic.
  To understand what renders inside the innermost block, the reader must parse:
  "unless the cart is empty, unless the customer is not logged in, unless
  discounts are not available..." — three simultaneous negations. A single
  logic error in any condition silently inverts the display without a runtime error.
{%- endcomment -%}

{%- unless cart.item_count == 0 -%}
  <div class="cart-summary">
    <p>You have {{ cart.item_count }} item(s) in your cart.</p>

    {%- unless customer == nil -%}
      <p class="customer-greeting">Welcome back, {{ customer.first_name }}.</p>

      {{- /* This double-negative is nearly unreadable in a code review */ -}}
      {%- unless customer.tags contains 'no-discount' -%}
        {%- for discount in cart.cart_level_discount_applications -%}
          <p class="discount-line">
            {{ discount.title }}: -{{ discount.total_allocated_amount | money }}
          </p>
        {%- endfor -%}
      {%- endunless -%}

    {%- endunless -%}
  </div>
{%- endunless -%}
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Collapse nested unless blocks into explicit if conditions with positive
  or clearly negated terms. Each condition is now a single positive assertion:
  "if cart has items", "if customer is logged in", "if customer is eligible".
  The logic can be read top-to-bottom without mental double-negative expansion.
  Each guard is independently reviewable and independently modifiable.
{%- endcomment -%}

{%- if cart.item_count > 0 -%}
  <div class="cart-summary">
    <p>You have {{ cart.item_count }} item(s) in your cart.</p>

    {%- if customer -%}
      <p class="customer-greeting">Welcome back, {{ customer.first_name }}.</p>

      {{- /* Positive condition — readable in isolation during code review */ -}}
      {%- assign customer_discount_eligible = true -%}
      {%- if customer.tags contains 'no-discount' -%}
        {%- assign customer_discount_eligible = false -%}
      {%- endif -%}

      {%- if customer_discount_eligible -%}
        {%- for discount in cart.cart_level_discount_applications -%}
          <p class="discount-line">
            {{ discount.title }}: -{{ discount.total_allocated_amount | money }}
          </p>
        {%- endfor -%}
      {%- endif -%}

    {%- endif -%}
  </div>
{%- endif -%}
```

---

## References

- [Shopify Liquid: unless tag](https://shopify.dev/docs/api/liquid/tags/unless)
- [Shopify Liquid: if tag](https://shopify.dev/docs/api/liquid/tags/if)
- [Shopify Theme Development Best Practices](https://shopify.dev/docs/themes/best-practices)
