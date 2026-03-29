# PERF-004 | Hero image without `fetchpriority="high"`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Homepage hero sections, landing page hero sections, any `<img>` or `image_tag` output that is the first visible image in the viewport for `section.index == 1`

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Largest Contentful Paint (LCP) measures the render time of the largest visible element in the viewport at initial page load. For the overwhelming majority of Shopify storefronts, this element is the hero image in the first above-the-fold section. The browser's resource priority queue governs fetch ordering: CSS, preloaded fonts, and scripts discovered in the `<head>` are assigned "Highest" or "High" priority by default. Images are assigned "Low" or "Medium" priority unless the browser's preload scanner heuristics promote them — typically only for the very first image in the document and only on some viewport sizes. Without an explicit `fetchpriority="high"` attribute, the hero image competes against lower-priority assets and may not enter the fetch queue until the preload scanner has traversed past it, adding 400–800ms to LCP on median connection speeds.

Shopify's CDN serves assets from edge nodes geographically distributed relative to the visitor. Even from a nearby edge node, the round-trip time for an image fetch at "Low" priority — delayed behind synchronous scripts and CSS — is measurably longer than the same fetch at "High" priority, because the browser's fetch scheduler queues low-priority fetches after all in-flight high-priority requests resolve. On a storefront with 3 render-blocking scripts and 2 CSS files, the hero image without `fetchpriority="high"` will not begin fetching until all 5 of those assets are at least partially received. Shopify's own performance documentation explicitly recommends `fetchpriority="high"` on the first image in `section.index == 1`, and the Dawn 2024 reference theme applies this attribute conditionally based on section index.

Google's Core Web Vitals "Good" threshold for LCP is 2.5 seconds. PageSpeed Insights uses field data from the Chrome User Experience Report (CrUX) to score LCP at the 75th percentile of real users. A single 400–800ms delay in hero image priority, compounded across mobile users on 4G connections (the largest traffic segment for most Shopify stores), routinely pushes the 75th-percentile LCP from the "Good" range into "Needs Improvement" (2.5–4.0s). Google uses Core Web Vitals as a search ranking signal via the Page Experience update; LCP failure in the "Needs Improvement" range can reduce organic search ranking for the homepage — directly impacting unpaid acquisition for merchants.

---

## Detection Logic (AST)

