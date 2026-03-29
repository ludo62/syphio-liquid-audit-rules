# LIQ-006 | `{% capture %}` without `{% endcapture %}`

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** Any template, section, or layout file containing a `{% capture %}` block without a matching `{% endcapture %}`

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{% capture %}` opens a string buffer in Liquid's output stream. All content that would normally be written to the HTTP response — HTML, Liquid variable output, whitespace — is instead redirected into an internal string accumulator bound to the variable name declared in the tag. The HTTP response output is suppressed for the duration of the capture block. This redirection is absolute: no output escapes to the response until `{% endcapture %}` is encountered. If `{% endcapture %}` is missing, the capture buffer remains open for the remainder of the template, and all subsequent content is swallowed into the variable and never rendered.

The failure mode is a blank page or a partially rendered page with no visible error. The visitor sees only the content that was rendered before the `{% capture %}` tag was encountered — typically just the opening HTML structure from the layout file, with no body content. The issue does not raise a visible exception in Shopify's production rendering pipeline; the template "succeeds" from the renderer's perspective, it simply produces a response body of minimal or zero meaningful HTML. This makes the bug extremely difficult to diagnose without source-level inspection. Shopify's storefront error log will not capture this as an error; it is syntactically valid Liquid that produces an unintended (but well-defined) output.

The bug most commonly originates from three sources: (1) merge conflicts that split a `{% capture %}...{% endcapture %}` block across conflict markers, leaving one side with only the opening tag; (2) copy-paste errors where a developer copies the `{% capture %}` tag from one location and forgets to include the `{% endcapture %}`; and (3) template refactoring where the `{% endcapture %}` is deleted when removing content from the block but the opening tag is left in place. When it occurs in `layout/theme.liquid`, the entire store body — every page, every section, every product — is swallowed. The store appears completely blank to all visitors until the theme is rolled back.

---

## Detection Logic (AST)

1. **Collect all `LiquidTag[name=capture]` nodes.** The AST walker traverses the template tree and collects every `LiquidTag` where `name === "capture"`. These are `LiquidTagOpen` nodes that begin a block scope — they require a matching `LiquidTagClose[name=endcapture]` at the same nesting depth.

2. **Build a stack-based open/close matcher.** For each `LiquidTag[capture]` encountered during traversal, a depth counter is pushed onto a stack with the tag's source position. For each `LiquidTag[endcapture]` encountered, the stack is popped. If the depth counter is already zero when an `endcapture` is encountered, it is an orphaned closing tag. If the stack is non-empty after the full template has been traversed, every remaining stack entry is an unclosed `capture` tag — each one is a violation.

3. **Handle nested captures.** Liquid supports nested `{% capture %}` blocks (a capture inside a capture is valid, though unusual). The stack-based approach handles arbitrary nesting depth correctly by tracking each open tag independently, keyed by variable name. A `{% endcapture %}` closes the most recently opened capture, regardless of nesting depth.

4. **Check cross-file partial splits.** If the file is a snippet (under `/snippets/`) and contains a `LiquidTag[capture]` with no matching `endcapture`, cross-file analysis is triggered: inspect all files that `{% render %}` this snippet to determine whether the `endcapture` appears in the calling template. This pattern (open in snippet, close in parent) is not valid Liquid behavior — `{% render %}` does not share scope — and the split is flagged as CRITICAL with an explanation that scope isolation prevents cross-file capture blocks.

**Why regex fails here:** A regex matching `{%\s*capture\s+\w+\s*%}` counts opening tags, and a regex matching `{%\s*endcapture\s*%}` counts closing tags. Comparing these counts would give false negatives when a template has correctly balanced captures in a different order (e.g., conditional captures where one branch has a capture and another doesn't). The AST stack-based approach tracks nesting position and identifies exactly which opening tag is unmatched, providing an accurate source location for the violation.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: {% capture %} blocks without {% endcapture %}.

  Failure mode 1: Main capture for structured badge HTML — endcapture omitted.
  Everything after this tag (the entire product section body) is captured into
  `product_badge` and never rendered. Visitor sees blank product section.

  Failure mode 2: Nested capture for price string — also unclosed.
  Even if the outer capture were closed, this inner capture would swallow
  the product title, description, and add-to-cart form.

  Failure mode 3: In layout/theme.liquid — the entire store body (all sections,
  header, footer, cart, everything) is swallowed. Full blank store.
{%- endcomment -%}

{%- comment -%} Missing endcapture — badge HTML and everything below is eaten -%}
{% capture product_badge %}
  {% if product.available %}
    <span class="badge badge--available">In Stock</span>
  {% elsif product.compare_at_price > product.price %}
    <span class="badge badge--sale">Sale</span>
  {% endif %}

{%- comment -%} This second capture is also unclosed — double failure -%}
{% capture formatted_price %}
  <span class="price">{{ product.price | money }}</span>
  {% if product.compare_at_price > product.price %}
    <span class="price--compare">{{ product.compare_at_price | money }}</span>
  {% endif %}

<div class="product-main">
  <h1>{{ product.title }}</h1>
  {{ product_badge }}
  {{ formatted_price }}
  <div class="product-description">
    {{ product.description }}
  </div>
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Close every {% capture %} block with {% endcapture %}.

  Change 1: product_badge capture is correctly closed with {% endcapture %}.
  The badge HTML is stored in the variable and rendered explicitly below.

  Change 2: formatted_price capture is correctly closed with {% endcapture %}.
  The price HTML is stored and rendered at the correct position in the layout.

  Change 3: Both variables are output explicitly with {{ }}.
  Output is predictable and ordered — no content is swallowed.
{%- endcomment -%}

{%- comment -%}
  Capture product_badge: builds the badge HTML string into a variable.
  Closed with endcapture on the correct line.
{%- endcomment -%}
{% capture product_badge %}
  {% if product.available %}
    <span class="badge badge--available">In Stock</span>
  {% elsif product.compare_at_price > product.price %}
    <span class="badge badge--sale">Sale</span>
  {% endif %}
{% endcapture %}

{%- comment -%}
  Capture formatted_price: builds the price HTML string into a variable.
  Closed with endcapture immediately after the content.
{%- endcomment -%}
{% capture formatted_price %}
  <span class="price">{{ product.price | money }}</span>
  {% if product.compare_at_price > product.price %}
    <span class="price--compare">{{ product.compare_at_price | money }}</span>
  {% endif %}
{% endcapture %}

{%- comment -%}
  Both variables are now correctly scoped strings.
  Output them explicitly at the correct positions in the layout.
{%- endcomment -%}
<div class="product-main">
  <h1>{{ product.title }}</h1>
  {{ product_badge }}
  {{ formatted_price }}
  <div class="product-description">
    {{ product.description }}
  </div>
</div>
```

---

## References

- [Shopify Liquid — `capture` tag](https://shopify.dev/docs/api/liquid/tags/capture)
- [Shopify Theme Check — MissingEndtag rule](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/missing-endtag)
- [Shopify — Liquid template rendering pipeline](https://shopify.dev/docs/storefronts/themes/architecture)
- [Shopify — Theme rollback and version management](https://shopify.dev/docs/storefronts/themes/tools/theme-kit)
