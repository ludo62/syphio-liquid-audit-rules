# LIQ-004 | Misspelled closing tag `{% endiff %}` / `{% end for %}`

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** Any template containing block tags (`{% if %}`, `{% for %}`, `{% unless %}`, `{% case %}`) with misspelled or malformed closing counterparts

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

Shopify's Liquid renderer is a strict parser. Tag names are matched against a fixed set of known identifiers — there is no fuzzy matching, no autocorrection, and no fallback behavior. When the parser encounters an unknown tag name such as `{% endiff %}`, `{% end for %}` (with a space), or `{% endIf %}` (with uppercase), it raises a `LiquidError::SyntaxError`. In Shopify's storefront rendering pipeline, a syntax error in a section causes that section to render as completely blank — the HTML container is output, but all content within the section is suppressed. The visitor sees an empty white area with no indication of failure, no error message, and no console output in production.

The failure mode is particularly dangerous because it survives most development workflows. In the Shopify theme editor (Online Store > Themes > Customize), section previews may not render at all, but the editor UI continues to function. The theme can be published with the error in place. Theme Check catches this class of error during `shopify theme check`, but only if the developer runs it — it is not enforced on publish. Once deployed, the error only manifests to visitors, not to developers reviewing code in the editor. Storefront error logs (accessible via Partner Dashboard) capture the SyntaxError with a line number, but merchants rarely monitor these logs.

The most common variants are: `{% endiff %}` (double-f, mirroring the if/endif asymmetry that confuses developers coming from other templating languages), `{% end for %}` (space between `end` and `for`, common in merge conflict resolution artifacts), `{% endIf %}` or `{% endFor %}` (case errors from auto-correct tools), and `{% end_if %}` or `{% end_for %}` (underscore variants from Ruby/Python background developers). Each variant is a distinct unknown tag that triggers the same blank-render outcome. This rule uses edit-distance analysis on tag names to catch all variants without enumerating every possible misspelling.

---

## Detection Logic (AST)

1. **Collect all `LiquidTag` nodes from the parsed template.** The AST walker traverses all `LiquidTag` nodes in the document. The `LiquidTag.name` property contains the normalized tag name string as parsed from the source.

2. **Build the reference set of valid closing tag names.** The complete set of valid Liquid closing tags is: `endif`, `endfor`, `endunless`, `endcase`, `endcapture`, `endform`, `endpaginate`, `endtablerow`, `endschema`, `endstyle`, `endjavascript`, `endsection`, `endraw`. This set is fixed and does not change between Liquid versions supported by Shopify.

3. **Apply normalized edit-distance comparison.** For each `LiquidTag.name` that does not appear in the known valid tag set (opening or closing), the walker computes the Levenshtein edit distance between the unknown name and each member of the valid closing tag set. The normalization step lowercases the tag name and removes all whitespace characters before comparison, catching case and space variants. Any unknown tag with edit distance ≤ 2 from a valid closing tag is flagged as a probable misspelling.

4. **Classify severity by block tag type.** Misspellings of `endif` and `endfor` are CRITICAL — these are the most common block tags and their corruption blanks entire section content. Misspellings of `endcapture` are CRITICAL because they trigger the page-swallow failure described in LIQ-006. Misspellings of `endschema`, `endstyle`, and `endjavascript` are HIGH — they corrupt section schema metadata or inline asset processing.

**Why regex fails here:** A regex matching patterns like `{%\s*end\s*if\s*%}` or `{%\s*endiff\s*%}` requires a separate pattern for every misspelling variant. The combinatorial space of possible misspellings (case variations × space insertions × character transpositions) is too large to enumerate exhaustively. The AST approach normalizes the tag name and applies edit-distance comparison once against the known-good set, catching all variants including ones not yet observed in the wild.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Multiple misspelled closing tags across a product section.

  Failure mode 1: {% endiff %} — double-f typo. Parser raises SyntaxError.
  The entire section renders blank. Visitors see empty product area.

  Failure mode 2: {% end for %} — space in tag name. Unknown tag to Liquid.
  The for loop renders blank. All product cards are suppressed.

  Failure mode 3: {% endIf %} — case mismatch from autocorrect. Unknown tag.
  The conditional block never closes. Everything after it is consumed.

  Failure mode 4: {% end_unless %} — underscore variant. Raises SyntaxError.
  The unless block never terminates. Customer-specific content is suppressed.
{%- endcomment -%}

<div class="product-section">
  {% if product.available %}
    <div class="product-available">
      <p>In stock — ships within 2 business days.</p>

      {% for variant in product.variants %}
        <div class="variant-option">
          <label>{{ variant.title }}</label>
          <span>{{ variant.price | money }}</span>
        </div>
      {% end for %}

    </div>
  {% endiff %}

  {% unless customer %}
    <div class="guest-prompt">
      <p>Sign in for member pricing.</p>
      <a href="{{ routes.account_login_url }}">Sign in</a>
    </div>
  {% end_unless %}

  {% if product.metafields.custom.highlight_text != blank %}
    <div class="product-highlight">
      <p>{{ product.metafields.custom.highlight_text.value }}</p>
    </div>
  {% endIf %}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Correct all closing tag names to their valid lowercase single-word forms.

  Change 1: {% endiff %} → {% endif %} — single i, no double-f.
  Change 2: {% end for %} → {% endfor %} — no space, single word.
  Change 3: {% endIf %} → {% endif %} — all lowercase.
  Change 4: {% end_unless %} → {% endunless %} — no underscore.

  All Liquid closing tags follow the pattern: "end" + opening tag name,
  no spaces, no underscores, all lowercase. There are no exceptions.
{%- endcomment -%}

<div class="product-section">
  {% if product.available %}
    <div class="product-available">
      <p>In stock — ships within 2 business days.</p>

      {%- comment -%}
        endfor: no space, no uppercase. Closes the for block correctly.
      {%- endcomment -%}
      {% for variant in product.variants %}
        <div class="variant-option">
          <label>{{ variant.title }}</label>
          <span>{{ variant.price | money }}</span>
        </div>
      {% endfor %}

    </div>
  {%- comment -%}
    endif: single word, lowercase. Closes the if block correctly.
  {%- endcomment -%}
  {% endif %}

  {%- comment -%}
    endunless: no underscore, no space. Closes the unless block correctly.
  {%- endcomment -%}
  {% unless customer %}
    <div class="guest-prompt">
      <p>Sign in for member pricing.</p>
      <a href="{{ routes.account_login_url }}">Sign in</a>
    </div>
  {% endunless %}

  {% if product.metafields.custom.highlight_text != blank %}
    <div class="product-highlight">
      <p>{{ product.metafields.custom.highlight_text.value }}</p>
    </div>
  {%- comment -%} endif: lowercase only. -%}
  {% endif %}
</div>
```

---

## References

- [Shopify Liquid — Tag reference (all valid tag names)](https://shopify.dev/docs/api/liquid/tags)
- [Shopify Theme Check — UnknownTag rule](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/unknown-tag)
- [Shopify — Liquid SyntaxError behavior in sections](https://shopify.dev/docs/storefronts/themes/architecture/sections)
- [Levenshtein distance — edit distance algorithm](https://en.wikipedia.org/wiki/Levenshtein_distance)
