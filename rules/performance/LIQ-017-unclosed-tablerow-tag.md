# LIQ-017 | `{% tablerow %}` without `{% endtablerow %}`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Templates using `{% tablerow %}` for tabular data display; product specification tables, variant comparison grids

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`{% tablerow %}` is a Liquid block tag that generates `<tr>` and `<td>` HTML table elements for each iteration of an array. It must be closed with a matching `{% endtablerow %}` tag. Without it, the Liquid parser raises a syntax error at render time and the entire section or template block containing the unclosed tag renders blank. Shopify's Storefront Renderer does not emit a partial render for sections with syntax errors — the section output is suppressed in its entirety and replaced with an empty string. On Online Store 2.0 themes, this means the section's slot in the layout renders as empty space rather than raising a visible error, making the defect non-obvious to developers who test only in the Theme Editor where errors are surfaced in the preview panel.

`{% tablerow %}` exposes `tablerowloop` — a loop object with properties `tablerowloop.col`, `tablerowloop.row`, `tablerowloop.col_first`, `tablerowloop.col_last`, `tablerowloop.first`, `tablerowloop.last`, `tablerowloop.col_count`, and `tablerowloop.index`. These properties are only populated within a valid `{% tablerow %}...{% endtablerow %}` block. Code referencing `tablerowloop` properties outside a properly closed block receives `nil` silently — the properties do not raise an error, they simply return nil, causing column-break logic and conditional first/last-cell styling to fail invisibly. In addition, the `cols:` parameter (e.g., `{% tablerow product in collection.products cols: 3 %}`) controls the number of columns per row; without `{% endtablerow %}`, the `cols:` parameter is parsed but never applied.

A secondary failure mode occurs during Theme Store automated validation. Shopify's Theme Inspector scans theme files for unclosed block tags as part of the submission review pipeline. An unclosed `{% tablerow %}` in any template file — even one that is not exercised by the automated test store — will cause the theme to fail the syntax validation phase of Theme Store review. The rule applies equally to `{% tablerow %}` tags inside `{% liquid %}` multi-statement blocks, where the visual absence of `{%` and `%}` delimiters makes the missing `endtablerow` harder to spot.

---

## Detection Logic (AST)

1. **Identify `LiquidTag` nodes with tag name `tablerow`.** Walk the AST for all `LiquidTag` nodes where `tag.name === 'tablerow'`.
2. **Verify the `blockEndTag` property.** In the Liquid AST produced by `@shopify/liquid-html-parser`, block tags carry a `blockEndTag` property that references the corresponding close tag node. If `tag.blockEndTag` is `null` or `undefined`, the block was not closed.
3. **Verify tag pair integrity.** Cross-reference the position of the opening `tablerow` tag against the document's list of `endtablerow` tags. If the opening tag has no corresponding close at the same nesting depth, emit a HIGH violation.
4. **Check for `tablerowloop` references outside valid blocks.** As a secondary pass, identify any `VariableLookup` nodes whose root identifier is `tablerowloop`. Verify that each such reference is a descendant of a properly closed `tablerow` block node. References outside a valid block emit a HIGH violation with a message indicating that `tablerowloop` is only available inside `{% tablerow %}`.
5. **Report the opening tag position.** Include the line number of the `{% tablerow %}` opening tag in the diagnostic to allow the developer to locate the unclosed block.

**Why regex fails here:** A regex can identify `{% tablerow %}` opening tags and scan for `{% endtablerow %}` tags, but it cannot correctly account for nesting depth across multi-line blocks, cannot distinguish a `{% tablerow %}` inside a `{% comment %}` or `{% raw %}` block (where it should not be counted), and cannot validate the structural pairing of open and close tags when there are multiple `{% tablerow %}` blocks in the same file. AST parsing resolves block tag pairing structurally during the parse phase, making the `blockEndTag` property a single O(1) property access rather than a multi-pass regex scan.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: tablerow tag is opened but never closed with endtablerow.
  The Liquid parser raises a syntax error at render time. The entire section
  renders blank. tablerowloop.col and tablerowloop.col_last are also referenced
  in the intended output — these would return nil even if the tag were closed,
  because they are referenced in a separate output expression outside the block.
  The cols: parameter is parsed but never applied without endtablerow.
{%- endcomment -%}

<section class="variant-comparison">
  <h2>{{ section.settings.heading }}</h2>

  <table class="variant-table">
    <thead>
      <tr>
        <th>Variant</th>
        <th>Price</th>
        <th>SKU</th>
        <th>Available</th>
      </tr>
    </thead>
    <tbody>
      {%- tablerow variant in product.variants cols: 4 -%}
        <td class="{% if tablerowloop.col_first %}first-col{% endif %}">
          {{ variant.title }}
        </td>
        <td>{{ variant.price | money }}</td>
        <td>{{ variant.sku | default: '—' }}</td>
        <td>{{ variant.available | yesno: 'Yes,No' }}</td>
      {{- /* Missing: endtablerow — entire section renders blank */ -}}
    </tbody>
  </table>
</section>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Close the tablerow block with endtablerow. The tablerowloop object is
  then fully populated within the block. cols: 4 is applied correctly, generating
  a new <tr> after every 4th variant. tablerowloop.col_first and col_last are
  used for conditional cell styling within the block where they are valid.
  The blockEndTag is present — AST validation passes, section renders correctly.
{%- endcomment -%}

<section class="variant-comparison">
  <h2>{{ section.settings.heading }}</h2>

  <table class="variant-table">
    <thead>
      <tr>
        <th>Variant</th>
        <th>Price</th>
        <th>SKU</th>
        <th>Available</th>
      </tr>
    </thead>
    <tbody>
      {%- tablerow variant in product.variants cols: 4 -%}
        {{- /* tablerowloop properties are valid here — inside the closed block */ -}}
        <td class="{%- if tablerowloop.col_first -%}first-col{%- endif -%}">
          {{ variant.title }}
        </td>
        <td>{{ variant.price | money }}</td>
        <td>{{ variant.sku | default: '—' }}</td>
        <td>
          {%- if variant.available -%}
            <span class="badge badge--success">In stock</span>
          {%- else -%}
            <span class="badge badge--error">Sold out</span>
          {%- endif -%}
        </td>
      {%- endtablerow -%}
    </tbody>
  </table>
</section>
```

---

## References

- [Shopify Liquid: tablerow tag](https://shopify.dev/docs/api/liquid/tags/tablerow)
- [Shopify Liquid: tablerowloop object](https://shopify.dev/docs/api/liquid/objects/tablerowloop)
- [Shopify Theme Check: SyntaxError](https://shopify.dev/docs/themes/tools/theme-check/checks)
- [Shopify Theme Store: submission requirements](https://shopify.dev/docs/themes/store/requirements)
