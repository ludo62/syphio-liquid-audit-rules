# LIQ-015 | `{% render %}` with undefined variable as parameter

**Category:** Merchant Logic

**Severity:** 🟠 HIGH

**Affects:** All templates using `{% render %}` with explicit parameter passing; product, collection, cart, and account templates

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

When a `{% render %}` call passes a named parameter whose value is a variable that is undefined in the current rendering scope, the parameter arrives inside the snippet as `nil`. Liquid's scoping rules for `{% render %}` are strict by design: the snippet executes in an isolated scope that inherits only explicitly passed parameters, Shopify global objects (`shop`, `request`, `routes`, `settings`), and loop variables if the snippet is rendered via `{% render 'name' for array as item %}`. No variables from the calling template's local scope leak into the snippet. The result is that an undefined variable passed as a parameter is indistinguishable from an intentionally nil-valued parameter — the snippet receives `nil` with no diagnostic information.

Liquid renders `nil` as an empty string in output expressions. Any conditional inside the snippet that tests `{% if param %}` evaluates to false when `param` is `nil`, causing entire branches to silently not render. Unlike JavaScript `undefined` errors or Python `NameError` exceptions, Liquid does not raise an error or emit a warning when an undefined variable is accessed. The page renders, apparently successfully, but with missing content. This makes the defect difficult to detect in manual QA — the developer must know what the expected output was in order to notice its absence. In Shopify's Storefront Renderer, this pattern is particularly dangerous because the silence is total: no server log entry, no browser console warning, no visible error.

The bug surface is large. Undefined variable parameters occur most frequently when a `{% render %}` call is copied from a product page template (where `product` is the page object) into a collection page template (where `product` is the loop iteration variable `{% for product in collection.products %}` and may not be defined at the point of the `render` call), or into a search results template where the variable name differs. They also occur when a developer renames an `assign` variable and forgets to update all `render` call sites. Shopify's Theme Check detects some cases via static analysis, but dynamic variable names and aliased objects defeat its scope resolution.

---

## Detection Logic (AST)

1. **Identify `LiquidTag` nodes with tag name `render`.** Walk the AST for all `LiquidTag` nodes where `tag.name === 'render'`. These represent snippet invocations.
2. **Extract named parameters.** For each `render` tag, collect all named parameters from the tag's `args` list. Each parameter has a `name` (the identifier inside the snippet) and a `value` (the expression from the calling scope).
3. **Resolve each parameter value to a variable.** For parameter values that are `VariableLookup` nodes (i.e., not string literals, number literals, or `true`/`false`), extract the root identifier name.
4. **Query the symbol table for the identifier.** Check whether the root identifier was defined in the current template scope via a prior `assign`, `capture`, `for` loop variable declaration, `paginate` tag, or is a known Shopify global object. If the identifier is not found in any of these, classify the parameter as potentially undefined.
5. **Emit HIGH violation with parameter name and snippet name.** Report the specific parameter and snippet filename so the developer can locate and correct the missing `assign` or fix the variable reference.

**Why regex fails here:** A regex matching `render 'snippet' with undefined_var` cannot determine whether `undefined_var` is defined — the definition may appear on any prior line, in a parent `{% liquid %}` block, in a `{% capture %}` block, or in a `{% for %}` loop header. Regex has no awareness of scope: it treats every identifier as either present or absent on the same line. AST-level analysis with a symbol table built from traversing `assign`, `capture`, `for`, `paginate`, and `tablerow` tags in document order is the only mechanism that can accurately determine whether a variable is in scope at the point of the `render` call.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: render call on a collection page passes 'current_product' which
  was never assigned in this template. On a product page, 'product' is the page
  object. On a collection page, the developer intended to use the loop variable
  but used the wrong name. The snippet receives nil and renders no price or CTA.
  No error is raised. The storefront appears to load correctly.
{%- endcomment -%}

{{- /* collection.liquid — current_product is never defined in this scope */ -}}
<div class="collection-grid">
  {%- for product in collection.products -%}
    <div class="collection-grid__item">

      {{- /* BUG: 'current_product' is undefined; snippet receives nil */ -}}
      {%- render 'product-card',
        product: current_product,
        show_vendor: true,
        show_quick_add: section.settings.enable_quick_add
      -%}

    </div>
  {%- endfor -%}
</div>

{{- /* product-card.liquid — the snippet param 'product' is nil */ -}}
{%- if product -%}
  {{- /* This entire block is silently skipped */ -}}
  <article class="product-card">
    <h2>{{ product.title }}</h2>
    <span class="price">{{ product.price | money }}</span>
  </article>
{%- endif -%}
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Pass the loop variable 'product' (defined by the for loop) explicitly to
  the render call. Each render tag should reference only variables that are
  demonstrably in scope at the call site. When refactoring render calls across
  templates, audit every parameter name against the destination template's scope.
  Add an explicit fallback check inside snippets for nil parameters to surface
  errors visibly in development (using Shopify's comment-debug pattern).
{%- endcomment -%}

{{- /* collection.liquid — 'product' is the for-loop variable, defined per iteration */ -}}
<div class="collection-grid">
  {%- for product in collection.products -%}
    <div class="collection-grid__item">

      {{- /* CORRECT: pass the in-scope loop variable 'product' */ -}}
      {%- render 'product-card',
        product: product,
        show_vendor: true,
        show_quick_add: section.settings.enable_quick_add
      -%}

    </div>
  {%- endfor -%}
</div>

{{- /* product-card.liquid — defensive nil guard surfaces broken calls in dev */ -}}
{%- if product == nil -%}
  {%- comment -%}DEBUG: product-card rendered with nil product parameter{%- endcomment -%}
{%- else -%}
  <article class="product-card">
    <h2>{{ product.title }}</h2>
    <span class="price">{{ product.price | money }}</span>
    {%- if show_vendor -%}
      <span class="vendor">{{ product.vendor }}</span>
    {%- endif -%}
  </article>
{%- endif -%}
```

---

## References

- [Shopify Liquid: render tag — variable scoping](https://shopify.dev/docs/api/liquid/tags/render)
- [Shopify Theme Check: UndefinedObject](https://shopify.dev/docs/themes/tools/theme-check/checks/undefined-object)
- [Shopify Liquid: variable scoping rules](https://shopify.dev/docs/api/liquid/basics#variable-scope)
