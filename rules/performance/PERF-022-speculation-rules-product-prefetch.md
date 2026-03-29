# PERF-022 | Speculation Rules API absent for product page prefetch

**Category:** Performance

**Severity:** 🟢 LOW

**Affects:** Collection page templates (`collection.liquid`), search results pages, and any template listing multiple product links

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

The Speculation Rules API is a browser-native declarative prefetch and prerender mechanism introduced in Chrome 109 and Edge 109 (January 2023). Unlike `<link rel="prefetch">`, which fetches a resource passively, the Speculation Rules API can trigger a full prerender of a destination page — including JavaScript execution and CSS application — while the user is still on the origin page. When the user navigates to the prerendered page, the browser activates the already-rendered page instantly, reducing the perceived navigation time to near-zero. For pages that are not fully prerendered, prefetch-level speculation still eliminates TTFB on the destination page, reducing navigation latency by 200–400ms on typical Shopify storefronts. The rules are declared as a `<script type="speculationrules">` block containing a JSON payload — on browsers that do not support the API (Safari, Firefox as of 2026-03), the script block is silently ignored with zero cost.

For Shopify themes, the highest-value speculation target is the product page navigation from collection pages. The collection-to-product-detail page (PDP) transition is the most critical path in the purchase funnel — it is the navigation a user takes when they are actively shopping. In Google's Chrome User Experience Report (CrUX) data for Shopify storefronts, collection-to-PDP navigation latency is a measurable contributor to session abandonment rates. A prerender rule that activates when a user hovers over a product link (using the `"eagerness": "moderate"` setting, which triggers on pointer-over with a 200ms debounce) effectively eliminates the navigation latency for most cursor-driven interactions. On mobile (where hover does not apply), `"eagerness": "conservative"` triggers on `pointerdown`, reducing latency for the tap-to-navigate interaction.

The implementation is a single `<script type="speculationrules">` block with a JSON body. The `"prerender"` array specifies URL patterns to match. For Shopify, product page URLs follow the pattern `/products/*`, which can be targeted with a `"href_matches"` pattern. The rule should be scoped to same-origin product URLs only — cross-origin speculation is blocked by browser security policy by default and would require explicit opt-in headers on the destination. The `"eagerness"` value should be `"moderate"` for desktop (hover-triggered) rather than `"eager"` (which would prerender all matching links on page load, wasting resources). The API carries no risk of over-fetching on non-supporting browsers because unsupported `<script>` types are inert.

---

## Detection Logic (AST)

1. **Identify collection and search result template files by filename.** The rule applies to `collection.liquid`, `search.liquid`, and any section file with a section schema `"templates"` value including `"collection"` or `"search"`. These are the templates where product links are most numerous and where speculation rules provide the highest impact.

2. **Scan `HtmlRawElement[script]` nodes for `type="speculationrules"`.** Within the identified template files and their rendered layout (`theme.liquid`), walk all `HtmlRawElement` nodes with `tagName === "script"`. For each, inspect the `HtmlAttribute` list for `name === "type"` with value `"speculationrules"`. If no such node exists in the template's output path (accounting for layout-level injection), the rule fires.

3. **If a `speculationrules` script exists, validate its JSON content.** Extract the `RawMarkup` content of the `speculationrules` script block. Attempt JSON parsing. Verify that the parsed object contains either a `"prerender"` or `"prefetch"` key. Verify that at least one rule within those arrays contains a `"href_matches"` or `"selector_matches"` condition that would match product page URLs (containing `/products/`). If the block is present but malformed or does not match product URLs, emit LOW advisory.

4. **Check `eagerness` value.** If the speculation rule is present and valid, inspect the `"eagerness"` property. A value of `"eager"` on a collection page with many product links (24+) may prerender all products simultaneously, consuming excessive memory (each prerender holds a full page render in memory). Emit LOW advisory if `"eagerness": "eager"` is used on a template with `collection.products_count` potentially exceeding 12.

