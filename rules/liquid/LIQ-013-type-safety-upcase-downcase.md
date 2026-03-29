# LIQ-013 | `| upcase` / `| downcase` on a non-string object

**Category:** Performance

**Severity:** 🟡 MEDIUM

**Affects:** Any template applying `| upcase` or `| downcase` to numeric, boolean, array, or potentially-nil Liquid objects

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

Liquid's `upcase` and `downcase` filters are defined exclusively for string input. When applied to a non-string value, the filter coerces the input through Liquid's type system before applying the case transformation. Integer and boolean values are coerced to their string representations first (`42` becomes `"42"`, `true` becomes `"true"`, `false` becomes `"false"`), making the filter appear to work — `{{ product.variants.size | upcase }}` outputs `"24"` rather than crashing. This apparent success masks a type assumption error: the developer has applied a string operation to a numeric value, indicating they either misread the API documentation or are operating on the wrong property of the object.

The critical failure mode occurs when the filter input is `nil`. In Shopify Liquid, `nil` is returned whenever a property is accessed on an object that doesn't have that property set — this includes unset metafields, variant options that haven't been configured, product attributes that are blank, and customer properties that haven't been filled. When `nil` is passed to `upcase` or `downcase`, the filter returns an empty string `""` silently. There is no error, no log entry, and no indication that the input was nil. A badge label built as `{{ product.metafields.custom.badge_text.value | upcase }}` renders as an empty string if the metafield is not set for that product — the badge HTML element renders with no visible text, potentially producing an empty badge container that disrupts the layout.

This class of bug is particularly prevalent in variant option display, size guide labels, and badge systems — contexts where the developer intends to normalize the capitalization of merchant-entered text. The correct defensive pattern is to guard against nil before applying string filters, using `{% if value != blank %}` or the `| default:` filter to provide a fallback value before the case filter in the chain. The `| default:` filter returns the provided fallback when the input is nil, false, or empty string, ensuring the case filter always receives a string. This pattern also makes the intent explicit: the developer is declaring what the output should be when no value is set, rather than silently rendering blank.

---

## Detection Logic (AST)

1. **Identify `LiquidFilter` nodes where `name === "upcase"` or `name === "downcase"`.** The AST walker traverses all `LiquidFilter` nodes. Any node where `name` is `"upcase"` or `"downcase"` is a candidate for type-safety analysis. The filter is valid for string inputs, so the rule does not fire on all occurrences — only on those where the input type is determinably non-string or potentially nil.

2. **Resolve the type of the filter's input expression.** The walker traverses to the `LiquidVariable` parent of the `LiquidFilter[upcase/downcase]` node and inspects the full expression. If the filter is not the first in the chain, its input is the output of the previous filter — the walker resolves the output type of the preceding filter. If `upcase` or `downcase` is the first (or only) filter, its input is the raw Liquid variable expression.

3. **Check against known non-string Shopify drop properties.** The walker maintains a type registry of Shopify Liquid drop properties that return non-string types. Known integer properties include: `.price`, `.compare_at_price`, `.id`, `.position`, `.quantity`, `.weight`, `.variants.size`, `.images.size`, `.products_count`, `.items_count`. Known boolean properties include: `.available`, `.published`, `.requires_shipping`, `.taxable`. If the expression resolves to any of these, the rule fires — these are type errors, not nil-safety issues.

4. **Check for nil-unsafe metafield and option access patterns.** The walker additionally flags `| upcase` or `| downcase` applied to metafield value access expressions (`product.metafields.*.*.value`, `variant.metafields.*.*.value`) and variant option access (`variant.option1`, `variant.option2`, `variant.option3`) when no `| default:` filter precedes the case filter in the chain. These properties can return nil and are classified as nil-safety violations (MEDIUM severity, lower than type errors).

**Why regex fails here:** A regex matching `\|\s*(upcase|downcase)` applied after `\w+\.price` or similar patterns would require enumerating every known numeric Shopify property in the pattern — a list that expands with every Shopify API update. It also cannot detect the nil-propagation risk on metafield access, which depends on the property path pattern rather than the property name alone. The AST type registry is updatable independently of the detection logic; new non-string properties can be added to the registry without modifying the walker code.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: upcase/downcase applied to non-string and nil-unsafe objects.

  Failure mode 1: product.price is an Integer (price in cents).
  | upcase coerces 1999 to "1999" then upcases it — same output, type error.
  The developer intended {{ product.price | money | upcase }} but is applying
  the filter in the wrong position (before money, not after).

  Failure mode 2: variant.available is a Boolean.
  | downcase coerces `true` or `false` to "true" or "false".
  This is a logic error — the developer should use a conditional, not a filter.

  Failure mode 3: metafield .value — returns nil when metafield is not set.
  | upcase on nil returns "". Badge renders blank for products without the
  metafield. Empty badge <span> may break the layout or show an orphaned pill.

  Failure mode 4: variant.option1 — returns nil for products with fewer
  option dimensions than expected. | downcase on nil = silent empty output.
{%- endcomment -%}

