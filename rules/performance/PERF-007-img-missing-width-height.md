# PERF-007 | `<img>` without explicit `width` and `height` attributes

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** All `<img>` elements across all templates — product grids, collection pages, article pages, cart line items, search results

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Cumulative Layout Shift (CLS) measures unexpected visual instability during page load — specifically, the sum of all layout shift scores for unexpected shifts occurring during the page's lifespan. Each layout shift score is the product of the shift's impact fraction (the viewport area affected) and the distance fraction (how far elements moved). When an `<img>` element lacks explicit `width` and `height` attributes, the browser reserves zero vertical space for it during the layout phase before the image file is fetched. When the image dimensions become known — after the image response headers arrive — the browser recalculates layout, inserting the image's actual height into the flow. Everything below the image shifts downward by that height. For a 400px tall product image in a grid, this is a downward shift affecting potentially 60–80% of the viewport area. The resulting layout shift score for a single image can be 0.2–0.4; twelve images in a product grid can accumulate a CLS score of 0.5–1.0 on slow connections where images load sequentially.

The browser's aspect-ratio reservation mechanism, introduced in Chrome 79 and now universally supported, solves this problem without requiring the developer to hardcode fixed dimensions. When both `width` and `height` attributes are present, modern browsers compute the image's intrinsic aspect ratio (`height / width`) and apply it via CSS `aspect-ratio` to reserve the correct vertical space before the image loads. The image dimensions in the HTML attributes do not need to match the rendered CSS size — they simply establish the ratio. An image with `width="1200" height="800"` will reserve a 3:2 aspect-ratio box at whatever CSS width the image is rendered at, preventing layout shift entirely. Shopify's `image_url` objects expose `.width` and `.height` as integer properties of the `ImageFile` object — these should always be passed to `image_tag` or to the raw `<img>` element.

Google's Core Web Vitals "Good" threshold for CLS is 0.1 at the 75th percentile of real users. A product collection page with 24 items, each missing `width` and `height`, will exceed 0.1 CLS on any connection slower than a fast 4G network, where images load with visible latency. The CLS metric is measured for the full lifespan of the page, including shifts caused by image loads that occur during scroll (for lazy-loaded below-fold images). Shopify's Theme Store review process includes an automated CLS check; themes that score CLS > 0.1 on the review harness receive a performance warning. Shopify's `image_tag` filter automatically includes `width` and `height` attributes when passed `width:` and `height:` arguments — themes using raw `<img src="{{ ... }}">` syntax have no automatic protection.

---

## Detection Logic (AST)

1. **Identify all `HtmlVoidElement[img]` nodes in the template AST.** The walker traverses the full template tree collecting every `HtmlVoidElement` node where `name === "img"`. These represent all `<img>` tags — both static and Liquid-interpolated source images.

2. **Collect `HtmlAttribute` child nodes for each `<img>`.** For each `HtmlVoidElement[img]`, enumerate all `HtmlAttribute` children. Build a set of present attribute names. Check whether both `"width"` and `"height"` are members of this set.

3. **Distinguish zero-value attributes from missing attributes.** An `HtmlAttribute` with `name === "width"` and `value === "0"` is present but meaningless; flag it the same as missing. An `HtmlAttribute` with a `LiquidVariable` value (e.g., `width="{{ image.width }}"`) is valid — the attribute is present and dynamically set. Only flag cases where the attribute node is entirely absent from the `HtmlAttribute` child list.

4. **Check `image_tag` filter nodes for `width` and `height` arguments.** For `LiquidVariable` expressions that chain `image_tag`, inspect the filter's named argument list. If both `width:` and `height:` arguments are present with non-nil values (including Liquid variable references to `image.width` / `image.height`), no violation is raised. If either is absent, flag as a violation.

**Why regex fails here:** A regex matching `<img` without `width` must perform negative lookahead across the entire opening tag content, which may span multiple lines and contain Liquid interpolation. Multi-line Liquid-interpolated attribute values (e.g., `widths: '400,800'` on a separate indented line) make multi-line attribute parsing by regex unreliable. The AST represents each attribute as a discrete typed node — presence checking is a set membership operation with no string parsing required. Additionally, the regex cannot distinguish the `image_tag` filter output (where `width`/`height` may be in the filter arguments) from a raw `<img>` tag (where they must be HTML attributes).

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: <img> elements without explicit width and height attributes.
  Context: A collection product grid rendering 24 product cards.

  Failure mode 1: Zero vertical space reserved per image — each of the 24
  images causes a layout shift when it loads, shifting all content below it.
  CLS score can reach 0.5–1.0 on slow connections. "Good" threshold is 0.1.

  Failure mode 2: The aspect ratio of each product image is unknown until
  fetch completes. If images have mixed aspect ratios (portraits, landscapes,
  squares), the layout shifts are unpredictably sized and directioned.

  Failure mode 3: Even loading="lazy" does not prevent CLS — the shift occurs
  when the lazy image enters the viewport and loads, causing a shift during
  scroll that also counts toward the CLS score.

  Failure mode 4: image_tag filter invoked without width/height arguments,
  so the generated <img> also lacks these attributes. This is the most
  common form — the filter is used but incompletely configured.
{%- endcomment -%}

