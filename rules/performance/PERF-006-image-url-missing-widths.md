# PERF-006 | `image_url` without `widths` parameter

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** All templates using `image_url` filter without a `widths:` argument — product pages, collection grids, article images, section backgrounds

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Shopify's CDN can serve any image at arbitrary widths by appending query parameters to the CDN URL. The `image_url` filter with a `widths:` parameter generates a comma-separated srcset string containing multiple CDN URLs, each targeting a specific pixel width. The browser uses this srcset, in combination with the `sizes` attribute, to select the smallest image that will fill the display area at the current viewport width and device pixel ratio. Without `widths:`, a single URL is generated at the requested width (defaulting to the image's original upload size if no width is specified). A desktop hero image uploaded at 2000×1000px and served without a srcset will be downloaded at full 2000px width on a 375px mobile screen — delivering 2000px of horizontal resolution where 750px (375 × 2× DPR) is the maximum useful width. The excess bandwidth is 40–60% waste on typical mobile devices.

Across a product page with 10 images and a collection grid with 12 product thumbnails — 22 images total — the absence of responsive srcset on a store with 70% mobile traffic means 22 full-resolution images are served to mobile users on every page view. A 1200px product image at 150KB becomes a 40KB image at 400px wide, without quality loss perceptible at that display size. At 1,000 daily visitors with 70% mobile share, the cumulative bandwidth waste is approximately (110KB × 22 images × 700 visitors) = ~1.7GB per day of excess CDN data transfer for a single store. Shopify's CDN does not charge merchants by bandwidth, but the user-facing consequence is real: increased page weight degrades LCP and Time to Interactive on mobile networks, pushing Core Web Vitals scores into the "Needs Improvement" range.

The fix is to combine `image_url` with `image_tag` and provide a `widths:` argument. Shopify's `image_tag` filter, when given `widths:`, automatically constructs the full `srcset` attribute with all specified breakpoint URLs. The `sizes` attribute should be set to match the CSS layout breakpoints where the image is displayed. For product thumbnails in a 4-column desktop / 2-column mobile grid, the correct `sizes` value is `(min-width: 750px) 25vw, 50vw`. Shopify's Dawn and Craft themes use this pattern consistently across all image outputs. The `image_url` filter alone — chained into an `<img src>` without a corresponding srcset — is the legacy pattern from pre-OS2.0 themes that had no image transformation infrastructure.

---

## Detection Logic (AST)

1. **Identify all `LiquidFilter` nodes with `name === "image_url"`.** The AST walker collects every `LiquidFilter` node where the filter name is `image_url`. These nodes appear in `LiquidVariable` expression chains (e.g., `{{ image | image_url: width: 800 }}`) and in `LiquidTag[assign]` right-hand sides.

2. **Inspect the filter's argument list for a `widths` named argument.** Each `LiquidFilter` node carries an `args` or `namedArguments` list. Check whether any argument key is `"widths"` with a non-empty value (a comma-separated list of integer widths or a Liquid variable). If the `widths` argument is absent, this is a candidate violation.

3. **Check whether the `image_url` output is consumed by `image_tag`.** In the filter chain, if the next filter (the node immediately following `image_url` in the chain) is `image_tag`, inspect the `image_tag` argument list for `widths:`. An `image_tag` with a `widths:` argument generates the srcset correctly even if `image_url` itself did not receive `widths:`. In this case, no violation is raised — the srcset is handled at the `image_tag` layer.

4. **Flag `image_url` outputs used as bare `src` attribute values.** If the `LiquidVariable` containing the `image_url` filter chain is the value of an `HtmlAttribute` node with `name === "src"` on an `HtmlVoidElement[img]`, and the same `<img>` node has no `srcset` attribute, flag as a violation. Single-URL `src`-only image tags on any non-`<picture>` element represent the full failure mode.

**Why regex fails here:** A regex matching `image_url` without `widths` would need to parse the full filter argument string to confirm the absence of `widths:` — distinguishing `image_url: width: 800` (no srcset) from `image_url: width: 800, widths: '400,800,1200'` (correct). It would also need to determine whether the `image_url` result flows into an `image_tag` filter (where `widths` may be declared at a later stage in the chain). Chained filter expressions can span multiple lines and assignment intermediaries, making argument-presence detection by regex unreliable. The AST walker traverses the filter chain as a structured list of nodes with typed argument maps.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: image_url used without widths parameter.
  Context: A product page rendering the main product image gallery
  and a related products grid — all images served at a single large size.

  Failure mode 1: Single URL, no srcset — every viewport (375px mobile
  to 2560px desktop) receives the identical 1200px image. Mobile users
  download 40–60% more image data than needed.

  Failure mode 2: Featured image and gallery images both affected —
  10–15 images per product page, each serving full resolution to mobile.
  On a 100Mbps mobile LTE connection, this is ~1.5s of image transfer time
  that could be eliminated with responsive srcset.

  Failure mode 3: Related product thumbnails displayed at ~200px wide
  are loaded at 600px — 3x the needed resolution.
{%- endcomment -%}

