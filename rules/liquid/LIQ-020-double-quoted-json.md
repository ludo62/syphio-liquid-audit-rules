# LIQ-020 | `'{{ value | json }}'` — `| json` wrapped in additional quotes

**Category:** Security

**Severity:** 🟠 HIGH

**CWE:** CWE-116 — Improper Encoding or Escaping of Output

**Affects:** All templates with `<script>` blocks that embed Liquid variables; particularly themes migrated from `| escape` to `| json` without removing surrounding quote delimiters

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

The `| json` filter wraps its output in double quotes as part of the JSON encoding specification: `{{ product.title | json }}` outputs `"My Product Title"` — the double quotes are part of the filter's output, not part of the surrounding template. When a developer additionally wraps this expression in single-quote string delimiters in the JavaScript assignment — `var title = '{{ product.title | json }}'` — the rendered output becomes `var title = '"My Product Title"'`. The variable `title` is now a string whose value is `"My Product Title"` — a string containing literal double-quote characters. The type is correct in a superficial sense (it is a string), but the value is wrong: it is a JSON-encoded string rather than the raw string value. Any code that subsequently uses `title` in a comparison, a DOM text node assignment, or an analytics payload will receive the double-quote-wrapped version, not the intended plain string.

The more critical failure mode occurs during theme migrations. Themes that previously used `| escape` for JavaScript interpolation — `var title = "{{ product.title | escape }}"` — are frequently updated by replacing `escape` with `json` without removing the surrounding quotes: `var title = "{{ product.title | json }}"`. The result is a string containing literal double-quote characters: `var title = ""My Product Title""`. This produces a JavaScript syntax error for string values that do not contain quotes, and a functional error for string values that do. For values containing single quotes (e.g., a product titled `"O'Brien Wetsuit"`), the `| json` filter correctly escapes the single quote as `\u0027` or `\'`. But if the surrounding template delimiter is a single quote, the escaping strategy of `| json` is correct for JSON/JS string content but the outer single-quote delimiter creates a mismatched quoting context: `var title = '{{ "O\'Brien Wetsuit" }}'` — the backslash-escaped single quote inside the `| json` output may or may not be interpreted correctly depending on the JavaScript engine's handling of the escape sequence in the outer context.

This rule targets a pattern that frequently evades manual review because the rendered output "looks almost right" — the value is present, the page does not crash on most inputs, and the bug only manifests for specific product names or when the developer attempts to use the value programmatically. The correct form for JavaScript string interpolation is `{{ value | json }}` with no surrounding template-level quotes: the quotes are provided by `| json`.

---

## Detection Logic (AST)

1. **Identify `HtmlRawElement` nodes with tag name `script`.** Walk the AST for `HtmlRawElement` nodes in JavaScript execution context (same script-type filter as LIQ-019).
2. **Parse the raw text content for string literal contexts.** Within the script body, identify character positions that are inside JavaScript string literal delimiters: single-quote (`'...'`), double-quote (`"..."`), or template literal (`` `...` ``) contexts.
3. **Find `LiquidVariable` nodes whose source position falls inside a string literal context.** For each `LiquidVariable` embedded in the script body, determine whether its source position is between string-delimiter characters in the surrounding static text.
4. **Inspect the filter chain for `json` filter.** If the `LiquidVariable`'s filter chain includes a `LiquidFilter` with `name === 'json'`, and the expression's position is inside string delimiters, emit a HIGH violation. The diagnostic should identify the delimiter character, the filter, and the variable name.
5. **Distinguish double-wrapped variants.** If the surrounding delimiter is a double quote (`"`) and `| json` is used, note that the rendered output will contain literal `"value"` inside `"..."`, producing `""value""` — a JavaScript syntax error for most string values. If the surrounding delimiter is a single quote, note the quoting-context mismatch.

**Why regex fails here:** A regex can match `'{{ ... | json }}'` on a single line but cannot determine whether a `{{ ... | json }}` expression on a line is inside a JavaScript string literal context when the opening and closing quotes are on different lines (multiline template literals, concatenated strings), or when the Liquid expression spans multiple lines. More fundamentally, a regex cannot distinguish JavaScript string contexts from HTML attribute contexts — `class="{{ value | json }}"` would be a false positive (wrapping in HTML attribute quotes is correct for non-JS attributes). Only AST traversal with awareness of both the HTML raw-element boundary and the JavaScript string-literal context can correctly identify the violation.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: | json is used but its output is additionally wrapped in
  single-quote or double-quote template delimiters. | json already outputs
  quotes as part of its encoding. The double-wrapping produces literal quote
  characters in the JS value. For a title "O'Brien", the rendered output
  becomes var title = '"O\'Brien"' — a string containing JSON, not a string.
  The second example shows a migrated escape pattern where " delimiters remain.
{%- endcomment -%}

<script>
  // WRONG: single-quote wrapping — title receives '"Product Title"' as value
  var productTitle = '{{ product.title | json }}';

  // WRONG: double-quote wrapping after escape→json migration — syntax error risk
  var vendorName = "{{ product.vendor | json }}";

  // WRONG: both quotes and json in attribute-style assignment
  var sku = '{{ variant.sku | json }}';

  // WRONG: template literal with json — double-quote chars inside backtick string
  var description = `{{ product.description | strip_html | json }}`;

  // WRONG: object literal property with extra quotes
  window.ShopData = {
    title:    '{{ product.title | json }}',
    vendor:   '{{ product.vendor | json }}',
    currency: '{{ shop.currency | json }}'
  };
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Remove all surrounding quote delimiters from | json interpolations.
  | json outputs a fully quoted, fully escaped JSON string value — including
  the enclosing double quotes. The template should use | json without any
  surrounding template-level string delimiters. For object literals, assign
  each property directly from | json output. For multiple properties, consider
  serialising the entire Liquid object with | json for type fidelity.
{%- endcomment -%}

<script>
  // CORRECT: no surrounding quotes — | json provides them
  var productTitle = {{ product.title | json }};

  // CORRECT: no surrounding quotes after escape→json migration
  var vendorName = {{ product.vendor | json }};

  // CORRECT: | json on plain-text fields — null-safe via default filter
  var sku = {{ variant.sku | default: '' | json }};

  // CORRECT: strip_html first (description is HTML), then json for JS context
  var description = {{ product.description | strip_html | json }};

  // CORRECT: object literal — each property is a | json expression without quotes
  window.ShopData = {
    title:    {{ product.title | json }},
    vendor:   {{ product.vendor | json }},
    currency: {{ shop.currency | json }},
    locale:   {{ request.locale.iso_code | json }}
  };
</script>
```

---

## References

- [CWE-116: Improper Encoding or Escaping of Output](https://cwe.mitre.org/data/definitions/116.html)
- [Shopify Liquid: json filter](https://shopify.dev/docs/api/liquid/filters/json)
- [OWASP XSS Prevention Cheat Sheet: JavaScript Data Values](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [MDN: JSON.stringify — string encoding](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify)
