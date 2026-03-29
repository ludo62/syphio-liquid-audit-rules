# LIQ-024 | `{% liquid %}` tag with missing `assign` keyword

**Category:** Merchant Logic

**Severity:** 🟡 MEDIUM

**Affects:** All templates and snippets using `{% liquid %}` multi-statement blocks; particularly sections refactored from multiple `{% assign %}` tags

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

The `{% liquid %}` multi-line tag allows multiple Liquid statements to be written without individual `{%` and `%}` delimiters. Each line within a `{% liquid %}` block is a complete Liquid statement. The `assign` keyword is required to bind a value to a variable name: `assign variable_name = value`. A common mistake is writing `variable_name = value` without the `assign` keyword — the Liquid parser interprets this as a bare expression. In most Liquid implementations, a bare expression that does not match any known tag name is silently ignored: the statement evaluates the right-hand side and discards the result without binding it to any variable. The identifier `variable_name` remains undefined in the outer scope, and any subsequent reference to it returns nil with no error.

This error pattern is acutely common during refactoring. When a developer consolidates a block of multiple individual `{% assign %}` tags into a single `{% liquid %}` block, the refactoring step strips the `{% assign %}` delimiters but may omit the `assign` keyword from one or more statements — particularly when the refactoring is done manually via find-and-replace. The resulting `{% liquid %}` block appears structurally correct (correct indentation, correct number of lines, correct closing tag), but one or more variables are not assigned. Because the variables remain undefined and Liquid renders nil as an empty string, the page renders without an obvious error: fields display as blank, conditionals evaluate as false, and price calculations silently produce `0` or empty money-formatted strings.

The `{% liquid %}` block's compactness — its primary benefit — also makes this error harder to spot in code review than the equivalent error in standard `{% assign %}` tags. In a block of 15 `assign` statements, a missing keyword on line 8 does not alter the visual structure of the block. Theme Check's `ValidSchema` check does not cover this pattern; it is a semantic error in the Liquid statement rather than a schema or syntax error. Syphio detects it via AST inspection of the `{% liquid %}` block's parsed statement list, where each statement carries a type classification that distinguishes `assign` statements from bare expression statements.

---

## Detection Logic (AST)

1. **Identify `LiquidTag` nodes with tag name `liquid`.** Walk the AST for all `LiquidTag` nodes where `tag.name === 'liquid'`. These represent `{% liquid %}` multi-statement blocks, whose child statements are parsed as a flat list.
2. **Extract the statement list from the `liquid` tag body.** The `{% liquid %}` tag's body contains a sequence of parsed statement nodes. Each node has a `type` or `kind` property identifying what kind of statement it is (e.g., `LiquidTagAssign`, `LiquidTagIf`, `LiquidTagRender`, `LiquidRawTag`, `TextNode`).
3. **Identify `TextNode` or unrecognised expression nodes.** Bare expressions (lines without a recognised tag keyword) are parsed as `TextNode` or expression nodes inside the `{% liquid %}` block. A line of the form `variable_name = value` with no `assign` keyword produces either a parse error node or a raw text node depending on the parser implementation.
4. **Pattern-match the raw content of unrecognised nodes.** For each `TextNode` or error node inside the `{% liquid %}` block, check whether its raw content matches the pattern `/^[a-z_][a-z0-9_]*\s*=\s*.+/i` — i.e., an identifier followed by `=` followed by a value, with no leading keyword. This is the fingerprint of a missing `assign` keyword.
5. **Emit MEDIUM violation with the variable name and line number.** Report the apparent variable name (the left-hand side of the `=`) in the diagnostic so the developer can identify which assignment is missing its keyword.

**Why regex fails here:** A regex scanning the raw source for lines matching `variable_name = value` inside `{% liquid %}` blocks would flag legitimate Liquid `if` condition comparisons (`if count = items | size` — though this is not valid Liquid syntax, similar patterns exist), legitimate `assign` statements where the keyword is present (`assign count = items | size` — the regex would match after stripping `assign`), and Liquid `echo` statements or `render` parameters that use `=` for parameter assignment (`render 'snippet', param: value`). Only AST-level analysis of the `{% liquid %}` block's parsed statement list can identify which lines were parsed as unrecognised expressions versus recognised `assign` statements.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: A {% liquid %} block refactored from individual {% assign %} tags.
  The 'assign' keyword was dropped from three statements during refactoring.
  'product_title', 'on_sale', and 'price_range' are never assigned — they remain
  nil throughout the template. The product card renders with blank title, no
  sale badge, and no price range. No error is raised anywhere in the render cycle.
{%- endcomment -%}

{% liquid
  assign locale         = request.locale.iso_code
  assign current_tags   = collection.current_tags
  product_title         = product.title | escape
  assign image_src      = product.featured_image | image_url: width: 600
  assign image_alt      = product.featured_image.alt | default: product.title
  on_sale               = product.compare_at_price > product.price
  assign variants_count = product.variants.size
  price_range           = product.price_min | money | append: ' – ' | append: product.price_max | money
%}

<article class="product-card">
  <h2>{{ product_title }}</h2>  {{- /* nil — renders blank */ -}}

  {%- if on_sale -%}            {{- /* nil — never true, badge never renders */ -}}
    <span class="badge">Sale</span>
  {%- endif -%}

  <img src="{{ image_src }}" alt="{{ image_alt }}">
  <p class="price">{{ price_range }}</p>  {{- /* nil — renders blank */ -}}
</article>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add the missing 'assign' keyword to each statement in the {% liquid %}
  block. Every variable binding requires 'assign' as the first token on the line.
  After refactoring from individual {% assign %} tags, verify each line in the
  {% liquid %} block starts with a recognised keyword: assign, capture, echo,
  if, unless, case, for, render, paginate, comment, or liquid.
{%- endcomment -%}

{% liquid
  assign locale         = request.locale.iso_code
  assign current_tags   = collection.current_tags
  assign product_title  = product.title | escape
  assign image_src      = product.featured_image | image_url: width: 600
  assign image_alt      = product.featured_image.alt | default: product.title
  assign on_sale        = false
  if product.compare_at_price > product.price
    assign on_sale = true
  endif
  assign variants_count = product.variants.size
  assign price_min      = product.price_min | money
  assign price_max      = product.price_max | money
  assign price_range    = price_min | append: ' – ' | append: price_max
%}

<article class="product-card">
  <h2>{{ product_title }}</h2>

  {%- if on_sale -%}
    <span class="badge">Sale</span>
  {%- endif -%}

  <img src="{{ image_src }}" alt="{{ image_alt }}">
  <p class="price">{{ price_range }}</p>
</article>
```

---

## References

- [Shopify Liquid: liquid tag](https://shopify.dev/docs/api/liquid/tags/liquid)
- [Shopify Liquid: assign tag](https://shopify.dev/docs/api/liquid/tags/assign)
- [Shopify Theme Check: checks reference](https://shopify.dev/docs/themes/tools/theme-check/checks)
- [Shopify Liquid: variable scope in liquid blocks](https://shopify.dev/docs/api/liquid/basics#variable-scope)