<div class="product-media" data-section-id="{{ section.id }}">
  <div class="product-media__main">
    {%- assign featured = product.selected_or_first_available_variant.featured_image
      | default: product.featured_image
    -%}
    {%- if featured != blank -%}
      <img
        src="{{ featured | image_url: width: 1200 }}"
        alt="{{ featured.alt | escape }}"
        class="product-media__featured"
        id="main-product-image"
      >
    {%- endif -%}
  </div>

  <div class="product-media__thumbnails">
    {%- for image in product.images limit: 5 -%}
      <button class="product-media__thumb-btn" data-image-id="{{ image.id }}">
        <img
          src="{{ image | image_url: width: 200 }}"
          alt="{{ image.alt | escape }}"
          class="product-media__thumb"
        >
      </button>
    {%- endfor -%}
  </div>
</div>

<div class="related-products">
  {%- for related_product in collection.products limit: 4 -%}
    {%- unless related_product.id == product.id -%}
      <div class="related-products__item">
        <img
          src="{{ related_product.featured_image | image_url: width: 600 }}"
          alt="{{ related_product.featured_image.alt | escape }}"
          class="related-products__image"
          loading="lazy"
        >
        <p>{{ related_product.title }}</p>
      </div>
    {%- endunless -%}
  {%- endfor -%}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add widths parameter to all image_url usages and use image_tag
  to generate proper srcset and sizes attributes automatically.

  Change 1: image_tag with widths generates a full srcset string for all
  specified breakpoints. The browser fetches only the width matching the
  current viewport × DPR — reducing mobile bandwidth by 40–60%.

  Change 2: sizes attribute is tailored to the CSS layout. The main product
  image fills 100vw on mobile and ~50vw on desktop (two-column layout).
  Thumbnails are fixed at ~80px. Related products are ~25vw on desktop.

  Change 3: The hero/featured image retains loading="eager" and
  fetchpriority="high" (PERF-004/005 compliance). Gallery thumbnails and
  related products use loading="lazy" — they are below the fold.

  Change 4: width and height from the image object enable aspect-ratio
  reservation, preventing CLS while images load.
{%- endcomment -%}

<div class="product-media" data-section-id="{{ section.id }}">
  <div class="product-media__main">
    {%- assign featured = product.selected_or_first_available_variant.featured_image
      | default: product.featured_image
    -%}
    {%- if featured != blank -%}
      {{-
        featured
        | image_url: width: 1200
        | image_tag:
          class: 'product-media__featured',
          id: 'main-product-image',
          loading: 'eager',
          fetchpriority: 'high',
          widths: '400,600,900,1200',
          sizes: '(min-width: 1200px) 50vw, (min-width: 750px) 60vw, 100vw',
          width: featured.width,
          height: featured.height,
          alt: featured.alt
      -}}
    {%- endif -%}
  </div>

  <div class="product-media__thumbnails">
    {%- for image in product.images limit: 5 -%}
      <button class="product-media__thumb-btn" data-image-id="{{ image.id }}">
        {{-
          image
          | image_url: width: 160
          | image_tag:
            class: 'product-media__thumb',
            loading: 'lazy',
            widths: '80,160',
            sizes: '80px',
            width: image.width,
            height: image.height,
            alt: image.alt
        -}}
      </button>
    {%- endfor -%}
  </div>
</div>

<div class="related-products">
  {%- for related_product in collection.products limit: 4 -%}
    {%- unless related_product.id == product.id -%}
      <div class="related-products__item">
        {{-
          related_product.featured_image
          | image_url: width: 400
          | image_tag:
            class: 'related-products__image',
            loading: 'lazy',
            widths: '200,400',
            sizes: '(min-width: 750px) 25vw, 50vw',
            width: related_product.featured_image.width,
            height: related_product.featured_image.height,
            alt: related_product.featured_image.alt
        -}}
        <p>{{ related_product.title }}</p>
      </div>
    {%- endunless -%}
  {%- endfor -%}
</div>
```

---

## References

- [Shopify — `image_url` filter](https://shopify.dev/docs/api/liquid/filters/image_url)
- [Shopify — `image_tag` filter: widths and sizes](https://shopify.dev/docs/api/liquid/filters/image_tag)
- [Shopify — Responsive images guide](https://shopify.dev/docs/storefronts/themes/best-practices/images)
- [MDN — Responsive images: srcset and sizes](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images)
- [web.dev — Use images with correct dimensions](https://web.dev/articles/uses-responsive-images)
- [Shopify Dawn — Responsive image implementation](https://github.com/Shopify/dawn/blob/main/snippets/image.liquid)