<div class="product-card">
  {%- comment -%} Type error: price is Integer, not String -%}
  <p class="price">{{ product.price | upcase }}</p>

  {%- comment -%} Type error: available is Boolean — use conditional logic -%}
  <span class="availability">{{ variant.available | downcase }}</span>

  {%- comment -%}
    Nil-unsafe: metafield value is nil for products without this metafield.
    Renders blank badge element with no text — broken UI, no error.
  {%- endcomment -%}
  {% if product.metafields.custom.badge_text != blank %}
    <span class="product-badge">
      {{ product.metafields.custom.badge_text.value | upcase }}
    </span>
  {% endif %}

  {%- comment -%}
    Nil-unsafe: option2 is nil for single-option products.
    | downcase silently returns "" — size label renders blank.
  {%- endcomment -%}
  <p class="size-label">Size: {{ variant.option2 | downcase }}</p>

  {%- comment -%} Type error: variants.size is Integer -%}
  <p class="variant-count">{{ product.variants.size | upcase }} options</p>
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Apply upcase/downcase only to string values. Guard nil-unsafe sources.

  Change 1: product.price is an Integer — use | money filter, not | upcase.
  | money formats the integer as a currency string. | upcase on the result
  of | money is still unusual (currency strings don't typically need casing)
  but is at least a string-on-string operation.

  Change 2: variant.available is Boolean — use {% if %} conditional.
  Output a localized string for each state. Do not coerce booleans to strings.

  Change 3: metafield badge_text — guard with | default: '' before | upcase.
  | default: '' ensures upcase receives an empty string (not nil) when the
  metafield is unset. The {% if %} guard above the block prevents rendering
  the empty badge container.

  Change 4: variant.option2 — guard with | default: before | downcase.
  Provides a visible fallback for single-option products instead of blank output.

  Change 5: variants.size is Integer — output directly, no string filter needed.
{%- endcomment -%}

<div class="product-card">
  {%- comment -%}
    Correct: price formatted as currency string via | money.
    If uppercase currency display is needed: {{ product.price | money | upcase }}
    but only after confirming | money returns a string (it always does).
  {%- endcomment -%}
  <p class="price">{{ product.price | money }}</p>

  {%- comment -%}
    Correct: Boolean availability uses conditional, not string coercion.
    Outputs localized strings, not raw "true"/"false" values.
  {%- endcomment -%}
  {% if variant.available %}
    <span class="availability availability--in-stock">
      {{ 'products.product.in_stock' | t }}
    </span>
  {% else %}
    <span class="availability availability--sold-out">
      {{ 'products.product.sold_out' | t }}
    </span>
  {% endif %}

  {%- comment -%}
    Correct: metafield value guarded against nil with | default: ''.
    If badge_text metafield is unset, | default: '' returns empty string,
    upcase returns "", and the outer if prevents rendering a blank badge.
  {%- endcomment -%}
  {% assign badge_text = product.metafields.custom.badge_text.value | default: '' %}
  {% if badge_text != blank %}
    <span class="product-badge">{{ badge_text | upcase }}</span>
  {% endif %}

  {%- comment -%}
    Correct: option2 guarded with | default: fallback string.
    Single-option products display the fallback rather than blank output.
  {%- endcomment -%}
  {% assign size_label = variant.option2 | default: 'N/A' %}
  <p class="size-label">{{ 'products.product.size' | t }}: {{ size_label | downcase }}</p>

  {%- comment -%}
    Correct: variants.size is Integer — output directly, no filter needed.
    Integers require no case transformation.
  {%- endcomment -%}
  <p class="variant-count">{{ product.variants.size }} {{ 'products.product.options' | t }}</p>
</div>
```

---

## References

- [Shopify Liquid — `upcase` filter](https://shopify.dev/docs/api/liquid/filters/upcase)
- [Shopify Liquid — `downcase` filter](https://shopify.dev/docs/api/liquid/filters/downcase)
- [Shopify Liquid — `default` filter](https://shopify.dev/docs/api/liquid/filters/default)
- [Shopify Liquid — nil and blank values](https://shopify.dev/docs/api/liquid/basics#nil)
- [Shopify — Metafield types and value access](https://shopify.dev/docs/apps/build/custom-data/metafields/types)