1. **Identify sections where `section.index == 1` or `section.index` equals the first rendered section.** In the template AST, locate `LiquidVariable` nodes that output `section.index`. The PERF-004 rule applies specifically to sections that render first in the page order — either statically (the first `{% section %}` or `{% sections %}` call in `theme.liquid`) or dynamically (a JSON template's first section entry). Flag the first section block in the template as the hero candidate scope.

2. **Within the hero section scope, find all `HtmlElement[img]` and `HtmlVoidElement[img]` nodes.** Traverse the section's AST for `HtmlVoidElement` nodes where `name === "img"`. Also identify `LiquidTag[assign]` or `LiquidVariable` nodes that produce `image_tag` filter output — these generate `<img>` elements at render time and must be checked via their filter argument list.

3. **Inspect the `HtmlAttribute` list of each `<img>` node for `fetchpriority`.** For `HtmlVoidElement[img]`, collect all `HtmlAttribute` child nodes and check whether any has `name === "fetchpriority"` and `value === "high"` (case-insensitive). For `image_tag` filter nodes, inspect the filter's named argument list for `fetchpriority: 'high'`.

4. **Classify by position within section.** The first `<img>` in the section AST (lowest source offset) is the primary candidate. If it lacks `fetchpriority="high"`, report as HIGH. If `loading="lazy"` is also present on the same element, escalate to CRITICAL (see PERF-005) and emit both violations.

**Why regex fails here:** A regex matching `<img` followed by inspection of attributes fails to associate the image with its section context — it cannot determine whether the image is in `section.index == 1` or in a below-fold section where `fetchpriority="high"` would be incorrect. Adding `fetchpriority="high"` to all images (to satisfy a naive regex-based rule) is harmful: it causes resource contention by elevating non-critical images to the highest fetch priority, starving the actual LCP element. The AST walker maintains the parent section scope during traversal, enabling precise targeting of only the hero section's first image.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: Hero image rendered without fetchpriority="high".
  Context: The first section on the homepage — the primary LCP element.

  Failure mode 1: Browser assigns "Low" priority to this image fetch.
  With 3 render-blocking scripts and 2 CSS files in the <head>, the browser
  does not begin fetching the hero until those assets resolve — adding
  400–800ms to LCP on median 4G connections.

  Failure mode 2: loading="lazy" is absent (good), but the missing
  fetchpriority means the preload scanner does not treat this image
  as a priority candidate even after it is discovered.

  Failure mode 3: No explicit width/height — aspect ratio unknown to browser
  before load, causing CLS in addition to LCP regression (see PERF-007).

  Failure mode 4: image_url without widths parameter means the full-size
  image is served to all viewports (see PERF-006), compounding the LCP
  delay with unnecessary bandwidth on mobile.
{%- endcomment -%}

{%- assign hero_image = section.settings.image -%}

<section
  class="hero"
  data-section-id="{{ section.id }}"
  data-section-type="hero"
>
  <div class="hero__media">
    {%- if hero_image != blank -%}
      <img
        src="{{ hero_image | image_url: width: 1440 }}"
        alt="{{ hero_image.alt | escape }}"
        class="hero__image"
      >
    {%- else -%}
      {{ 'image' | placeholder_svg_tag: 'hero__placeholder' }}
    {%- endif -%}
  </div>

  <div class="hero__content">
    <h1 class="hero__heading">{{ section.settings.heading }}</h1>
    <p class="hero__subheading">{{ section.settings.subheading }}</p>
    <a href="{{ section.settings.button_url }}" class="hero__cta btn">
      {{ section.settings.button_label }}
    </a>
  </div>
</section>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Add fetchpriority="high" to the hero image and apply additional
  LCP optimizations consistent with Shopify Dawn's hero implementation.

  Change 1: fetchpriority="high" instructs the browser's fetch scheduler
  to treat this image as the highest-priority network request, placing it
  ahead of non-critical scripts and late-discovered images.

  Change 2: loading="eager" is explicit — this is the browser default, but
  explicit declaration prevents any downstream template modification from
  accidentally adding loading="lazy" to this element.

  Change 3: widths parameter generates a srcset so the browser fetches the
  smallest image that covers the display resolution — reducing payload by
  40–60% on mobile viewports without sacrificing visual quality.

  Change 4: Explicit width and height prevent CLS by allowing the browser to
  reserve aspect-ratio space before the image loads.

  Change 5: fetchpriority is only applied when section.index == 1 — below-fold
  sections receive no priority boost, avoiding fetch contention.
{%- endcomment -%}

{%- assign hero_image = section.settings.image -%}
{%- assign is_first_section = false -%}
{%- if section.index == 1 -%}
  {%- assign is_first_section = true -%}
{%- endif -%}

<section
  class="hero"
  data-section-id="{{ section.id }}"
  data-section-type="hero"
>
  <div class="hero__media">
    {%- if hero_image != blank -%}
      {{-
        hero_image
        | image_url: width: 1500
        | image_tag:
          class: 'hero__image',
          loading: 'eager',
          fetchpriority: is_first_section | if: 'high' | default: 'auto',
          widths: '375,750,1100,1500',
          width: hero_image.width,
          height: hero_image.height,
          alt: hero_image.alt
      -}}
    {%- else -%}
      {{ 'image' | placeholder_svg_tag: 'hero__placeholder' }}
    {%- endif -%}
  </div>

  <div class="hero__content">
    <h1 class="hero__heading">{{ section.settings.heading }}</h1>
    <p class="hero__subheading">{{ section.settings.subheading }}</p>
    {%- if section.settings.button_label != blank -%}
      <a href="{{ section.settings.button_url }}" class="hero__cta btn">
        {{ section.settings.button_label }}
      </a>
    {%- endif -%}
  </div>
</section>
```

---

## References

- [Shopify — `image_tag` filter with `fetchpriority`](https://shopify.dev/docs/api/liquid/filters/image_tag)
- [Shopify — Theme performance: LCP optimization](https://shopify.dev/docs/storefronts/themes/best-practices/performance#optimize-lcp)
- [web.dev — Optimize LCP: fetchpriority](https://web.dev/articles/fetch-priority)
- [MDN — HTMLImageElement: fetchPriority](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/fetchPriority)
- [Google — Core Web Vitals: LCP](https://web.dev/articles/lcp)
- [Shopify Dawn — Hero section implementation](https://github.com/Shopify/dawn/blob/main/sections/image-banner.liquid)
