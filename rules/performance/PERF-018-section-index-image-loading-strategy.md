# PERF-018 | `section.index` not used to differentiate above/below-fold images

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Section files that render images via `image_tag` or `<img>` elements, particularly hero, slideshow, and collection grid sections

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`section.index` is a Shopify Liquid integer property that returns the 1-based position of a section in the rendered section stack for the current template. The first section (`section.index == 1`) renders at the top of the page and its content is always above the fold on initial paint — its images are Largest Contentful Paint candidates. Sections with `section.index > 1` render below the first section and are almost always outside the initial viewport; their images should not be fetched until the user scrolls them into view. The correct image loading strategy differs fundamentally between these two groups: above-fold images should use `fetchpriority="high"` and no `loading="lazy"` attribute; below-fold images should use `loading="lazy"` and no `fetchpriority` attribute (or `fetchpriority="low"`).

When themes apply a uniform image loading strategy — either `loading="lazy"` for all images or `loading="eager"` for all images — both extremes produce measurable regressions. A hero image with `loading="lazy"` is deprioritized by the browser's resource scheduler. In Chrome's implementation, lazy-loaded images are loaded with a network priority of "Low" and are not fetched until the image enters the viewport proximity threshold (typically 1250px from the viewport edge on slow connections). A hero image that should be the LCP element is instead loaded last — in documented cases, this causes LCP regressions of 500ms–2s on mobile. Conversely, applying `loading="eager"` to all images including those in the 4th, 5th, and 6th sections causes the browser to fetch images that may never be viewed in a single session, wasting bandwidth and increasing Time to Interactive by occupying network bandwidth that could serve the LCP image.

The `section.index` branching pattern — `{% if section.index == 1 %}fetchpriority="high"{% else %}loading="lazy"{% endif %}` — is the standard Shopify Dawn-derived optimization. It requires zero JavaScript, adds no network overhead, and is evaluated at Liquid render time on Shopify's edge. The fix is a single conditional per image render call. Themes that do not implement this pattern are eligible for Theme Store rejection under the performance audit checklist, which explicitly checks that hero-section images are not lazy-loaded. For themes using the `image_tag` filter, `fetchpriority` and `loading` values can be passed as named arguments directly to the filter.

---

## Detection Logic (AST)

