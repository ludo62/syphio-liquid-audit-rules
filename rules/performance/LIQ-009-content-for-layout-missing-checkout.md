# LIQ-009 | `{{ content_for_layout }}` missing in checkout.liquid

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** `layout/checkout.liquid` (Shopify Plus stores only)

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{{ content_for_layout }}` in `checkout.liquid` is the render target for Shopify's checkout page content engine. On each checkout step — customer information, shipping method, payment, order confirmation — Shopify's platform compiles the appropriate step template and delivers its output through this variable. The output includes the order summary panel, the customer information form, the shipping address fields, the payment method selection UI, and the checkout progress indicator. None of this content is available through any other Liquid variable; it is injected exclusively via `content_for_layout`. Without this variable in the layout, the checkout renders as a structurally valid HTML page with header and footer components, but a completely blank content area — the merchant's customers land on a checkout page with no form fields, no order summary, and no way to complete the purchase.

`{{ content_for_layout }}` in `checkout.liquid` also serves as the injection point for all Checkout UI extensions registered by installed apps. Extensions providing upsell widgets (post-purchase offers, gift message fields, loyalty point redemption displays, address validation prompts) are all delivered through the same variable. A merchant with 5 checkout apps and a missing `content_for_layout` loses not only the checkout UI but all app extensions simultaneously — no error is reported at the app level, the checkout simply does not render. Shopify Plus merchants upgrading from legacy Checkout 1.0 themes to Checkout Extensibility themes encounter this issue when migrating layout files, as the variable name is the same as in `theme.liquid` but the consequences are more severe (checkout failure = direct revenue loss).

The `checkout.liquid` layout file is exclusive to Shopify Plus merchants. Standard Shopify plans use a platform-managed checkout layout that does not support customization. This means the bug only manifests on Plus plans, it is not caught by testing on a development store unless the store is a Plus development store, and it affects only the highest-revenue storefronts in the Shopify ecosystem. The bug's impact is proportional to the merchant's checkout conversion volume — for a merchant processing $1M/month, even 30 minutes of blank checkout represents measurable revenue loss.

---

## Detection Logic (AST)

1. **Scope detection to `checkout.liquid`.** The rule applies exclusively to files named `checkout.liquid` in the `/layout/` directory. The walker identifies the file path and skips all other files. Theme.liquid and custom layout files are covered by a separate rule (LIQ-008 for `content_for_header`); this rule is specific to the checkout layout.

2. **Identify `LiquidVariableOutput` nodes where the expression resolves to `content_for_layout`.** Within `checkout.liquid`, the walker traverses all `LiquidVariableOutput` nodes and inspects the `expression` property. A `LiquidVariableOutput` where `expression.name === "content_for_layout"` is the target node. The variable must not be filtered — `{{ content_for_layout | some_filter }}` is invalid and would itself be flagged.

3. **Count occurrences and verify exactly one.** If `count === 0`, the rule fires as MISSING with CRITICAL severity. If `count > 1`, the rule fires as DUPLICATED with HIGH severity — duplication causes checkout UI components to render twice, producing a broken layout with doubled form fields. The checkout API would only process the first form submission, but the doubled UI confuses customers.

4. **Verify placement within the `<main>` or `<body>` element.** The `LiquidVariableOutput[content_for_layout]` must appear within the page body — specifically inside an `HtmlElement[tag=main]` if present, or within `HtmlElement[tag=body]` at minimum. If found inside `HtmlElement[tag=head]`, the placement is a critical error — checkout HTML inserted into `<head>` produces malformed markup that browsers cannot render.

**Why regex fails here:** A regex counting `\{\{\s*content_for_layout\s*\}\}` occurrences in `checkout.liquid` would match the string even if it appears inside a `{% comment %}` block (documentation comments are common in Plus theme files explaining variable purposes). The AST walker excludes `LiquidTag[comment]` descendants from the occurrence count, ensuring only live output nodes are counted. A regex also cannot determine whether the variable appears inside `<head>` versus `<body>` without parsing the surrounding HTML structure.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: content_for_layout missing from checkout.liquid.

  Failure mode 1: The checkout body renders completely blank.
  No customer information form, no shipping selector, no payment form.
  Customers cannot complete purchases. Conversion rate drops to 0%.

  Failure mode 2: All Checkout UI extensions (upsell widgets, gift messages,
  loyalty redemption, address validation) fail to load — they are delivered
  via content_for_layout. Apps show no error; they simply do not appear.

  Failure mode 3: Order confirmation page also blank — customers cannot
  confirm their order was placed. Support ticket volume spikes immediately.

  The developer has placed custom HTML in the content area, likely attempting
  to manually reconstruct checkout UI, which cannot work — Shopify's checkout
  forms require server-side session tokens and CSRF protection that only
  the platform can inject via content_for_layout.
{%- endcomment -%}

<!DOCTYPE html>
<html lang="{{ locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    {{ content_for_header }}
    <title>{{ page_title }} — {{ shop.name }}</title>
    {{ 'checkout.css' | asset_url | stylesheet_tag }}
  </head>
  <body>
    <header class="checkout-header">
      <a href="{{ shop.url }}">
        {% if shop.logo %}
          {{
            shop.logo
            | image_url: width: 200
            | image_tag: alt: shop.name, loading: 'eager'
          }}
        {% else %}
          <span class="shop-name">{{ shop.name }}</span>
        {% endif %}
      </a>
    </header>

    <main class="checkout-main">
      {%- comment -%}
        Developer has attempted to manually render checkout content here.
        This does not work — checkout step content is only available via
        content_for_layout. The manual HTML below renders but has no
        server-generated form fields, session tokens, or checkout state.
      {%- endcomment -%}
      <div class="checkout-placeholder">
        <p>Loading checkout...</p>
      </div>
    </main>

    <footer class="checkout-footer">
      <p>{{ 'checkout.general.privacy_policy' | t }}</p>
    </footer>
  </body>
</html>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add {{ content_for_layout }} inside <main> in checkout.liquid.

  Change 1: content_for_layout placed inside <main> — renders the active
  checkout step: customer info, shipping, payment, or order confirmation.

  Change 2: Remove manual placeholder HTML — it is never needed and conflicts
  with the platform-rendered content.

  Change 3: main element retains role="main" and appropriate id for
  accessibility — Checkout UI extensions use the landmark for focus management.

  Verification: After adding content_for_layout, test all checkout steps
  on a Plus development store: customer info, shipping, payment, order status.
  Each step must render its platform UI correctly.
{%- endcomment -%}

<!DOCTYPE html>
<html lang="{{ locale.iso_code }}">
  <head>
    <meta charset="UTF-8">
    {%- comment -%}
      content_for_header: required in checkout.liquid as in theme.liquid.
      Injects checkout-specific scripts, app bridge, analytics hooks.
    {%- endcomment -%}
    {{ content_for_header }}
    <title>{{ page_title }} — {{ shop.name }}</title>
    {{ 'checkout.css' | asset_url | stylesheet_tag }}
  </head>
  <body>
    <header class="checkout-header" role="banner">
      <a href="{{ shop.url }}" aria-label="{{ shop.name }}">
        {% if shop.logo %}
          {{
            shop.logo
            | image_url: width: 200
            | image_tag: alt: shop.name, loading: 'eager'
          }}
        {% else %}
          <span class="shop-name">{{ shop.name }}</span>
        {% endif %}
      </a>
    </header>

    {%- comment -%}
      content_for_layout: renders the active checkout step template.
      Also delivers all Checkout UI extensions registered by installed apps.
      Must appear exactly once, inside <main>, with no filters applied.
    {%- endcomment -%}
    <main id="checkout-main" class="checkout-main" role="main" tabindex="-1">
      {{ content_for_layout }}
    </main>

    <footer class="checkout-footer" role="contentinfo">
      <ul class="checkout-footer__links">
        <li>
          <a href="{{ shop.url }}/policies/privacy-policy">
            {{ 'checkout.general.privacy_policy' | t }}
          </a>
        </li>
      </ul>
    </footer>
  </body>
</html>
```

---

## References

- [Shopify — `content_for_layout` global variable](https://shopify.dev/docs/api/liquid/objects/content_for_layout)
- [Shopify — Checkout customization (Plus)](https://shopify.dev/docs/storefronts/themes/architecture/layouts/checkout-liquid)
- [Shopify — Checkout UI extensions](https://shopify.dev/docs/api/checkout-ui-extensions)
- [Shopify Theme Check — RequiredLayoutThemeObject](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/required-layout-theme-object)
