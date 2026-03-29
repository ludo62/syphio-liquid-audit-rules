# LIQ-002 | `img_url` filter deprecated — replace with `image_url`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Any template outputting product images, collection images, article images, or metafield images via `| img_url`

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`img_url` is a Shopify Liquid 1.0 filter that accepts a size string (`'800x'`, `'600x400'`, `'master'`) and returns a single fixed-resolution image URL. It has no knowledge of the client's display density, viewport width, or supported image formats. Every request — whether from a 4K desktop monitor, a low-end mobile device, or a Googlebot crawler — receives the same single URL at the same resolution. This eliminates Shopify's CDN format negotiation layer: the CDN cannot serve WebP to Chrome or AVIF to Safari 16+ because the URL encodes a fixed format rather than delegating format selection to content negotiation headers.

The practical performance impact is measurable. A product image served via `img_url` at `'1200x'` in JPEG format is typically 180–400 KB. The same image served via `image_url` with WebP format negotiation is 80–160 KB (55–60% reduction). With AVIF for supported browsers, the reduction reaches 70–80%. When multiplied across a collection grid of 24–48 product images, the difference in total page weight is 3–12 MB per page load. Beyond bandwidth: `img_url` returns no aspect ratio metadata, so the browser cannot reserve layout space before the image loads. This causes Cumulative Layout Shift (CLS) on every image load — a Core Web Vitals signal that directly affects Google Search ranking. `image_url` combined with `image_tag` generates `width` and `height` attributes from the image's stored dimensions, eliminating CLS.

`img_url` is classified as `ObsoleteFilter` in Shopify Theme Check and causes Theme Store submission rejection. The filter was officially deprecated when Shopify introduced the Files API and CDN image transformation pipeline in 2021. It will continue to function in existing themes for backward compatibility, but receives no maintenance — new CDN capabilities (AVIF support, lazy loading hints, responsive breakpoint generation) are exposed only through `image_url` and `image_tag`. Themes still using `img_url` are locked out of Shopify's ongoing CDN performance investments.

---

## Detection Logic (AST)

1. **Identify `LiquidFilter` nodes where `name === "img_url"`.** The AST walker traverses all `LiquidFilter` nodes across the template tree. `LiquidFilter.name` contains the filter identifier. Any node where `name` equals `"img_url"` is a violation regardless of context. The filter is obsolete in all usages.

2. **Traverse to the parent `LiquidVariableOutput` node.** From the `LiquidFilter[img_url]` node, the walker ascends to the enclosing `LiquidVariableOutput` (the `{{ }}` expression). This provides the full expression context: what object the filter is applied to (`product.featured_image`, `collection.image`, `article.image`, `settings.logo`), and what other filters follow in the chain.

3. **Inspect the filter chain for `image_tag` presence.** If the `LiquidVariableOutput` filter chain contains `image_tag` after `img_url`, this indicates the developer is generating a full `<img>` tag. The fix must replace the entire chain with `image_url | image_tag` plus `widths:` and `sizes:` arguments. If only `img_url` is present (URL-only usage, likely in a `src` attribute), the fix replaces `img_url` with `image_url`.

4. **Inspect the parent `HtmlElement` or `HtmlVoidElement` for `<img>` context.** If the `LiquidVariableOutput` containing `img_url` is the value of an `HtmlAttribute[name=src]` on an `HtmlVoidElement[tag=img]`, the rule checks whether `width`, `height`, and `loading` attributes are present. Missing dimension attributes are flagged as a CLS vector co-violation alongside the `img_url` deprecation.

**Why regex fails here:** A regex matching `\|\s*img_url` cannot determine whether the filter is applied to a product image (needing `widths:` for responsive srcset) versus a background image URL used in inline CSS (where `image_url` with a fixed size is acceptable). The AST rule resolves the expression root type and the parent HTML context to generate the correct replacement signature for each case.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: img_url used across multiple image contexts.

  Failure mode 1: Fixed JPEG URL — no WebP/AVIF, no format negotiation.
  Bandwidth penalty: ~250 KB per image vs ~80 KB with WebP. For a 48-product
  collection grid, this is ~12 MB of unnecessary JPEG payload per page load.

  Failure mode 2: No width/height attributes on <img> — browser cannot
  reserve layout space before image loads. CLS fires on every image. Google
  ranks pages with CLS > 0.1 lower in Core Web Vitals assessment.

  Failure mode 3: No srcset — every device (mobile 375px, desktop 1440px,
  Retina 2x DPR) loads the same 1200px image. Mobile users download 3x
  the pixels they can display.

  Failure mode 4: img_url inside a for loop without lazy loading —
  all 48 product images begin loading simultaneously, saturating bandwidth.
{%- endcomment -%}

