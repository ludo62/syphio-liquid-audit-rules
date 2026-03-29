# PERF-005 | Hero image with `loading="lazy"`

**Category:** Performance

**Severity:** 🔴 CRITICAL

**Affects:** First above-the-fold image in any template — homepage hero, landing page hero, collection banner, product media in `section.index == 1`

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`loading="lazy"` instructs the browser to defer an image's network fetch until the image enters — or is near — the viewport boundary. For images below the fold, this is the correct behavior: it eliminates unnecessary fetches for content the user may never scroll to. For the hero image — always in the viewport on initial page load — `loading="lazy"` is a category error with measurable consequences. The browser cannot determine whether an image is in the viewport until it has completed layout calculation. Layout calculation requires the CSSOM to be constructed (CSS fully parsed and applied) and the layout tree to be built. Only after this layout phase does the browser evaluate intersection with the viewport, and only then does it trigger the deferred image fetch. This deferred fetch adds 1–3 seconds to LCP, depending on device class and network conditions.

The mechanism is distinct from simply not using `fetchpriority="high"`. Without `fetchpriority`, the browser still initiates the image fetch during the preload scanning phase — it is merely deprioritized. With `loading="lazy"`, the browser actively suppresses the image fetch until after layout. On a 4G connection with 100ms round-trip time and a render-blocking CSS file of 40KB, the hero image with `loading="lazy"` will not begin downloading until approximately 400–600ms after the HTML response is received. This pushes LCP from a potential 1.5–2.0s range into a consistent 2.5–4.0s range — across the "Needs Improvement" and "Poor" Core Web Vitals thresholds. Google's LCP "Good" threshold is 2.5 seconds at the 75th percentile of real users; a hero image with `loading="lazy"` reliably fails this threshold on mid-range mobile devices, which represent 60–70% of Shopify storefront traffic globally.

This is the highest single-attribute LCP regression achievable in a Shopify theme. The Shopify Theme Store automated review process flags `loading="lazy"` on the first section's primary image as a performance violation. Shopify's Dawn reference theme (all versions from 1.0 through 2024) explicitly applies `loading: 'eager'` (or omits `loading` entirely, relying on the eager default) to the first image in hero sections, and conditionally applies `loading="lazy"` only to images in sections with `section.index > 1`. The pattern typically enters codebases when developers apply a global `loading="lazy"` attribute to all images as a blanket optimization pass, without accounting for the hero exception.

---

## Detection Logic (AST)

1. **Identify candidate hero image elements.** The AST walker locates all `HtmlVoidElement[img]` nodes and `LiquidVariable` nodes that produce `image_tag` output within sections. For each, determine the containing section's render index — either by checking the `section.index` Liquid expression in the surrounding template scope or by identifying the first `{% section %}` or first JSON template section block.

2. **Inspect the `loading` attribute on each `HtmlVoidElement[img]`.** For `HtmlVoidElement` nodes, collect child `HtmlAttribute` nodes and find any where `name === "loading"`. If `value === "lazy"` (case-insensitive), this is a candidate violation. Evaluate the hero context next.

3. **For `image_tag` filter nodes, inspect the named argument list.** `LiquidFilter` nodes with `name === "image_tag"` carry named arguments as a key-value list. If any argument key is `"loading"` and value is `"lazy"`, this is a candidate violation. Trace the filter's input to determine if it is a section-level image (checking whether the expression root is `section.settings.*` or the first image reference in the section scope).

4. **Apply hero context gating.** Only flag the violation if the image is within the first-rendered section scope (static analysis of section render order) or if the element is identified as likely above-the-fold by position in the AST (first `<img>` in the document, or first image in a section block). Images in sections with explicit `section.index > 1` guards in the surrounding Liquid are excluded.

**Why regex fails here:** A regex matching `loading="lazy"` would flag every image in the theme — including correctly lazy-loaded below-fold images, product grid images, and collection thumbnails — producing an overwhelming false-positive rate. The rule is only meaningful when the `loading="lazy"` attribute is on the hero image specifically. Determining "hero" requires tracing the image to its containing section's render order, which requires AST parent-chain traversal through the template hierarchy. No regex can resolve the relationship between an `<img>` attribute and the `section.index` context in which it is rendered.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: loading="lazy" applied to the hero image.
  Context: A developer applied lazy loading to all images as a global
  optimization, unaware that the hero is always in the initial viewport.

  Failure mode 1: Fetch suppressed until after layout — the browser will
  not begin fetching this image until the full layout tree is built and
  intersection with the viewport is evaluated. On a 4G connection, this
  adds 1–3 seconds of suppression delay before the fetch even begins.

  Failure mode 2: LCP is this element — the largest visible element in the
  viewport is the one being most aggressively delayed. The LCP score is
  measured from navigation start to this image's render-complete time.

  Failure mode 3: Double penalty with missing fetchpriority — lazy suppresses
  the fetch entirely until after layout; then the fetch begins without
  high priority. Total delay: 1–3s suppression + 400–800ms deprioritization.

  Failure mode 4: Theme Store rejection — Shopify's automated review detects
  loading="lazy" on first-section images as a performance violation.
{%- endcomment -%}