**Why regex fails here:** A regex scanning for `speculationrules` in template files cannot determine whether the script block is present in the layout file (which applies to all templates) versus only in a specific section or template. The AST walker resolves the complete render graph — layout + template + sections + snippets — and can determine whether a `speculationrules` block exists anywhere in the composed output path for a collection page render. Additionally, validating the JSON content of the script block requires structured parsing, not pattern matching.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Collection page renders product cards with links but has no
  Speculation Rules API block. Every collection-to-product navigation
  incurs a full TTFB round-trip to Shopify's edge (~150–400ms).

  Failure mode 1: Users browsing the collection grid click a product and
  wait for the full page navigation — DNS + TCP + TLS + TTFB + HTML parse
  + LCP. No prefetch or prerender was initiated during browse time.

  Failure mode 2: The collection page is one of the highest-traffic pages
  in a Shopify store. Every session that browses products pays the full
  navigation cost. The Speculation Rules API would eliminate this cost
  on Chrome/Edge (70%+ of Shopify traffic) with zero impact on Safari.

  Failure mode 3: No speculation = no competitive differentiation.
  Stores using the Dawn theme with Speculation Rules enabled appear
  measurably faster on the collection-to-PDP transition.
{%- endcomment -%}

{%- paginate collection.products by section.settings.products_per_page -%}
  <div class="collection" data-section-id="{{ section.id }}">
    <ul class="product-grid" role="list">
      {%- render 'product-card' for collection.products as product,
        show_vendor: section.settings.show_vendor
      -%}
    </ul>

    {%- if paginate.pages > 1 -%}
      {%- render 'pagination', paginate: paginate -%}
    {%- endif -%}
  </div>
{%- endpaginate -%}

{{- 'collection.js' | asset_url | script_tag -}}
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add a Speculation Rules API block targeting product page URLs.

  Change 1: <script type="speculationrules"> with a JSON body. Browsers
  that do not support the API (Safari, Firefox) silently ignore this
  script block — zero cost, zero risk on unsupporting browsers.

  Change 2: "prerender" rule with "href_matches": "/products/*" targets
  all same-origin product URLs. The pattern matches Shopify's standard
  product URL structure including handle-based URLs.

  Change 3: "eagerness": "moderate" triggers prerender on pointer-over
  with a ~200ms debounce. This limits prerender to links the user is
  actively targeting, avoiding simultaneous prerender of all 24 product
  cards on page load (which would exhaust memory).

  Change 4: A "prefetch" fallback rule with "eagerness": "conservative"
  (triggers on pointerdown) provides a lower-cost hint for browsers that
  support prefetch-level speculation but not full prerender.
{%- endcomment -%}

{%- paginate collection.products by section.settings.products_per_page -%}
  <div class="collection" data-section-id="{{ section.id }}">
    <ul class="product-grid" role="list">
      {%- render 'product-card' for collection.products as product,
        show_vendor: section.settings.show_vendor
      -%}
    </ul>

    {%- if paginate.pages > 1 -%}
      {%- render 'pagination', paginate: paginate -%}
    {%- endif -%}
  </div>
{%- endpaginate -%}

{{- 'collection.js' | asset_url | script_tag -}}

<!-- Speculation Rules: prerender product pages on hover (Chrome/Edge 109+) -->
<script type="speculationrules">
{
  "prerender": [
    {
      "where": { "href_matches": "/products/*" },
      "eagerness": "moderate"
    }
  ],
  "prefetch": [
    {
      "where": { "href_matches": "/products/*" },
      "eagerness": "conservative"
    }
  ]
}
</script>
```

---

## References

- [MDN — Speculation Rules API](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API)
- [web.dev — Speculation Rules API](https://developer.chrome.com/docs/web-platform/prerender-pages)
- [Chrome — Speculation Rules explainer](https://github.com/WICG/nav-speculation/blob/main/triggers.md)
- [web.dev — Prefetch resources to speed up future navigations](https://web.dev/articles/link-prefetch)
- [Shopify — Theme performance best practices](https://shopify.dev/docs/storefronts/themes/best-practices/performance)
