# PERF-021 | `window.Shopify.routes.root` absent from AJAX calls

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** JavaScript files, inline `<script>` blocks in section and layout files that make fetch/XMLHttpRequest calls to Shopify API endpoints

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Shopify Markets enables international storefronts with locale-specific URL prefixes: `/en-us/`, `/fr-fr/`, `/de-de/`, `/pt-br/`, and so on. When a storefront is configured for a non-root market, every Shopify API endpoint is served at the market-prefixed path. The cart AJAX endpoint moves from `/cart/add.js` to `/en-us/cart/add.js`. The product recommendations endpoint moves from `/recommendations/products.json` to `/en-us/recommendations/products.json`. Hardcoded paths — strings that begin with `/cart/`, `/collections/`, `/search`, `/recommendations/` — return 404 errors on any non-root market because the server-side router expects the locale prefix. This is not a graceful degradation; it is a complete functional failure. Cart add-to-cart buttons stop working. Collection filtering returns empty results. Product recommendations fail silently. On a Shopify store with Markets enabled, any merchant who activates a second market immediately breaks every AJAX feature that uses hardcoded paths.

`window.Shopify.routes.root` is a JavaScript string property injected by Shopify into every storefront page via `{{ content_for_header }}`. On the root market (the default locale), its value is `"/"`. On a localized market, its value is the locale prefix: `"/en-us/"`, `"/fr-fr/"`, etc. Every AJAX path must be constructed by prepending this value: `${window.Shopify.routes.root}cart/add.js` resolves to `/cart/add.js` on the root market and `/en-us/cart/add.js` on the English-US market — correct in both cases. The variable is available immediately on DOMContentLoaded because `{{ content_for_header }}` emits an inline script block that sets it synchronously. It does not require an async load or a wait state.

The failure mode is particularly damaging for international merchants because it manifests only after Markets activation — the theme appears to work correctly in the default locale. Developers testing on a single-market store never encounter the bug. The defect is discovered when the merchant goes international, at which point all JavaScript-driven commerce features break simultaneously across the new market. Shopify's Theme Store requires that submitted themes use `window.Shopify.routes.root` for all cart, search, and API AJAX paths — failure to do so results in rejection. For themes already published to the Theme Store, merchant support tickets about broken international carts often trace back to this single omission.

---

## Detection Logic (AST)

1. **Identify `HtmlRawElement[script]` nodes and inline script content.** The AST walker collects `HtmlRawElement` nodes with `tagName === "script"`. For each, extract the inner `RawMarkup` content. Also collect `HtmlElement` nodes with `tagName === "script"` that contain inline text. JavaScript asset files (`.js`) linked via `src` attribute must be analyzed separately — flag them as a separate finding indicating that JavaScript files require manual audit for hardcoded paths.

2. **Scan script content for hardcoded Shopify API path prefixes.** Within the extracted script text, apply pattern matching for string literals beginning with the known Shopify AJAX endpoint prefixes: `/cart/`, `/collections/`, `/search`, `/recommendations/`, `/account`, `/pages/`, `/blogs/`. Flag any JavaScript string literal (single-quoted, double-quoted, or template literal) that starts with these prefixes and does not include a variable interpolation of `Shopify.routes.root` or `window.Shopify.routes.root` before the path segment.

3. **Verify that flagged paths are not already prefixed by `routes.root`.** For each candidate hardcoded path, inspect the surrounding expression context. If the string is part of a template literal that includes `${window.Shopify.routes.root}` or `${Shopify.routes.root}` as a preceding interpolation, or if the string is concatenated with a variable referencing `routes.root`, the usage is compliant. Only strings used as self-contained URL values without routes.root prefixing are violations.

4. **Classify severity by endpoint type.** Hardcoded paths to `/cart/add.js`, `/cart/change.js`, `/cart/update.js`, `/cart/clear.js` are HIGH severity — these are primary commerce operations. Hardcoded paths to `/recommendations/products.json`, `/search/suggest.json` are MEDIUM severity — these affect discovery features but not purchase flow. Hardcoded paths to `/account` or `/pages/` are LOW severity.