<div class="hero">
  <img
    src="{{ section.settings.hero_image | img_url: '1600x' }}"
    alt="{{ section.settings.hero_image.alt }}"
  >
</div>

<div class="product-grid">
  {% for product in collection.products %}
    <div class="product-card">
      <a href="{{ product.url }}">
        {%- comment -%}
          No loading="lazy", no width/height, fixed JPEG at 800px.
          Every viewport gets identical bytes. No CLS protection.
        {%- endcomment -%}
        <img
          src="{{ product.featured_image | img_url: '800x' }}"
          alt="{{ product.featured_image.alt | escape }}"
        >
      </a>
      <p>{{ product.title }}</p>
    </div>
  {% endfor %}
</div>

<div class="article-thumb">
  <img
    src="{{ article.image | img_url: '400x300' }}"
    alt="{{ article.title }}"
  >
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace img_url with image_url + image_tag throughout.

  Change 1: image_url accepts width: integer — Shopify CDN generates the
  transformation URL. Combined with image_tag's widths: argument, the CDN
  produces a srcset with one entry per listed width automatically.

  Change 2: image_tag generates width and height attributes from the stored
  image dimensions. Browser reserves layout space before the image loads —
  CLS score drops to 0 for these images.

  Change 3: fetchpriority: "high" on the hero image signals to the browser
  that this is the LCP candidate. loading: "lazy" on below-fold images
  defers their fetch until they approach the viewport.

  Change 4: Shopify CDN negotiates format via the Accept request header —
  WebP for Chrome/Edge, AVIF for Safari 16+, JPEG as universal fallback.
  No explicit format: argument needed for standard cases.
{%- endcomment -%}

<div class="hero">
  {%- comment -%}
    Hero image: LCP candidate. fetchpriority: high, no lazy loading.
    sizes: 100vw tells browser this image spans the full viewport width.
  {%- endcomment -%}
  {{
    section.settings.hero_image
    | image_url: width: 1600
    | image_tag:
      widths: '400, 800, 1200, 1600',
      sizes: '100vw',
      fetchpriority: 'high',
      alt: section.settings.hero_image.alt
  }}
</div>

<div class="product-grid">
  {%- comment -%}
    Use render's for: syntax — snippet receives one product at a time.
    image_tag inside the snippet handles all responsive attributes.
  {%- endcomment -%}
  {% render 'product-card' for collection.products as product %}
</div>

{%- comment -%}
  Inside product-card snippet — lazy loading is safe: cards are below fold.
  sizes: reflects the CSS grid layout at each breakpoint.
{%- endcomment -%}
<a href="{{ product.url }}">
  {{
    product.featured_image
    | image_url: width: 800
    | image_tag:
      widths: '200, 400, 600, 800',
      sizes: '(min-width: 990px) 25vw, (min-width: 750px) 33vw, 50vw',
      loading: 'lazy',
      alt: product.featured_image.alt
  }}
</a>

<div class="article-thumb">
  {%- comment -%}
    Article thumbnail: constrained display size, lazy load below fold.
  {%- endcomment -%}
  {{
    article.image
    | image_url: width: 400
    | image_tag:
      widths: '200, 400',
      sizes: '(min-width: 750px) 400px, 100vw',
      loading: 'lazy',
      alt: article.title | escape
  }}
</div>
```

---

## References

- [Shopify — `image_url` filter](https://shopify.dev/docs/api/liquid/filters/image_url)
- [Shopify — `image_tag` filter](https://shopify.dev/docs/api/liquid/filters/image_tag)
- [Shopify Theme Check — ObsoleteFilter rule](https://shopify.dev/docs/storefronts/themes/tools/theme-check/checks/obsolete-filter)
- [web.dev — Cumulative Layout Shift (CLS)](https://web.dev/articles/cls)
- [web.dev — Largest Contentful Paint (LCP)](https://web.dev/articles/lcp)
