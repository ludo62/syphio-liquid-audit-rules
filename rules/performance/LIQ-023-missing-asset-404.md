# LIQ-023 | `| asset_url` on an asset not present in `/assets/`

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** All templates using `| asset_url`; section files, layout files, snippets referencing CSS, JavaScript, font, or image assets

**Added:** 2025-01 · **Last updated:** 2026-03

---

## Context

`| asset_url` generates a Shopify CDN URL for a file in the theme's `/assets/` directory. The filter constructs the URL by concatenating the CDN base path, the theme ID, and the provided filename — it performs no validation that the file actually exists in the theme's asset manifest. If the referenced file is absent from `/assets/`, the generated URL returns an HTTP 404 response. Shopify's CDN does not cache 404 responses from asset URLs, meaning every request to the page triggers a new origin request for the missing asset. For a missing CSS file, the consequence is unstyled content across every page load. For a missing JavaScript file, the consequence is broken interactivity — and any JavaScript that depends on the missing module will throw a `ReferenceError` or fail silently depending on how the file is loaded.

The failure mode is disproportionately common in two scenarios. First, during local development on macOS: the macOS filesystem is case-insensitive, so `{{ 'CustomFont.woff2' | asset_url }}` resolves correctly to `assets/customfont.woff2` in local testing. Shopify's CDN infrastructure runs on Linux with a case-sensitive filesystem — the production URL for `CustomFont.woff2` returns 404 if the file in `/assets/` is named `customfont.woff2`. This case-sensitivity mismatch produces a defect that is invisible in development and only surfaces in production or staging. Second, when a developer references an asset filename in a Liquid template but forgets to include the asset file when committing or pushing the theme — the template is pushed, the asset is not, and the URL generates a valid-looking CDN path that 404s for every visitor.

Theme Store automated testing crawls themes with a real browser and checks for console errors and failed network requests. A 404 on a CSS or JavaScript asset file triggers an automatic failure of the Network Requests check in Theme Store review. Even for themes not submitted to the Theme Store, a 404 asset URL is surfaced as a performance issue in Shopify's built-in Theme Inspector and Google Lighthouse reports (as a failed resource load). The rule applies to all `| asset_url` usages where the filename can be statically resolved — dynamic filenames constructed via `| append` are excluded from static analysis but flagged for manual review.

---

## Detection Logic (AST)

1. **Enumerate the theme's asset manifest.** Before AST analysis begins, collect the full set of filenames present in the theme's `/assets/` directory. Normalize all filenames to their exact case for comparison (do not apply case-folding).
2. **Identify `LiquidFilter` nodes with filter name `asset_url`.** Walk the AST for all `LiquidFilter` nodes where `filter.name === 'asset_url'`. These represent `| asset_url` filter applications.
3. **Resolve the string argument.** Check the input expression of the `asset_url` filter — the value to its left in the filter chain. If the input is a `StringLiteral` node, extract the literal string value. This is the filename being referenced.
4. **Check the filename against the asset manifest.** Perform a case-sensitive lookup of the extracted filename in the enumerated asset manifest from step 1. If the filename is not found, emit a HIGH violation specifying the missing filename and the template location.
5. **Handle dynamic filenames conservatively.** If the input to `asset_url` is not a `StringLiteral` — it is a `VariableLookup`, a concatenation expression, or a filter chain output — emit a MEDIUM informational note that the asset reference cannot be statically verified and should be reviewed manually. Do not emit a HIGH violation for non-static filenames.

**Why regex fails here:** A regex matching `'filename.css' | asset_url` can extract the quoted filename string, but it cannot perform a case-sensitive lookup against the theme's actual asset manifest — the manifest is a filesystem artifact, not a code artifact. More critically, regex cannot distinguish `'theme.css' | asset_url` (string literal — verifiable) from `file_name | asset_url` (variable — not statically verifiable). This distinction is critical: flagging all dynamic asset references as violations would produce unacceptable false-positive rates, while flagging none would miss the common static-reference case. AST analysis provides the `StringLiteral` node type to make this distinction cleanly.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: asset_url references filenames that do not exist in /assets/.
  'CustomIcons.svg' has a capital C and I — the actual file is 'customicons.svg'
  (case mismatch: works on macOS dev, 404s on Shopify CDN in production).
  'product-zoom.js' was never added to the theme's /assets/ directory — the
  developer wrote the reference in the template but forgot to upload the file.
  Both generate valid-looking CDN URLs that return 404 on every page load.
{%- endcomment -%}

{%- comment -%} Case mismatch: CDN is case-sensitive, macOS is not {%- endcomment -%}
<link rel="stylesheet" href="{{ 'Theme-Styles.css' | asset_url }}">

{%- comment -%} File never uploaded to /assets/ — 404 in production {%- endcomment -%}
<script src="{{ 'product-zoom.js' | asset_url }}" defer></script>

{%- comment -%} Font file referenced but not present in /assets/ {%- endcomment -%}
{%- assign font_url = 'GreatVibes-Regular.woff2' | asset_url -%}
<style>
  @font-face {
    font-family: 'Great Vibes';
    src: url('{{ font_url }}') format('woff2');
    font-display: swap;
  }
</style>

{%- comment -%} Icon sprite — file exists but with different extension {%- endcomment -%}
<img src="{{ 'icons-sprite.svg' | asset_url }}" alt="" aria-hidden="true">
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Ensure every filename passed to | asset_url exactly matches — including
  case — a file present in /assets/. Use lowercase filenames throughout to
  eliminate case-sensitivity mismatches between macOS development and Shopify CDN.
  Verify the /assets/ directory before deploying: run `shopify theme push --only assets`
  and confirm no 404s appear in browser DevTools Network panel on the production theme.
  For fonts and icons, use the exact filename as stored in /assets/ — no assumptions.
{%- endcomment -%}

{%- comment -%} Filename matches /assets/theme-styles.css exactly (lowercase) {%- endcomment -%}
<link rel="stylesheet" href="{{ 'theme-styles.css' | asset_url }}">

{%- comment -%} File confirmed present in /assets/product-zoom.js after push {%- endcomment -%}
<script src="{{ 'product-zoom.js' | asset_url }}" defer></script>

{%- comment -%} Font file: exact case match with /assets/great-vibes-regular.woff2 {%- endcomment -%}
{%- assign font_url = 'great-vibes-regular.woff2' | asset_url -%}
<style>
  @font-face {
    font-family: 'Great Vibes';
    src: url('{{ font_url }}') format('woff2');
    font-display: swap;
  }
</style>

{%- comment -%} Icon sprite: exact filename match including extension {%- endcomment -%}
<img src="{{ 'icons-sprite.svg' | asset_url }}" alt="" aria-hidden="true">
```

---

## References

- [Shopify Liquid: asset_url filter](https://shopify.dev/docs/api/liquid/filters/asset_url)
- [Shopify: Theme assets — naming conventions](https://shopify.dev/docs/themes/architecture/assets)
- [Shopify CLI: theme push command](https://shopify.dev/docs/themes/tools/cli/commands#push)
- [Shopify Theme Store: Network Requests check](https://shopify.dev/docs/themes/store/requirements)