**Why regex fails here:** A regex matching `/cart/` in script content would flag every occurrence, including strings inside comments, strings inside non-fetch contexts (CSS class names, HTML strings that happen to contain path text), and strings inside conditional branches that only execute on the root market. The AST approach — parsing JavaScript content from the `RawMarkup` node and resolving string literal contexts — can exclude commented code, identify the expression context of each string literal, and distinguish between a URL passed to `fetch()` versus a URL in an `href` assignment or a log message. A regex cannot differentiate these cases without parser-level context.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: AJAX calls to Shopify endpoints using hardcoded paths.
  These work on the default market but break with 404 errors on any
  Shopify Markets locale that has a URL prefix enabled.
{%- endcomment -%}

<script>
  // Anti-pattern: hardcoded /cart/ path — breaks on /en-us/, /fr-fr/, etc.
  class CartManager {
    constructor() {
      this.cartCountEl = document.querySelector('.cart-count');
    }

    async addItem(variantId, quantity = 1) {
      const response = await fetch('/cart/add.js', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ id: variantId, quantity })
      });

      if (!response.ok) throw new Error('Cart add failed');
      await this.refreshCart();
    }

    async refreshCart() {
      // Hardcoded /cart.js path — same 404 failure on localized markets
      const response = await fetch('/cart.js');
      const cart = await response.json();
      this.cartCountEl.textContent = cart.item_count;
    }

    async removeItem(lineItemKey) {
      // Hardcoded /cart/change.js — breaks internationally
      await fetch('/cart/change.js', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ id: lineItemKey, quantity: 0 })
      });
      await this.refreshCart();
    }
  }

  window.cartManager = new CartManager();
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Prefix all Shopify AJAX endpoint paths with window.Shopify.routes.root.

  Change 1: window.Shopify.routes.root is injected by {{ content_for_header }}
  as a synchronous inline script. It is available immediately — no async wait
  or DOMContentLoaded guard is needed. On the root market its value is "/";
  on localized markets it is the locale prefix (e.g. "/en-us/").

  Change 2: Template literals with ${window.Shopify.routes.root} correctly
  compose the full path: on root market -> "/cart/add.js"; on en-us market
  -> "/en-us/cart/add.js". No conditional logic required — the prefix
  handles both cases transparently.

  Change 3: Note that routes.root already ends with "/", so the path
  segment must NOT begin with "/". Use "cart/add.js" not "/cart/add.js"
  to avoid double-slash URLs ("/en-us//cart/add.js").

  Change 4: content_for_header must be present in the layout file for
  window.Shopify to be available. This is a layout-level requirement,
  not a per-script requirement. All Shopify-compliant themes include it.
{%- endcomment -%}

<script>
  class CartManager {
    constructor() {
      this.cartCountEl = document.querySelector('.cart-count');
      // routes.root ends with "/"; path segments must NOT start with "/"
      this.routes = window.Shopify.routes.root;
    }

    async addItem(variantId, quantity = 1) {
      const response = await fetch(`${this.routes}cart/add.js`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ id: variantId, quantity })
      });

      if (!response.ok) throw new Error('Cart add failed');
      await this.refreshCart();
    }

    async refreshCart() {
      // Correctly prefixed: resolves to /cart.js or /en-us/cart.js
      const response = await fetch(`${this.routes}cart.js`);
      const cart = await response.json();
      this.cartCountEl.textContent = cart.item_count;
    }

    async removeItem(lineItemKey) {
      // Correctly prefixed: resolves to /cart/change.js or /en-us/cart/change.js
      await fetch(`${this.routes}cart/change.js`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ id: lineItemKey, quantity: 0 })
      });
      await this.refreshCart();
    }
  }

  window.cartManager = new CartManager();
</script>
```

---

## References

- [Shopify — `window.Shopify.routes.root`](https://shopify.dev/docs/api/ajax/reference/cart#routes-root)
- [Shopify — Markets: URL structure](https://shopify.dev/docs/apps/build/markets/urls)
- [Shopify — AJAX API overview](https://shopify.dev/docs/api/ajax)
- [Shopify — `content_for_header`](https://shopify.dev/docs/api/liquid/objects/content_for_header)
- [Shopify — Theme Store requirements](https://shopify.dev/docs/storefronts/themes/store/requirements)
