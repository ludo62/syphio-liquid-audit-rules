# LIQ-019 | `| escape` inside `<script>` tag instead of `| json`

**Category:** Security

**Severity:** 🔴 CRITICAL

**CWE:** CWE-79 — Improper Neutralization of Input During Web Page Generation (Cross-site Scripting)

**OWASP:** A03:2021 — Injection

**Affects:** All templates with `<script>` blocks that embed Liquid variables; `product.liquid`, `collection.liquid`, `cart.liquid`, theme layout files

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`| escape` applies HTML entity encoding to its input: `<` → `&lt;`, `>` → `&gt;`, `&` → `&amp;`, `"` → `&quot;`, `'` → `&#39;`. These transformations are correct and sufficient for HTML text content and HTML attribute values, where the browser's HTML parser performs entity decoding before presenting the value to the rendering engine. Inside a `<script>` block, however, the HTML parser operates in raw text mode: it does not apply entity decoding. The JavaScript engine receives the literal byte sequence that the HTML parser delivers — including the literal string `&lt;` rather than `<`. This means `| escape` does not transform the data into a form that is semantically safe in JavaScript string context; it transforms it into a form that is correct for HTML parsing but produces corrupted string values in JavaScript.

The XSS vector is precise and not immediately obvious. Consider a product title containing `</script><script>alert(document.domain)</script>`. Applying `| escape` transforms `<` → `&lt;`, `>` → `&gt;`, and `"` → `&quot;`. The forward slash `/` is not escaped by `| escape`. The HTML parser, upon encountering `</script>` inside the `<script>` raw text element, terminates the script block at that position — before the JavaScript engine has processed the string assignment. The `&lt;` entities that `| escape` produced from the preceding `<` characters are irrelevant: the parser already found the termination sequence `</script>` and closed the element. The content after the first `</script>` becomes live HTML, and the second `<script>alert(document.domain)</script>` executes. The `| escape` filter provided false confidence without providing actual protection.

The correct filter for embedding Liquid values in JavaScript string context is `| json`. The `| json` filter produces a JSON-encoded string including surrounding double quotes: `{{ product.title | json }}` outputs `"My Product Title"`. Critically, `| json` escapes the forward slash (`/` → `\/`), preventing `</script>` from terminating the script element. It also escapes backslashes, double quotes, and control characters in a way that is semantically valid to the JavaScript engine. A value assigned as `var title = {{ product.title | json }};` is a correctly typed JavaScript string without surrounding quote delimiters in the template. This rule is one of the highest-priority security rules in the Syphio ruleset because the false confidence pattern is widespread in legacy Shopify themes predating the `| json` filter's standardisation.

---

## Detection Logic (AST)

1. **Identify `HtmlRawElement` nodes with tag name `script`.** Walk the AST for `HtmlRawElement` nodes where the element's tag name is `script` and the `type` attribute is absent, `text/javascript`, or `module` (i.e., JavaScript execution contexts, not `application/json` or `text/template` blocks).
2. **Collect `LiquidVariable` descendants inside the script body.** For each matching `HtmlRawElement`, traverse its children to find all `LiquidVariable` nodes embedded in the raw text content.
3. **Inspect the filter chain of each `LiquidVariable`.** For each `LiquidVariable`, check whether its filter chain includes a `LiquidFilter` with `name === 'escape'` or `name === 'escape_once'`.
4. **Verify absence of `json` filter.** If `escape` or `escape_once` is present and `json` is absent from the filter chain, emit a CRITICAL violation. If `json` is present in the chain (regardless of `escape` being also present), emit a separate MEDIUM violation noting that `| escape` after `| json` double-encodes the output and is unnecessary.
5. **Flag unfiltered variables as HIGH.** Any `LiquidVariable` inside a `<script>` block with no filter at all (raw interpolation of a non-static value) should be flagged as HIGH — unfiltered output in script context is an XSS risk even if lower severity than the wrong filter pattern.

**Why regex fails here:** A regex scanning for `| escape` anywhere in a file cannot determine whether the expression is inside a `<script>` block or inside HTML body content where `| escape` is correct. Detecting the HTML parsing context of a Liquid expression requires knowledge of its ancestor elements — specifically whether an `HtmlRawElement` with tag name `script` is an ancestor in the interleaved Liquid-HTML AST. Without this context, the regex produces massive false-positive rates (flagging correct `| escape` usage in HTML body) and cannot reason about the parsing mode in which the browser will evaluate the output.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: | escape is used inside a <script> block to embed Liquid values.
  HTML entity encoding is wrong for JavaScript context: &lt; appears literally
  as &lt; in JS strings (not as <), and the forward slash in </script> is NOT
  escaped by | escape, allowing a crafted product title or metafield value to
  terminate the script block and inject executable markup into the page.
{%- endcomment -%}

<script>
  // WRONG: | escape does not protect in JS context
  var productTitle   = "{{ product.title | escape }}";
  var productVendor  = "{{ product.vendor | escape }}";
  var collectionName = "{{ collection.title | escape }}";

  // WRONG: unescaped metafield — direct injection vector
  var productMeta    = "{{ product.metafields.custom.promo_text }}";

  // WRONG: | escape on an object serialization — loses type information
  var selectedVariant = {
    id:    {{ product.selected_or_first_available_variant.id }},
    price: {{ product.selected_or_first_available_variant.price }},
    title: "{{ product.selected_or_first_available_variant.title | escape }}"
  };

  ShopTheme.init(productTitle, selectedVariant);
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Use | json for all Liquid variable interpolation in <script> context.
  | json wraps the output in double quotes and escapes /, \, ", and control chars.
  The escaped forward slash (/) prevents </script> from terminating the element.
  For object and array values, | json serialises the entire Liquid object to
  valid JSON — no manual property extraction needed, no type mismatch possible.
  Remove the surrounding "..." delimiters from the template — | json includes them.
{%- endcomment -%}

<script>
  // CORRECT: | json escapes / and produces valid JS string literals with quotes
  var productTitle   = {{ product.title | json }};
  var productVendor  = {{ product.vendor | json }};
  var collectionName = {{ collection.title | json }};

  // CORRECT: | json on metafields — escapes all JS-dangerous characters
  var productMeta = {{ product.metafields.custom.promo_text | json }};

  // CORRECT: | json on a variant object — full serialisation, correct types
  var selectedVariant = {{ product.selected_or_first_available_variant | json }};

  // CORRECT: for complex data payloads, build an object in Liquid and serialise
  {%- assign page_data = product -%}
  var pageData = {{ page_data | json }};

  ShopTheme.init(productTitle, selectedVariant);
</script>
```

---

## References

- [CWE-79: Improper Neutralization of Input During Web Page Generation](https://cwe.mitre.org/data/definitions/79.html)
- [OWASP A03:2021 — Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [OWASP XSS Prevention Cheat Sheet: JavaScript contexts](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html#rule-2-attribute-encode-before-inserting-untrusted-data-into-html-common-attributes)
- [Shopify Liquid: json filter](https://shopify.dev/docs/api/liquid/filters/json)
- [Shopify Liquid: escape filter](https://shopify.dev/docs/api/liquid/filters/escape)
- [HTML5 Specification: script raw text element parsing](https://html.spec.whatwg.org/multipage/parsing.html#script-data-state)