1. **Identify `LiquidFilter` nodes named `image_tag` and `HtmlVoidElement[img]` nodes.** The AST walker collects two node types: `LiquidFilter` nodes with `name === "image_tag"` (Shopify's recommended image rendering filter), and `HtmlVoidElement` nodes with `tagName === "img"`. Both are image rendering sites subject to this rule.

2. **Inspect `loading` argument/attribute presence and value.** For `image_tag` filter nodes, inspect the `args` array for a `NamedArgument` with `name === "loading"`. For `HtmlVoidElement[img]` nodes, inspect the `HtmlAttribute` list for `name === "loading"`. If the value is unconditionally `"lazy"` (a string literal without a conditional expression), flag as a violation — the image loading strategy is uniform regardless of section position.

3. **Inspect `fetchpriority` argument/attribute presence.** Perform the same inspection for `fetchpriority` (on `image_tag`) or `fetchpriority` attribute (on `<img>`). An unconditional `fetchpriority="high"` on all images (including below-fold sections) wastes browser scheduler priority slots. An absent `fetchpriority` on section index 1 images is a missed LCP optimization.

4. **Check whether the loading value is conditioned on `section.index`.** Traverse the AST ancestor chain from the image node to locate a `LiquidTag[if]` or `LiquidTag[unless]` ancestor that references `section.index` in its condition. If the nearest conditional ancestor references `section.index == 1` (or `section.index > 1`), the loading strategy is section-index-aware and is compliant. If no such conditional ancestor exists, the strategy is unconditional and is a violation.

**Why regex fails here:** A regex matching `loading="lazy"` cannot determine whether the attribute appears inside a conditional block that references `section.index`. The conditional block may span many lines above the `<img>` tag; line-proximity heuristics would produce both false positives (a lazy image correctly inside an `{% elsif section.index > 1 %}` branch) and false negatives (a lazy image inside an `{% if settings.some_toggle %}` block that happens to contain `section.index` elsewhere). The AST walker resolves the precise ancestor chain from the image node to the enclosing conditional tag and can inspect the condition's expression for `section.index` membership with exact scope.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Uniform loading="lazy" applied to all images regardless of
  section position. This breaks LCP for the hero section (section.index == 1).

  Failure mode 1: The hero image (section.index == 1) receives loading="lazy".
  Chrome deprioritizes it to "Low" network priority. The LCP element is the
  last image fetched, causing LCP regressions of 500ms–2s on mobile 4G.

  Failure mode 2: fetchpriority="high" is not set for above-fold images.
  The browser's preload scanner does not elevate the hero image above other
  resources — CSS, scripts, and below-fold images compete equally.

  Failure mode 3: The section schema allows this section to be placed at
  any position (section.index 1–N). Using it as the first section — the
  hero position — with lazy loading is a guaranteed LCP failure.

  Failure mode 4: loading="lazy" on a section.index == 1 image is explicitly
  flagged in Shopify's automated Theme Store performance audit.
{%- endcomment -%}

<div class="image-banner" data-section-id="{{ section.id }}">
  {%- if section.settings.image != blank -%}
    <div class="image-banner__media">
      {{
        section.settings.image
        | image_url: width: 1500
        | image_tag:
          loading: 'lazy',
          widths: '375,750,1100,1500',
          sizes: '100vw',
          class: 'image-banner__image',
          alt: section.settings.image.alt | escape
      }}
    </div>
  {%- else -%}
    <div class="image-banner__placeholder">
      {{ 'lifestyle-1' | placeholder_svg_tag: 'placeholder-svg' }}
    </div>
  {%- endif -%}

  <div class="image-banner__content">
    <h2 class="image-banner__heading">{{ section.settings.heading }}</h2>
    <p class="image-banner__subheading">{{ section.settings.subheading }}</p>
  </div>
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Branch image loading strategy on section.index to apply the correct
  priority for above-fold (index == 1) and below-fold (index > 1) images.

  Change 1: section.index == 1 branch uses fetchpriority="high" and omits
  loading="lazy". The browser's preload scanner picks up the image with
  "High" network priority. This is the primary LCP optimization for hero
  sections — no JavaScript required, evaluated at render time on Shopify's edge.

  Change 2: section.index > 1 branch uses loading="lazy" without fetchpriority.
  Images outside the initial viewport are deferred until proximity to the
  viewport, saving initial page bandwidth for LCP-critical resources.

  Change 3: The Liquid assign pattern caches the loading/fetchpriority values
  before the image_tag call — clean, readable, and ensures the conditional
  logic executes once rather than being inlined into the filter argument list.

  Change 4: Both branches supply consistent widths and sizes attributes —
  responsive image behavior is identical; only the loading priority differs.
{%- endcomment -%}

{%- assign lazy_load = section.index > 1 -%}

<div class="image-banner" data-section-id="{{ section.id }}">
  {%- if section.settings.image != blank -%}
    <div class="image-banner__media">
      {%- if lazy_load -%}
        {{
          section.settings.image
          | image_url: width: 1500
          | image_tag:
            loading: 'lazy',
            widths: '375,750,1100,1500',
            sizes: '100vw',
            class: 'image-banner__image',
            alt: section.settings.image.alt | escape
        }}
      {%- else -%}
        {{
          section.settings.image
          | image_url: width: 1500
          | image_tag:
            fetchpriority: 'high',
            widths: '375,750,1100,1500',
            sizes: '100vw',
            class: 'image-banner__image',
            alt: section.settings.image.alt | escape
        }}
      {%- endif -%}
    </div>
  {%- else -%}
    <div class="image-banner__placeholder">
      {{ 'lifestyle-1' | placeholder_svg_tag: 'placeholder-svg' }}
    </div>
  {%- endif -%}

  <div class="image-banner__content">
    <h2 class="image-banner__heading">{{ section.settings.heading }}</h2>
    <p class="image-banner__subheading">{{ section.settings.subheading }}</p>
  </div>
</div>
```

---

## References

- [Shopify — `section.index` object property](https://shopify.dev/docs/api/liquid/objects/section#section-index)
- [Shopify — `image_tag` filter](https://shopify.dev/docs/api/liquid/filters/image_tag)
- [web.dev — Optimize LCP: Prioritize the LCP image](https://web.dev/articles/optimize-lcp#prioritize_the_lcp_image)
- [web.dev — `fetchpriority` attribute](https://web.dev/articles/fetch-priority)
- [MDN — `<img loading="lazy">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#loading)
- [Shopify Dawn theme — image loading pattern](https://github.com/Shopify/dawn)
