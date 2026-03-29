# Syphio — Complete Rule Index

> **251 rules** · 14 categories · 54 🔴 Critical · 146 🟠 High · 45 🟡 Medium · 6 🟢 Low
> Version 2.0 · March 2026 · [syphio.io](https://syphio.io)

Click any rule ID to read its full specification with Detection Logic, Anti-Pattern, and Optimized Fix.

---

## Legend

|     | Severity     | When to fix                                                                         |
| --- | ------------ | ----------------------------------------------------------------------------------- |
| 🔴  | **CRITICAL** | Store crash · XSS exploitable · legal violation · data loss — fix immediately       |
| 🟠  | **HIGH**     | Conversion impact · Theme Store rejection · measurable degradation — fix within 48h |
| 🟡  | **MEDIUM**   | Code quality · maintainability · minor performance — fix next sprint                |
| 🟢  | **LOW**      | Best practice · future-proofing — fix when convenient                               |

---

## Quick Jump

- [C1 — Liquid: Syntax & Deprecated Tags](#c1--liquid-syntax--deprecated-tags-25-rules) `25 rules`
- [C2 — Performance & Core Web Vitals](#c2--performance--core-web-vitals-30-rules) `30 rules`
- [C3 — JavaScript: Architecture & Lifecycle](#c3--javascript-architecture--lifecycle-28-rules) `28 rules`
- [C4 — Security](#c4--security-23-rules) `23 rules`
- [C5 — Accessibility (EAA 2025 / WCAG 2.2 AA)](#c5--accessibility-eaa-2025--wcag-22-aa-35-rules) `35 rules`
- [C6 — SEO & Structured Data](#c6--seo--structured-data-30-rules) `30 rules`
- [C7 — Schema JSON & Section Architecture](#c7--schema-json--section-architecture-17-rules) `17 rules`
- [C8 — Metafields & Metaobjects](#c8--metafields--metaobjects-10-rules) `10 rules`
- [C9 — Markets, i18n & Localisation](#c9--markets-i18n--localisation-9-rules) `9 rules`
- [C10 — Checkout, Selling Plans & Subscriptions](#c10--checkout-selling-plans--subscriptions-9-rules) `9 rules`
- [C11 — Hydrogen & Headless](#c11--hydrogen--headless-10-rules) `10 rules`
- [C12 — Theme Check & CI/CD](#c12--theme-check--cicd-12-rules) `12 rules`
- [C13 — Klaviyo & Third-Party Integrations](#c13--klaviyo--third-party-integrations-8-rules) `8 rules`
- [C14 — GDPR & Data Privacy](#c14--gdpr--data-privacy-5-rules) `5 rules`

---

## C1 — Liquid: Syntax & Deprecated Tags `25 rules`

| ID                             | Rule                                                                              | Severity    |
| ------------------------------ | --------------------------------------------------------------------------------- | ----------- |
| [LIQ-001](./liquid/LIQ-001.md) | `{% include %}` deprecated — replace with `{% render %}`                          | 🟠 HIGH     |
| [LIQ-002](./liquid/LIQ-002.md) | `img_url` filter obsolete — replace with `image_url`                              | 🟠 HIGH     |
| [LIQ-003](./liquid/LIQ-003.md) | `= 0` instead of `== 0` in `{% if %}` — always evaluates true                     | 🔴 CRITICAL |
| [LIQ-004](./liquid/LIQ-004.md) | Misspelled closing tag `{% endiff %}` `{% end for %}`                             | 🔴 CRITICAL |
| [LIQ-005](./liquid/LIQ-005.md) | `{% assign %}` inside `{% for %}` loop                                            | 🟡 MEDIUM   |
| [LIQ-006](./liquid/LIQ-006.md) | `{% capture %}` without `{% endcapture %}`                                        | 🔴 CRITICAL |
| [LIQ-007](./liquid/LIQ-007.md) | Filter chain depth > 6 levels                                                     | 🟡 MEDIUM   |
| [LIQ-008](./liquid/LIQ-008.md) | `{{ content_for_header }}` missing or duplicated in layout                        | 🔴 CRITICAL |
| [LIQ-009](./liquid/LIQ-009.md) | `{{ content_for_layout }}` missing in checkout.liquid                             | 🔴 CRITICAL |
| [LIQ-010](./liquid/LIQ-010.md) | `{% layout none %}` without explicit `<!DOCTYPE html>`                            | 🟡 MEDIUM   |
| [LIQ-011](./liquid/LIQ-011.md) | `{% paginate %}` without `{% endpaginate %}`                                      | 🟠 HIGH     |
| [LIQ-012](./liquid/LIQ-012.md) | `{% form %}` without `{% endform %}`                                              | 🟠 HIGH     |
| [LIQ-013](./liquid/LIQ-013.md) | `\| upcase` / `\| downcase` on non-string object                                  | 🟡 MEDIUM   |
| [LIQ-014](./liquid/LIQ-014.md) | `\| strip_html` on already plain-text field                                       | 🟢 LOW      |
| [LIQ-015](./liquid/LIQ-015.md) | `{% render %}` called with undefined variable as parameter                        | 🟠 HIGH     |
| [LIQ-016](./liquid/LIQ-016.md) | Nested `{% unless %}` creating cognitive confusion                                | 🟢 LOW      |
| [LIQ-017](./liquid/LIQ-017.md) | `{% tablerow %}` without `{% endtablerow %}`                                      | 🟠 HIGH     |
| [LIQ-018](./liquid/LIQ-018.md) | Liquid comment containing `-->` or `</` sequences                                 | 🟡 MEDIUM   |
| [LIQ-019](./liquid/LIQ-019.md) | `\| escape` inside `<script>` tag instead of `\| json`                            | 🔴 CRITICAL |
| [LIQ-020](./liquid/LIQ-020.md) | `'{{ value \| json }}'` — json wrapped in extra quotes                            | 🟠 HIGH     |
| [LIQ-021](./liquid/LIQ-021.md) | Deprecated `theme` Liquid object used                                             | 🟠 HIGH     |
| [LIQ-022](./liquid/LIQ-022.md) | `.css.liquid` or `.js.liquid` file — deprecated since 2025                        | 🔴 CRITICAL |
| [LIQ-023](./liquid/LIQ-023.md) | `\| asset_url` referencing non-existent asset in `/assets/`                       | 🟠 HIGH     |
| [LIQ-024](./liquid/LIQ-024.md) | `{% liquid %}` tag with missing `assign`                                          | 🟡 MEDIUM   |
| [LIQ-025](./liquid/LIQ-025.md) | `{{ product.selected_variant }}` instead of `selected_or_first_available_variant` | 🟠 HIGH     |

---

## C2 — Performance & Core Web Vitals `30 rules`

| ID                                    | Rule                                                              | Severity    |
| ------------------------------------- | ----------------------------------------------------------------- | ----------- |
| [PERF-001](./performance/PERF-001.md) | `collections.all` in non-paginated context                        | 🔴 CRITICAL |
| [PERF-002](./performance/PERF-002.md) | `all_products[handle]` — O(n) full catalog scan                   | 🔴 CRITICAL |
| [PERF-003](./performance/PERF-003.md) | `collections.all` inside `{% for %}` loop                         | 🔴 CRITICAL |
| [PERF-004](./performance/PERF-004.md) | Hero image missing `fetchpriority="high"`                         | 🟠 HIGH     |
| [PERF-005](./performance/PERF-005.md) | Hero image with `loading="lazy"` — LCP killer                     | 🔴 CRITICAL |
| [PERF-006](./performance/PERF-006.md) | `image_url` without `widths` parameter                            | 🟠 HIGH     |
| [PERF-007](./performance/PERF-007.md) | `<img>` without explicit `width` and `height` attributes          | 🟠 HIGH     |
| [PERF-008](./performance/PERF-008.md) | `script_tag` filter without `defer`                               | 🟠 HIGH     |
| [PERF-009](./performance/PERF-009.md) | Inline `<script>` without `defer`/`async` on non-critical path    | 🟠 HIGH     |
| [PERF-010](./performance/PERF-010.md) | jQuery loaded as global dependency — 87KB blocking                | 🟠 HIGH     |
| [PERF-011](./performance/PERF-011.md) | `section.settings` accessed inside `{% for %}` loop               | 🟡 MEDIUM   |
| [PERF-012](./performance/PERF-012.md) | `\| where` filter on array > 100 elements                         | 🟠 HIGH     |
| [PERF-013](./performance/PERF-013.md) | `\| sort` on `collections.all.products`                           | 🔴 CRITICAL |
| [PERF-014](./performance/PERF-014.md) | `font_face` without `font_display: 'swap'`                        | 🟠 HIGH     |
| [PERF-015](./performance/PERF-015.md) | Google Fonts without `&display=swap`                              | 🟠 HIGH     |
| [PERF-016](./performance/PERF-016.md) | Missing `preconnect` for critical third-party origins             | 🟡 MEDIUM   |
| [PERF-017](./performance/PERF-017.md) | `dns-prefetch` instead of `preconnect` for same-origin fonts      | 🟢 LOW      |
| [PERF-018](./performance/PERF-018.md) | `section.index` not used to differentiate above/below fold images | 🟠 HIGH     |
| [PERF-019](./performance/PERF-019.md) | `{% render %}` in `{% for %}` without `for:` syntax               | 🟡 MEDIUM   |
| [PERF-020](./performance/PERF-020.md) | CSS loaded via `stylesheet_tag` without `media` attribute         | 🟡 MEDIUM   |
| [PERF-021](./performance/PERF-021.md) | `window.Shopify.routes.root` absent from AJAX calls               | 🟠 HIGH     |
| [PERF-022](./performance/PERF-022.md) | Speculation Rules API absent for product page prefetch            | 🟢 LOW      |
| [PERF-023](./performance/PERF-023.md) | `innerHTML +=` pattern in cart/section JavaScript                 | 🟠 HIGH     |
| [PERF-024](./performance/PERF-024.md) | `setInterval` without `clearInterval` in Web Component            | 🟠 HIGH     |
| [PERF-025](./performance/PERF-025.md) | `document.querySelector` instead of `this.querySelector`          | 🟠 HIGH     |
| [PERF-026](./performance/PERF-026.md) | INP > 200ms — interaction handler blocking main thread            | 🟠 HIGH     |
| [PERF-027](./performance/PERF-027.md) | `scheduler.yield()` absent in long INP handlers > 50ms            | 🟡 MEDIUM   |
| [PERF-028](./performance/PERF-028.md) | App scripts loaded on all pages vs template-specific              | 🟠 HIGH     |
| [PERF-029](./performance/PERF-029.md) | `request.page_type` not used to conditionally load scripts        | 🟡 MEDIUM   |
| [PERF-030](./performance/PERF-030.md) | AVIF not served — Shopify CDN not leveraged                       | 🟡 MEDIUM   |

---

## C3 — JavaScript: Architecture & Lifecycle `28 rules`

| ID                               | Rule                                                                     | Severity    |
| -------------------------------- | ------------------------------------------------------------------------ | ----------- |
| [JS-001](./javascript/JS-001.md) | `disconnectedCallback` missing in Web Component                          | 🟠 HIGH     |
| [JS-002](./javascript/JS-002.md) | Event listeners added in `connectedCallback` without cleanup             | 🟠 HIGH     |
| [JS-003](./javascript/JS-003.md) | DOM query in Web Component `constructor()`                               | 🟠 HIGH     |
| [JS-004](./javascript/JS-004.md) | `customElements.define()` without `customElements.get()` guard           | 🟠 HIGH     |
| [JS-005](./javascript/JS-005.md) | `AbortController` absent for fetch cleanup                               | 🟡 MEDIUM   |
| [JS-006](./javascript/JS-006.md) | `AbortController` not used for event listener cleanup                    | 🟡 MEDIUM   |
| [JS-007](./javascript/JS-007.md) | `data.quantity` from `cart/add.js` used as cart total                    | 🟠 HIGH     |
| [JS-008](./javascript/JS-008.md) | Concurrent `cart/add.js` requests without Promise queue                  | 🟠 HIGH     |
| [JS-009](./javascript/JS-009.md) | `cart/update.js` instead of `cart/change.js` for line items              | 🟡 MEDIUM   |
| [JS-010](./javascript/JS-010.md) | Section Rendering API not used after cart mutations                      | 🟡 MEDIUM   |
| [JS-011](./javascript/JS-011.md) | Liquid variable inside `{% javascript %}` tag                            | 🟠 HIGH     |
| [JS-012](./javascript/JS-012.md) | Liquid data via inline script instead of data attributes                 | 🟡 MEDIUM   |
| [JS-013](./javascript/JS-013.md) | `localStorage` accessed without try/catch                                | 🟠 HIGH     |
| [JS-014](./javascript/JS-014.md) | `setInterval` for polling instead of `MutationObserver`                  | 🟡 MEDIUM   |
| [JS-015](./javascript/JS-015.md) | `requestAnimationFrame` absent for visual DOM updates                    | 🟡 MEDIUM   |
| [JS-016](./javascript/JS-016.md) | `innerHTML +=` to append cart items                                      | 🟠 HIGH     |
| [JS-017](./javascript/JS-017.md) | `WeakMap` absent for private component state                             | 🟢 LOW      |
| [JS-018](./javascript/JS-018.md) | Custom events without `bubbles: true` and `composed: true`               | 🟡 MEDIUM   |
| [JS-019](./javascript/JS-019.md) | Pubsub subscription not cleaned up in `disconnectedCallback`             | 🟠 HIGH     |
| [JS-020](./javascript/JS-020.md) | `document.write()` detected                                              | 🔴 CRITICAL |
| [JS-021](./javascript/JS-021.md) | `window.addEventListener` without `{ passive: true }` on scroll/touch    | 🟡 MEDIUM   |
| [JS-022](./javascript/JS-022.md) | `IntersectionObserver` without `.disconnect()` in `disconnectedCallback` | 🟠 HIGH     |
| [JS-023](./javascript/JS-023.md) | `cart/add.js` HTTP 422 response not handled                              | 🟠 HIGH     |
| [JS-024](./javascript/JS-024.md) | Shopify `routes.root` absent from global scope                           | 🟠 HIGH     |
| [JS-025](./javascript/JS-025.md) | `_shopify_country` cookie accessed — removed August 15, 2025             | 🔴 CRITICAL |
| [JS-026](./javascript/JS-026.md) | `ResizeObserver` without `.disconnect()`                                 | 🟠 HIGH     |
| [JS-027](./javascript/JS-027.md) | `Swiper.js` or `Slick` loaded as external dependency                     | 🟡 MEDIUM   |
| [JS-028](./javascript/JS-028.md) | Layout thrashing — DOM read followed by write in loop                    | 🟠 HIGH     |

---

## C4 — Security `23 rules`

| ID                                 | Rule                                                                  | Severity    |
| ---------------------------------- | --------------------------------------------------------------------- | ----------- |
| [XSS-001](./security/XSS-001.md)   | Liquid variable in `<script>` without `\| json` filter                | 🔴 CRITICAL |
| [XSS-002](./security/XSS-002.md)   | `innerHTML` with unescaped Liquid output                              | 🔴 CRITICAL |
| [XSS-003](./security/XSS-003.md)   | `eval()` in theme JavaScript                                          | 🔴 CRITICAL |
| [XSS-004](./security/XSS-004.md)   | `new Function(string)` — functional equivalent of eval()              | 🔴 CRITICAL |
| [XSS-005](./security/XSS-005.md)   | `setTimeout(string, delay)` — string argument evaluated as code       | 🟠 HIGH     |
| [XSS-006](./security/XSS-006.md)   | `outerHTML` assignment with external data                             | 🟠 HIGH     |
| [XSS-007](./security/XSS-007.md)   | URL parameter reflected into DOM via `innerHTML`                      | 🔴 CRITICAL |
| [XSS-008](./security/XSS-008.md)   | `document.write()` with dynamic content                               | 🔴 CRITICAL |
| [XSS-009](./security/XSS-009.md)   | `DOMPurify` absent before `innerHTML` of rich text metafield          | 🟠 HIGH     |
| [XSS-010](./security/XSS-010.md)   | `postMessage` listener without `event.origin` validation              | 🟠 HIGH     |
| [XSS-011](./security/XSS-011.md)   | `location.hash` reflected into DOM without sanitization               | 🔴 CRITICAL |
| [XSS-012](./security/XSS-012.md)   | `srcdoc` iframe with unsanitized data                                 | 🟠 HIGH     |
| [CSRF-001](./security/CSRF-001.md) | POST form without `authenticity_token`                                | 🟠 HIGH     |
| [CSRF-002](./security/CSRF-002.md) | Custom HTML form bypassing `{% form %}` tag                           | 🟠 HIGH     |
| [SEC-001](./security/SEC-001.md)   | `cart.token` exposed in JavaScript global scope                       | 🔴 CRITICAL |
| [SEC-002](./security/SEC-002.md)   | HMAC validation absent on App Proxy requests                          | 🔴 CRITICAL |
| [SEC-003](./security/SEC-003.md)   | Webhook HMAC not verified with `timingSafeEqual`                      | 🟠 HIGH     |
| [SEC-004](./security/SEC-004.md)   | CORS wildcard `Access-Control-Allow-Origin: *` on sensitive endpoints | 🟠 HIGH     |
| [SEC-005](./security/SEC-005.md)   | App embed script without `async` or `defer`                           | 🟠 HIGH     |
| [SEC-006](./security/SEC-006.md)   | Customer email in JavaScript without encoding                         | 🟠 HIGH     |
| [SEC-007](./security/SEC-007.md)   | `Content-Security-Policy` header absent                               | 🟠 HIGH     |
| [SEC-008](./security/SEC-008.md)   | `X-Frame-Options` absent on checkout pages                            | 🟠 HIGH     |
| [SEC-009](./security/SEC-009.md)   | Private metafield namespace exposed in public Liquid output           | 🟡 MEDIUM   |
| [SEC-010](./security/SEC-010.md)   | `selling_plan` not validated before cart submission                   | 🟠 HIGH     |
| [SEC-011](./security/SEC-011.md)   | `customer.id` exposed in JavaScript without protection                | 🟡 MEDIUM   |
| [SEC-012](./security/SEC-012.md)   | Redirect URL not validated in login/logout flow                       | 🟠 HIGH     |
| [SEC-013](./security/SEC-013.md)   | `Content-Type` header absent on custom JSON endpoints                 | 🟡 MEDIUM   |

---

## C5 — Accessibility (EAA 2025 / WCAG 2.2 AA) `35 rules`

> **Legal note:** EAA is enforceable since **June 28, 2025** in all 27 EU member states. Fines up to **€100,000** per infraction (Netherlands: up to €900,000 or 10% annual revenue).

| ID                                      | Rule                                                                | Severity    |
| --------------------------------------- | ------------------------------------------------------------------- | ----------- |
| [A11Y-001](./accessibility/A11Y-001.md) | Modal/drawer without focus trap                                     | 🔴 CRITICAL |
| [A11Y-002](./accessibility/A11Y-002.md) | `outline: none` without `:focus-visible` replacement                | 🔴 CRITICAL |
| [A11Y-003](./accessibility/A11Y-003.md) | Focus not returned to trigger after modal close                     | 🟠 HIGH     |
| [A11Y-004](./accessibility/A11Y-004.md) | Color contrast ratio < 4.5:1 for normal text                        | 🔴 CRITICAL |
| [A11Y-005](./accessibility/A11Y-005.md) | Color contrast < 3:1 for large text (18pt+)                         | 🟠 HIGH     |
| [A11Y-006](./accessibility/A11Y-006.md) | UI component contrast < 3:1 (buttons, inputs, icons)                | 🟠 HIGH     |
| [A11Y-007](./accessibility/A11Y-007.md) | `aria-expanded` not toggled on menu/accordion open/close            | 🟠 HIGH     |
| [A11Y-008](./accessibility/A11Y-008.md) | `aria-live` region absent on cart counter update                    | 🟠 HIGH     |
| [A11Y-009](./accessibility/A11Y-009.md) | `role="dialog"` without `aria-modal="true"`                         | 🟠 HIGH     |
| [A11Y-010](./accessibility/A11Y-010.md) | `aria-controls` pointing to non-existent ID                         | 🟠 HIGH     |
| [A11Y-011](./accessibility/A11Y-011.md) | `alt` attribute missing on product images                           | 🔴 CRITICAL |
| [A11Y-012](./accessibility/A11Y-012.md) | Decorative image with non-empty `alt`                               | 🟡 MEDIUM   |
| [A11Y-013](./accessibility/A11Y-013.md) | Icon-only button without accessible label                           | 🔴 CRITICAL |
| [A11Y-014](./accessibility/A11Y-014.md) | `placeholder` as only label for form input                          | 🟠 HIGH     |
| [A11Y-015](./accessibility/A11Y-015.md) | iOS scroll lock without appropriate `touch-action`                  | 🟠 HIGH     |
| [A11Y-016](./accessibility/A11Y-016.md) | Skip navigation link absent or non-functional                       | 🟠 HIGH     |
| [A11Y-017](./accessibility/A11Y-017.md) | `prefers-reduced-motion` not respected in animations                | 🟠 HIGH     |
| [A11Y-018](./accessibility/A11Y-018.md) | Heading hierarchy violation (H3 without H2 parent)                  | 🟠 HIGH     |
| [A11Y-019](./accessibility/A11Y-019.md) | Interactive element < 24x24px — WCAG 2.5.8                          | 🟠 HIGH     |
| [A11Y-020](./accessibility/A11Y-020.md) | `lang` attribute missing on `<html>` tag                            | 🟠 HIGH     |
| [A11Y-021](./accessibility/A11Y-021.md) | Error message not associated with field via `aria-describedby`      | 🟠 HIGH     |
| [A11Y-022](./accessibility/A11Y-022.md) | Color as only visual indicator (required field, error state)        | 🟠 HIGH     |
| [A11Y-023](./accessibility/A11Y-023.md) | **EAA 2025** — Non-semantic navigation (`<div>` instead of `<nav>`) | 🔴 CRITICAL |
| [A11Y-024](./accessibility/A11Y-024.md) | **EAA 2025** — Checkout form without `autocomplete` attributes      | 🟠 HIGH     |
| [A11Y-025](./accessibility/A11Y-025.md) | **EAA 2025** — Video content without captions                       | 🟠 HIGH     |
| [A11Y-026](./accessibility/A11Y-026.md) | FAQ using `<details>/<summary>` closed by default                   | 🟠 HIGH     |
| [A11Y-027](./accessibility/A11Y-027.md) | `tabindex` > 0                                                      | 🟡 MEDIUM   |
| [A11Y-028](./accessibility/A11Y-028.md) | Carousel auto-play without pause control                            | 🟠 HIGH     |
| [A11Y-029](./accessibility/A11Y-029.md) | **EAA 2025** — Accessibility statement absent                       | 🟠 HIGH     |
| [A11Y-030](./accessibility/A11Y-030.md) | **EAA 2025** — CAPTCHA without accessible alternative               | 🟠 HIGH     |
| [A11Y-031](./accessibility/A11Y-031.md) | Variant selector without screen reader announcement                 | 🟠 HIGH     |
| [A11Y-032](./accessibility/A11Y-032.md) | Add-to-cart button without `aria-disabled` during request           | 🟡 MEDIUM   |
| [A11Y-033](./accessibility/A11Y-033.md) | Cart notification without `role="status"` or `aria-live="polite"`   | 🟠 HIGH     |
| [A11Y-034](./accessibility/A11Y-034.md) | Strikethrough price without explicit accessible alternative         | 🟡 MEDIUM   |
| [A11Y-035](./accessibility/A11Y-035.md) | Collection filters without `aria-live` update                       | 🟠 HIGH     |

---

## C6 — SEO & Structured Data `30 rules`

| ID                          | Rule                                                                | Severity    |
| --------------------------- | ------------------------------------------------------------------- | ----------- |
| [SEO-001](./seo/SEO-001.md) | Duplicated `\| Syphio \| Syphio` suffix in `<title>` — template bug | 🟠 HIGH     |
| [SEO-002](./seo/SEO-002.md) | `<title>` tag missing                                               | 🔴 CRITICAL |
| [SEO-003](./seo/SEO-003.md) | `<meta name="description">` missing                                 | 🟠 HIGH     |
| [SEO-004](./seo/SEO-004.md) | H1 missing or without primary keyword                               | 🟠 HIGH     |
| [SEO-005](./seo/SEO-005.md) | Multiple H1 elements on a single page                               | 🟠 HIGH     |
| [SEO-006](./seo/SEO-006.md) | H-structure violation (H3 before H2)                                | 🟡 MEDIUM   |
| [SEO-007](./seo/SEO-007.md) | `<link rel="canonical">` missing                                    | 🟠 HIGH     |
| [SEO-008](./seo/SEO-008.md) | Canonical pointing to temporary domain instead of final domain      | 🔴 CRITICAL |
| [SEO-009](./seo/SEO-009.md) | Canonical missing on paginated collection pages                     | 🟠 HIGH     |
| [SEO-010](./seo/SEO-010.md) | JSON-LD `@graph` duplicated across multiple `<script>` blocks       | 🟠 HIGH     |
| [SEO-011](./seo/SEO-011.md) | JSON-LD `Product` without `offers.availability`                     | 🟠 HIGH     |
| [SEO-012](./seo/SEO-012.md) | JSON-LD `Product` with `price` without `priceCurrency`              | 🟠 HIGH     |
| [SEO-013](./seo/SEO-013.md) | `BreadcrumbList` JSON-LD missing                                    | 🟡 MEDIUM   |
| [SEO-014](./seo/SEO-014.md) | `FAQPage` JSON-LD missing on FAQ content                            | 🟠 HIGH     |
| [SEO-015](./seo/SEO-015.md) | FAQ HTML with `<details>/<summary>` without `open` attribute        | 🟠 HIGH     |
| [SEO-016](./seo/SEO-016.md) | `<img>` without `alt` on content images                             | 🟠 HIGH     |
| [SEO-017](./seo/SEO-017.md) | `robots.txt` missing                                                | 🔴 CRITICAL |
| [SEO-018](./seo/SEO-018.md) | `sitemap.xml` missing                                               | 🔴 CRITICAL |
| [SEO-019](./seo/SEO-019.md) | Legal pages not excluded from sitemap                               | 🟡 MEDIUM   |
| [SEO-020](./seo/SEO-020.md) | `/app/*` routes not disallowed in robots.txt                        | 🟠 HIGH     |
| [SEO-021](./seo/SEO-021.md) | `og:image` missing — incomplete Open Graph                          | 🟠 HIGH     |
| [SEO-022](./seo/SEO-022.md) | `og:image` dimensions ≠ 1200x630                                    | 🟡 MEDIUM   |
| [SEO-023](./seo/SEO-023.md) | `hreflang` absent on multi-market stores                            | 🟠 HIGH     |
| [SEO-024](./seo/SEO-024.md) | `hreflang` self-referencing entry missing                           | 🟡 MEDIUM   |
| [SEO-025](./seo/SEO-025.md) | Product variant URLs without canonical to parent                    | 🟠 HIGH     |
| [SEO-026](./seo/SEO-026.md) | `<meta robots="noindex">` accidentally present in production        | 🔴 CRITICAL |
| [SEO-027](./seo/SEO-027.md) | JSON-LD `TechArticle` without `proficiencyLevel` field              | 🟢 LOW      |
| [SEO-028](./seo/SEO-028.md) | `SoftwareApplication` JSON-LD absent on homepage                    | 🟡 MEDIUM   |
| [SEO-029](./seo/SEO-029.md) | `WebSite` JSON-LD with `SearchAction` absent                        | 🟢 LOW      |
| [SEO-030](./seo/SEO-030.md) | `/blog` page rendered CSR — `Loading…` in HTML source               | 🟠 HIGH     |

---

## C7 — Schema JSON & Section Architecture `17 rules`

| ID                                   | Rule                                                                 | Severity    |
| ------------------------------------ | -------------------------------------------------------------------- | ----------- |
| [SCHEMA-001](./schema/SCHEMA-001.md) | Section schema missing `name` field                                  | 🟠 HIGH     |
| [SCHEMA-002](./schema/SCHEMA-002.md) | `presets` array empty or missing                                     | 🟠 HIGH     |
| [SCHEMA-003](./schema/SCHEMA-003.md) | Block `type` containing spaces or uppercase                          | 🔴 CRITICAL |
| [SCHEMA-004](./schema/SCHEMA-004.md) | Setting `id` containing reserved characters                          | 🔴 CRITICAL |
| [SCHEMA-005](./schema/SCHEMA-005.md) | `content_for_index` missing in index template                        | 🔴 CRITICAL |
| [SCHEMA-006](./schema/SCHEMA-006.md) | Hardcoded HTML `id` attribute in section                             | 🟠 HIGH     |
| [SCHEMA-007](./schema/SCHEMA-007.md) | `section.id` not used for unique element IDs                         | 🟠 HIGH     |
| [SCHEMA-008](./schema/SCHEMA-008.md) | Block `limit` not defined on unbounded block type                    | 🟠 HIGH     |
| [SCHEMA-009](./schema/SCHEMA-009.md) | `type: "product"` setting replaced by handle + `all_products[]`      | 🔴 CRITICAL |
| [SCHEMA-010](./schema/SCHEMA-010.md) | `type: "collection"` setting replaced by `collections.all` iteration | 🔴 CRITICAL |
| [SCHEMA-011](./schema/SCHEMA-011.md) | Section without `{% schema %}` tag                                   | 🟡 MEDIUM   |
| [SCHEMA-012](./schema/SCHEMA-012.md) | `max_blocks` not defined — no upper limit                            | 🟡 MEDIUM   |
| [SCHEMA-013](./schema/SCHEMA-013.md) | **Horizon** — CSS not scoped to block ID                             | 🟠 HIGH     |
| [SCHEMA-014](./schema/SCHEMA-014.md) | **Horizon** — nested blocks > 8 levels                               | 🔴 CRITICAL |
| [SCHEMA-015](./schema/SCHEMA-015.md) | **Horizon** — direct modification of core theme files                | 🟠 HIGH     |
| [SCHEMA-016](./schema/SCHEMA-016.md) | **Horizon** — external JS library loaded (Swiper, Slick)             | 🟡 MEDIUM   |
| [SCHEMA-017](./schema/SCHEMA-017.md) | `type: "hidden"` setting used for sensitive data                     | 🟡 MEDIUM   |

---

## C8 — Metafields & Metaobjects `10 rules`

| ID                                   | Rule                                                             | Severity    |
| ------------------------------------ | ---------------------------------------------------------------- | ----------- |
| [META-001](./metafields/META-001.md) | Metafield accessed without `.value`                              | 🔴 CRITICAL |
| [META-002](./metafields/META-002.md) | Metafield accessed without nil check                             | 🟠 HIGH     |
| [META-003](./metafields/META-003.md) | `rich_text` metafield without `\| metafield_tag` filter          | 🟠 HIGH     |
| [META-004](./metafields/META-004.md) | `image_reference` metafield without nil check before `image_url` | 🟠 HIGH     |
| [META-005](./metafields/META-005.md) | `list.*` metafield not iterated with `{% for %}`                 | 🟠 HIGH     |
| [META-006](./metafields/META-006.md) | Handle string + `all_products[]` instead of `product_reference`  | 🟡 MEDIUM   |
| [META-007](./metafields/META-007.md) | Private metafield namespace exposed in public Liquid output      | 🟢 LOW      |
| [META-008](./metafields/META-008.md) | `metaobject` fields accessed without existence check             | 🟠 HIGH     |
| [META-009](./metafields/META-009.md) | `json` metafield not parsed with `\| parse_json`                 | 🟠 HIGH     |
| [META-010](./metafields/META-010.md) | `color` metafield used without format validation                 | 🟡 MEDIUM   |

---

## C9 — Markets, i18n & Localisation `9 rules`

| ID                              | Rule                                                                     | Severity    |
| ------------------------------- | ------------------------------------------------------------------------ | ----------- |
| [MKT-001](./markets/MKT-001.md) | Hardcoded AJAX endpoints without `routes.root` prefix                    | 🔴 CRITICAL |
| [MKT-002](./markets/MKT-002.md) | `{% form 'localization' %}` absent on locale selector                    | 🟠 HIGH     |
| [MKT-003](./markets/MKT-003.md) | CSS `direction: ltr` hardcoded without RTL override                      | 🟠 HIGH     |
| [MKT-004](./markets/MKT-004.md) | `margin-left` / `padding-left` instead of CSS logical properties         | 🟡 MEDIUM   |
| [MKT-005](./markets/MKT-005.md) | `{{ localization.language.iso_code }}` not used as `<html lang>`         | 🟠 HIGH     |
| [MKT-006](./markets/MKT-006.md) | Price formatted with hardcoded currency symbol instead of `money` filter | 🟠 HIGH     |
| [MKT-007](./markets/MKT-007.md) | User-visible string without `t:` translation key                         | 🟠 HIGH     |
| [MKT-008](./markets/MKT-008.md) | `localization.market` not used for market-conditional content            | 🟡 MEDIUM   |
| [MKT-009](./markets/MKT-009.md) | Post-purchase redirect URL without locale parameter                      | 🟡 MEDIUM   |

---

## C10 — Checkout, Selling Plans & Subscriptions `9 rules`

| ID                               | Rule                                                                       | Severity    |
| -------------------------------- | -------------------------------------------------------------------------- | ----------- |
| [CHK-001](./checkout/CHK-001.md) | `checkout.liquid` present after deprecation deadline (August 2024)         | 🔴 CRITICAL |
| [CHK-002](./checkout/CHK-002.md) | Third-party pixels injected via `checkout.liquid` instead of Custom Pixels | 🔴 CRITICAL |
| [CHK-003](./checkout/CHK-003.md) | `selling_plan` parameter absent in `cart/add.js` for subscription products | 🔴 CRITICAL |
| [CHK-004](./checkout/CHK-004.md) | Selling plan UI not reflecting `selling_plan_groups` Liquid object         | 🟠 HIGH     |
| [CHK-005](./checkout/CHK-005.md) | `checkout_url` instead of `routes.checkout_url`                            | 🟠 HIGH     |
| [CHK-006](./checkout/CHK-006.md) | Custom Pixels not migrated before August 2025 deadline                     | 🔴 CRITICAL |
| [CHK-007](./checkout/CHK-007.md) | GTM not migrated to Custom Pixel for checkout                              | 🔴 CRITICAL |
| [CHK-008](./checkout/CHK-008.md) | `order_placed` event absent in Custom Pixel                                | 🟠 HIGH     |
| [CHK-009](./checkout/CHK-009.md) | Checkout UI Extension not migrated from script injection                   | 🔴 CRITICAL |

---

## C11 — Hydrogen & Headless `10 rules`

| ID                               | Rule                                                           | Severity    |
| -------------------------------- | -------------------------------------------------------------- | ----------- |
| [HYD-001](./hydrogen/HYD-001.md) | `loader` functions without `Promise.all` for parallel fetches  | 🟠 HIGH     |
| [HYD-002](./hydrogen/HYD-002.md) | Customer Storefront API data without `cache: no-store`         | 🔴 CRITICAL |
| [HYD-003](./hydrogen/HYD-003.md) | Cart queries without `buyerIdentity` for international pricing | 🟠 HIGH     |
| [HYD-004](./hydrogen/HYD-004.md) | Storefront API pagination without `endCursor` validation       | 🟠 HIGH     |
| [HYD-005](./hydrogen/HYD-005.md) | `useLoaderData` result not typed with Storefront API types     | 🟡 MEDIUM   |
| [HYD-006](./hydrogen/HYD-006.md) | Cache strategy absent on product/collection loaders            | 🔴 CRITICAL |
| [HYD-007](./hydrogen/HYD-007.md) | Customer Account API session not validated on protected routes | 🔴 CRITICAL |
| [HYD-008](./hydrogen/HYD-008.md) | Hydrogen `cart` not invalidated after mutation                 | 🟠 HIGH     |
| [HYD-009](./hydrogen/HYD-009.md) | `await` waterfall in loader — avoidable sequential fetches     | 🟠 HIGH     |
| [HYD-010](./hydrogen/HYD-010.md) | `storefrontApiClient` without retry logic                      | 🟡 MEDIUM   |

---

## C12 — Theme Check & CI/CD `12 rules`

| ID                          | Rule                                                              | Severity    |
| --------------------------- | ----------------------------------------------------------------- | ----------- |
| [CI-001](./ci-cd/CI-001.md) | Theme Check not configured in CI/CD pipeline                      | 🟠 HIGH     |
| [CI-002](./ci-cd/CI-002.md) | `HardcodedRoutes` — `/cart`, `/products` hardcoded                | 🟠 HIGH     |
| [CI-003](./ci-cd/CI-003.md) | `MissingAsset` — asset referenced but absent from `/assets/`      | 🟠 HIGH     |
| [CI-004](./ci-cd/CI-004.md) | `TranslationKeyExists` — `t:` key absent from default locale JSON | 🟠 HIGH     |
| [CI-005](./ci-cd/CI-005.md) | `DeprecatedTag` — `{% include %}` still present                   | 🟠 HIGH     |
| [CI-006](./ci-cd/CI-006.md) | `ParserBlockingJavaScript` — scripts without `defer`/`async`      | 🟠 HIGH     |
| [CI-007](./ci-cd/CI-007.md) | `content_for_header` not first in `<head>`                        | 🟠 HIGH     |
| [CI-008](./ci-cd/CI-008.md) | `{% schema %}` JSON invalid — syntax error                        | 🔴 CRITICAL |
| [CI-009](./ci-cd/CI-009.md) | `ObsoleteFilter` — `img_url` still used                           | 🟡 MEDIUM   |
| [CI-010](./ci-cd/CI-010.md) | `RemoteAsset` — asset loaded from external non-Shopify CDN        | 🟡 MEDIUM   |
| [CI-011](./ci-cd/CI-011.md) | `DeprecatedFilter` — deprecated Liquid filters used               | 🟡 MEDIUM   |
| [CI-012](./ci-cd/CI-012.md) | `LiquidHTMLSyntaxError` — invalid HTML in Liquid file             | 🟠 HIGH     |

---

## C13 — Klaviyo & Third-Party Integrations `8 rules`

| ID                                   | Rule                                                             | Severity    |
| ------------------------------------ | ---------------------------------------------------------------- | ----------- |
| [KLV-001](./integrations/KLV-001.md) | Klaviyo v3 — missing `$` prefix on special properties            | 🔴 CRITICAL |
| [KLV-002](./integrations/KLV-002.md) | Klaviyo script blocking LCP — without `async`                    | 🟠 HIGH     |
| [KLV-003](./integrations/KLV-003.md) | `klaviyo.push` instead of `klaviyo.track` in API v3              | 🟠 HIGH     |
| [KLV-004](./integrations/KLV-004.md) | GTM/GA4 not migrated to Custom Pixels for checkout               | 🔴 CRITICAL |
| [KLV-005](./integrations/KLV-005.md) | Meta Pixel `fbq` injected via `<script>` instead of Custom Pixel | 🔴 CRITICAL |
| [INT-001](./integrations/INT-001.md) | App embed script loaded on all pages without condition           | 🟠 HIGH     |
| [INT-002](./integrations/INT-002.md) | `_shopify_country` cookie accessed — removed August 15, 2025     | 🔴 CRITICAL |
| [INT-003](./integrations/INT-003.md) | Shopify Pixel API `analytics.subscribe()` not used               | 🟡 MEDIUM   |

---

## C14 — GDPR & Data Privacy `5 rules`

| ID                             | Rule                                                          | Severity    |
| ------------------------------ | ------------------------------------------------------------- | ----------- |
| [GDPR-001](./gdpr/GDPR-001.md) | Cookie consent absent before loading tracking scripts         | 🔴 CRITICAL |
| [GDPR-002](./gdpr/GDPR-002.md) | `customer.email` in JavaScript without documented legal basis | 🟠 HIGH     |
| [GDPR-003](./gdpr/GDPR-003.md) | Analytics scripts loaded before user consent                  | 🔴 CRITICAL |
| [GDPR-004](./gdpr/GDPR-004.md) | Order data exposed in client-side JavaScript                  | 🟠 HIGH     |
| [GDPR-005](./gdpr/GDPR-005.md) | Privacy policy link absent from footer                        | 🟡 MEDIUM   |

---

## Summary

| Category                      | Rules   | 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low |
| ----------------------------- | ------- | ----------- | ------- | --------- | ------ |
| Liquid Syntax & Deprecated    | 25      | 6           | 11      | 7         | 1      |
| Performance & Core Web Vitals | 30      | 4           | 18      | 7         | 1      |
| JavaScript Architecture       | 28      | 2           | 19      | 6         | 1      |
| Security                      | 23      | 10          | 11      | 2         | 0      |
| Accessibility — EAA 2025      | 35      | 5           | 25      | 4         | 1      |
| SEO & Structured Data         | 30      | 5           | 18      | 6         | 1      |
| Schema & Section Architecture | 17      | 5           | 8       | 3         | 1      |
| Metafields & Metaobjects      | 10      | 1           | 7       | 2         | 0      |
| Markets & i18n                | 9       | 1           | 6       | 2         | 0      |
| Checkout & Selling Plans      | 9       | 5           | 4       | 0         | 0      |
| Hydrogen & Headless           | 10      | 3           | 6       | 1         | 0      |
| Theme Check & CI/CD           | 12      | 1           | 8       | 3         | 0      |
| Klaviyo & Integrations        | 8       | 4           | 3       | 1         | 0      |
| GDPR & Data Privacy           | 5       | 2           | 2       | 1         | 0      |
| **TOTAL**                     | **251** | **54**      | **146** | **45**    | **6**  |

---

_Syphio Rule Database v2.0 · March 2026_
_[syphio.io](https://syphio.io) · Built by [Ludovic Fournier](https://github.com/ludo62) · France_