{%- assign hero_image = section.settings.image -%}

<section class="hero-banner" data-section-id="{{ section.id }}">
  <div class="hero-banner__media-wrapper">
    {%- if hero_image != blank -%}
      <img
        src="{{ hero_image | image_url: width: 1440 }}"
        srcset="
          {{ hero_image | image_url: width: 375 }} 375w,
          {{ hero_image | image_url: width: 750 }} 750w,
          {{ hero_image | image_url: width: 1100 }} 1100w,
          {{ hero_image | image_url: width: 1440 }} 1440w
        "
        sizes="100vw"
        alt="{{ hero_image.alt | escape }}"
        loading="lazy"
        class="hero-banner__image"
      >
    {%- else -%}
      {{ 'lifestyle-1' | placeholder_svg_tag: 'hero-banner__placeholder' }}
    {%- endif -%}
  </div>

  <div class="hero-banner__content-wrapper">
    <div class="hero-banner__content">
      <h2 class="hero-banner__heading h1">{{ section.settings.heading | escape }}</h2>
      {%- if section.settings.subheading != blank -%}
        <p class="hero-banner__subheading">{{ section.settings.subheading | escape }}</p>
      {%- endif -%}
      {%- if section.settings.button_label != blank -%}
        <a href="{{ section.settings.button_link }}" class="hero-banner__btn btn btn--primary">
          {{ section.settings.button_label | escape }}
        </a>
      {%- endif -%}
    </div>
  </div>
</section>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Remove loading="lazy" from the hero image and replace with the
  correct combination of loading="eager" and fetchpriority="high".

  Change 1: loading="eager" is explicit. The browser default is eager, but
  explicit declaration prevents future template edits from accidentally
  inheriting a lazy attribute from a parent snippet or layout include.

  Change 2: fetchpriority="high" is added. This is the complementary
  fix — not only is the fetch no longer suppressed, it is actively promoted
  to the highest position in the browser's fetch queue.

  Change 3: The conditional fetchpriority assignment uses section.index to
  ensure the high-priority hint is applied only on first-section renders.
  In section 2+, eager loading is still correct but fetchpriority should
  not be "high" (it would compete with the actual first-section hero).

  Change 4: width and height are included from the image object to enable
  aspect-ratio reservation, eliminating CLS during the image load.
{%- endcomment -%}

{%- assign hero_image = section.settings.image -%}
{%- assign hero_loading = 'eager' -%}
{%- assign hero_fetchpriority = 'auto' -%}
{%- if section.index == 1 -%}
  {%- assign hero_fetchpriority = 'high' -%}
{%- endif -%}

<section class="hero-banner" data-section-id="{{ section.id }}">
  <div class="hero-banner__media-wrapper">
    {%- if hero_image != blank -%}
      <img
        src="{{ hero_image | image_url: width: 1440 }}"
        srcset="
          {{ hero_image | image_url: width: 375 }} 375w,
          {{ hero_image | image_url: width: 750 }} 750w,
          {{ hero_image | image_url: width: 1100 }} 1100w,
          {{ hero_image | image_url: width: 1440 }} 1440w
        "
        sizes="100vw"
        width="{{ hero_image.width }}"
        height="{{ hero_image.height }}"
        alt="{{ hero_image.alt | escape }}"
        loading="{{ hero_loading }}"
        fetchpriority="{{ hero_fetchpriority }}"
        class="hero-banner__image"
      >
    {%- else -%}
      {{ 'lifestyle-1' | placeholder_svg_tag: 'hero-banner__placeholder' }}
    {%- endif -%}
  </div>

  <div class="hero-banner__content-wrapper">
    <div class="hero-banner__content">
      <h2 class="hero-banner__heading h1">{{ section.settings.heading | escape }}</h2>
      {%- if section.settings.subheading != blank -%}
        <p class="hero-banner__subheading">{{ section.settings.subheading | escape }}</p>
      {%- endif -%}
      {%- if section.settings.button_label != blank -%}
        <a href="{{ section.settings.button_link }}" class="hero-banner__btn btn btn--primary">
          {{ section.settings.button_label | escape }}
        </a>
      {%- endif -%}
    </div>
  </div>
</section>
```

---

## References

- [Shopify — `image_tag` filter: loading and fetchpriority arguments](https://shopify.dev/docs/api/liquid/filters/image_tag)
- [Shopify — Theme performance: LCP](https://shopify.dev/docs/storefronts/themes/best-practices/performance#optimize-lcp)
- [web.dev — Browser-level lazy loading for the web](https://web.dev/articles/browser-level-lazy-loading-for-cmss)
- [web.dev — Avoid lazy loading above-the-fold images](https://web.dev/articles/lazy-loading-images#avoid_lazy_loading_images_that_are_in_the_first_visible_viewport)
- [Google — Core Web Vitals: LCP thresholds](https://web.dev/articles/lcp#what_is_a_good_lcp_score)
- [MDN — HTMLImageElement: loading](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/loading)
