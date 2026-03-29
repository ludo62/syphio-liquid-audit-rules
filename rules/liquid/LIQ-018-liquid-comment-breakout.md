# LIQ-018 | Liquid comment containing `-->` or `</`

**Category:** Security

**Severity:** 🟡 MEDIUM

**CWE:** CWE-116 — Improper Encoding or Escaping of Output

**Affects:** Templates where `{% comment %}` blocks appear inside HTML comments or inside `<script>`/`<style>` raw text elements

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`{%- comment -%}...{%- endcomment -%}` blocks are processed and stripped from output by Liquid's parser before the resulting HTML is sent to the browser. When used at the top level of a Liquid template — outside any HTML context — this is safe: the comment content never reaches the HTML parser. However, developers occasionally nest Liquid comments inside HTML comments (`<!-- ... -->`) or inside `<script>` or `<style>` blocks, either to temporarily disable content while preserving it for recovery, or to annotate script blocks with structured metadata. In these contexts, the content of the Liquid comment becomes part of the HTML or raw text content delivered to the browser, and the browser's HTML parser applies its own tokenization rules to that content.

The HTML5 specification defines the comment state machine: a comment begun with `<!--` is terminated by the first occurrence of `-->`. The string `-->` inside a Liquid comment that appears within an HTML comment terminates the HTML comment at the browser's parser level, regardless of what Liquid has or has not processed. Everything after `-->` and before the developer's intended `-->` close becomes live HTML content — parsed and potentially rendered. If the content after the inadvertent `-->` includes HTML tags, those tags are instantiated in the DOM. If it includes JavaScript, and the surrounding context is a `<script>` block, those tokens are executed. Similarly, inside `<script>` or `<style>` raw text elements, the HTML parser terminates the element at the first occurrence of `</script>` or `</style>` — the `</` sequence followed by the tag name. A Liquid comment containing `</script>` inside a `<script>` block terminates the script context at the HTML parser level before the JavaScript engine processes the enclosed content.

This vulnerability class is distinct from direct XSS via user input. It occurs in developer-authored code where the developer intends for content to be invisible but lacks awareness of the HTML parser's tokenization rules. The severity is MEDIUM rather than CRITICAL because exploitation requires the developer to have written content containing `-->` or `</script>` inside a Liquid comment in an HTML comment or script context — a narrower failure mode than direct injection. However, in theme stores where themes are audited for security, this pattern has been flagged during manual security reviews as evidence of unsafe HTML comment handling practices.

---

## Detection Logic (AST)

1. **Identify `LiquidRawTag` nodes with tag name `comment`.** Walk the AST for `LiquidRawTag` nodes where `tag.name === 'comment'`. These represent `{% comment %}...{% endcomment %}` blocks and their inner content is accessible via `tag.body`.
2. **Determine the HTML context of the comment node.** Traverse the ancestor chain of the `LiquidRawTag` node. If any ancestor is an `HtmlComment` node (representing `<!-- ... -->`), classify the context as `html-comment`. If any ancestor is an `HtmlRawElement` with tag name `script` or `style`, classify the context as `raw-text-element`.
3. **Scan the comment body for dangerous sequences.** For `html-comment` context: check whether `tag.body` contains the substring `-->`. For `raw-text-element` context with tag name `script`: check whether `tag.body` contains `</script` (case-insensitive). For `style`: check for `</style`.
4. **Classify by context.** `html-comment` context with `-->` present: MEDIUM violation. `raw-text-element` context with matching close tag sequence: MEDIUM violation with a note that the raw text element is terminated at the HTML parser level.
5. **Emit the dangerous substring position within the comment body.** Include the character offset of the dangerous sequence in the diagnostic to pinpoint the exact location.

**Why regex fails here:** A regex scanning for `-->` in any Liquid comment cannot determine whether that comment is nested inside an HTML comment — it would flag Liquid comments at the document root that are safely stripped before HTML parsing, producing false positives. Distinguishing "Liquid comment inside HTML comment" from "Liquid comment at document root" requires knowledge of the ancestor HTML context, which is only available via AST traversal of the interleaved Liquid-HTML node tree.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Liquid comment appears inside an HTML comment. The comment body
  contains '-->'. The browser's HTML parser terminates the HTML comment at the
  first '-->' — the content after it becomes live HTML. Additionally, a Liquid
  comment inside a script block contains '</script>', which terminates the script
  element at the HTML parser level before the JS engine processes the content.
{%- endcomment -%}

<!-- Development note: disabled promo banner 2025-12-01 --> still investigating
{%- comment -%}
  Old promo block removed for Q4 campaign. Re-enable with:
  <div class="promo-banner">{{ section.settings.promo_text }}</div>
  --> this line terminates the HTML comment above at the browser parser level
{%- endcomment -%}
-->

<script>
  // Temporarily disabled tracking snippet
  {%- comment -%}
    var analyticsPayload = {{ product | json }};
    </script><script>alert('parser breakout')</script>
    sendAnalytics(analyticsPayload);
  {%- endcomment -%}
  window.shopCurrency = {{ shop.currency | json }};
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Move developer notes out of HTML comment context entirely. Use only
  Liquid-level comments (outside HTML comments) for documentation — these are
  stripped before HTML parsing and never reach the browser. In script contexts,
  use JavaScript line comments (//) rather than Liquid comments to disable code,
  and never include raw HTML close-tag sequences in commented content.
  If script content must be disabled at the Liquid level, remove it entirely
  or guard it with a settings flag rather than leaving it in a comment.
{%- endcomment -%}

{%- comment -%}
  Development note: promo banner disabled 2025-12-01, investigating Q4 campaign.
  Re-enable the block below once campaign assets are confirmed.
{%- endcomment -%}

{{- /* Promo banner: disabled — remove comment guards to re-enable */ -}}
{%- comment -%}
<div class="promo-banner">{{ section.settings.promo_text }}</div>
{%- endcomment -%}

<script>
  // Analytics snippet temporarily disabled — re-enable after consent framework audit
  // var analyticsPayload = {{ product | json }};
  // sendAnalytics(analyticsPayload);

  window.shopCurrency = {{ shop.currency | json }};
  window.shopLocale  = {{ request.locale.iso_code | json }};
</script>
```

---

## References

- [CWE-116: Improper Encoding or Escaping of Output](https://cwe.mitre.org/data/definitions/116.html)
- [HTML5 Specification: comment tokenization state](https://html.spec.whatwg.org/multipage/parsing.html#comment-start-state)
- [HTML5 Specification: script data state — raw text elements](https://html.spec.whatwg.org/multipage/parsing.html#script-data-state)
- [Shopify Liquid: comment tag](https://shopify.dev/docs/api/liquid/tags/comment)
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