<div class="collection-grid" data-section-id="{{ section.id }}">
  {%- for product in collection.products -%}
    <div class="collection-grid__item">
      <a href="{{ product.url }}" class="collection-grid__link">
        {%- if product.featured_image != blank -%}
          <div class="collection-grid__media">
            {{
              product.featured_image
              | image_url: width: 500
              | image_tag:
                class: 'collection-grid__image',
                loading: 'lazy',
                alt: product.featured_image.alt
            }}
          </div>
        {%- else -%}
          <div class="collection-grid__media collection-grid__media--placeholder">
            {{ 'product-1' | placeholder_svg_tag }}
          </div>
        {%- endif -%}

        <div class="collection-grid__info">
          {%- if section.settings.show_vendor -%}
            <p class="collection-grid__vendor">{{ product.vendor }}</p>
          {%- endif -%}
          <h3 class="collection-grid__title">{{ product.title }}</h3>
          <p class="collection-grid__price">{{ product.price | money }}</p>
        </div>
      </a>
    </div>
  {%- endfor -%}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Pass width and height from the image object to image_tag.
  The browser uses these to compute the aspect ratio and reserves the
  correct vertical space before the image loads — eliminating layout shift.

  Change 1: width: product.featured_image.width and
  height: product.featured_image.height are passed as named arguments
  to image_tag. These become the <img width="..." height="..."> attributes
  in the rendered HTML output.

  Change 2: The values do not need to match the CSS rendered size. They
  establish the intrinsic aspect ratio, which browsers use via the implicit
  aspect-ratio CSS property. A 1200×800 image rendered at 500px CSS width
  will reserve a 500×333px box — the correct 3:2 ratio.

  Change 3: widths parameter added to generate srcset (PERF-006 compliance).
  sizes matches the CSS grid layout — 4 columns on desktop, 2 on mobile.

  Change 4: The vendor conditional is hoisted outside the loop using an
  {% assign %} above the {% for %} (PERF-011 compliance).
{%- endcomment -%}

{%- assign show_vendor = section.settings.show_vendor -%}

<div class="collection-grid" data-section-id="{{ section.id }}">
  {%- for product in collection.products -%}
    <div class="collection-grid__item">
      <a href="{{ product.url }}" class="collection-grid__link">
        {%- if product.featured_image != blank -%}
          <div class="collection-grid__media">
            {{-
              product.featured_image
              | image_url: width: 500
              | image_tag:
                class: 'collection-grid__image',
                loading: 'lazy',
                widths: '250,500,750',
                sizes: '(min-width: 1200px) 25vw, (min-width: 750px) 33vw, 50vw',
                width: product.featured_image.width,
                height: product.featured_image.height,
                alt: product.featured_image.alt
            -}}
          </div>
        {%- else -%}
          <div class="collection-grid__media collection-grid__media--placeholder">
            {{ 'product-1' | placeholder_svg_tag }}
          </div>
        {%- endif -%}

        <div class="collection-grid__info">
          {%- if show_vendor -%}
            <p class="collection-grid__vendor">{{ product.vendor }}</p>
          {%- endif -%}
          <h3 class="collection-grid__title">{{ product.title }}</h3>
          <p class="collection-grid__price">{{ product.price | money }}</p>
        </div>
      </a>
    </div>
  {%- endfor -%}
</div>
```

---

## References

- [Shopify — `image_tag` filter: width and height arguments](https://shopify.dev/docs/api/liquid/filters/image_tag)
- [Shopify — Image object: width and height properties](https://shopify.dev/docs/api/liquid/objects/image#image-width)
- [web.dev — Optimize Cumulative Layout Shift](https://web.dev/articles/optimize-cls)
- [web.dev — Setting width and height on images](https://web.dev/articles/optimize-cls#images_without_dimensions)
- [Google — Core Web Vitals: CLS](https://web.dev/articles/cls)
- [MDN — aspect-ratio and width/height attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#width)
