# PERF-025 | `document.querySelector` instead of `this.querySelector` in Web Component

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Inline `<script>` blocks and JavaScript files defining Web Components used in Shopify sections — particularly any section that can appear multiple times on a page

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

`document.querySelector(selector)` searches the entire document object model from the document root downward and returns the first element that matches the selector, regardless of which section instance it belongs to. In a Shopify theme where the same section template can be added multiple times to a page — two "Featured Collection" sections, two "Image with Text" sections, two "Testimonials" carousels — every Web Component instance that uses `document.querySelector` to find its internal elements will find elements from the first DOM instance of that component, not from the current instance. The component's `this` reference correctly identifies the current instance (the Web Component element itself), while `document.querySelector` silently returns elements from an entirely different section instance. This is not a visible error in development (where typically only one instance exists per section type); it is a defect that manifests only when a merchant adds the same section to a page twice.

The functional consequences are severe and non-obvious. Consider a "Product Tabs" Web Component that uses `document.querySelector('.product-tabs__content')` to find its content pane: when two "Product Tabs" sections exist on a page, both instances' click handlers target the content pane of the first instance. Clicking a tab in the second section changes the content in the first section and does nothing in the second. The second section appears completely broken. For slideshow components using `document.querySelector('.slideshow__slide')`, both slideshows advance slides in the first slideshow's DOM. Cart drawer components using `document.querySelector('.cart-drawer')` open the correct drawer (there is typically one cart drawer), but this is by coincidence of there being only one — the bug lies dormant until a second drawer-like element is added. In Shopify's Theme Editor, where merchants can freely duplicate sections, this defect is encountered frequently enough that it appears regularly in Shopify Partner community support threads.

`this.querySelector(selector)` searches only within the Web Component element's own subtree — the DOM nodes that are descendants of the component element itself. Because each section instance wraps its content inside its own Web Component element (e.g., `<featured-collection data-section-id="abc123">`), `this.querySelector` is scoped to exactly the correct instance. The selector space is also smaller, making `this.querySelector` fractionally faster than `document.querySelector` for deep documents (though this performance difference is secondary to the correctness concern). The only legitimate use of `document.querySelector` within a Web Component is for elements that are genuinely document-global: the `<body>` element, a single global header, a single cart drawer, a single modal overlay, or elements in the `<head>`. For all section-internal elements, `this.querySelector` is the correct and only safe choice.

---

## Detection Logic (AST)

1. **Identify Web Component class definitions.** The AST walker scans `HtmlRawElement[script]` node content for class definitions containing `connectedCallback`, and for `customElements.define(...)` call expressions. Collect the full class body text regions as the scope for further analysis.

2. **Detect `document.querySelector` and `document.querySelectorAll` calls within class method bodies.** Within each identified class body, scan for occurrences of `document.querySelector(` and `document.querySelectorAll(`. These are candidate violations. Also flag `document.getElementById(` and `document.getElementsByClassName(` — all document-root search methods have the same scoping defect.

3. **Classify the selector argument by scope.** Extract the string argument of each `document.querySelector` call. Apply a classification heuristic: if the selector matches a known document-global pattern (`.cart-drawer`, `#shopify-section-header`, `body`, `html`, `.overlay`, `[data-cart-drawer]`), classify as LOW advisory (may be intentional global access). If the selector matches a pattern that is plausibly section-internal (contains class names with component-specific prefixes, matches data attributes like `data-section-id`, or contains class names that appear in the same section's Liquid template), classify as HIGH violation.

4. **Check for `this.querySelector` alternatives in the same class.** If the class body also contains `this.querySelector` calls, emit an informational note that the class uses mixed scoping strategies, and flag the `document.querySelector` calls specifically. If the class body uses only `document.querySelector` with no `this.querySelector` calls at all, the pattern is pervasive — emit HIGH severity on all instances and recommend a systematic replacement.

**Why regex fails here:** A regex matching `document.querySelector` in script content cannot determine whether the call is inside a Web Component class method body (where it is problematic) versus a standalone utility function (where it may be acceptable). It also cannot distinguish between calls with section-internal selectors (violations) and calls with document-global selectors (acceptable). Both classifications require understanding the call's context within the class structure and the semantic scope implied by the selector string — analysis that requires parsing the JavaScript structure, not scanning raw text line by line.

---

## ❌ Anti-Pattern

```liquid

{%- comment -%}

  ANTI-PATTERN: document.querySelector used inside a Web Component to find

  section-internal elements. Works with one section instance, breaks silently

  when the merchant adds a second instance of the same section.

{%- endcomment -%}



<script>

  class FeaturedCollectionTabs extends HTMLElement {

    connectedCallback() {

      // Anti-pattern: finds the FIRST .tab-list in the document, not this

      // component's tab list. Second instance controls first instance's tabs.

      this.tabList = document.querySelector('.featured-collection__tab-list');



      // Anti-pattern: finds ALL .tab-button elements in the document.

      // If two instances exist, this.tabs contains buttons from BOTH instances.

      this.tabs = document.querySelectorAll('.featured-collection__tab-button');



      // Anti-pattern: finds the first .tab-panels container in the document.

      // Second instance's tab clicks show content in first instance's panels.

      this.panelsContainer = document.querySelector('.featured-collection__panels');



      this.tabs.forEach((tab, index) => {

        tab.addEventListener('click', () => this.activateTab(index));

      });



      // Activate the first tab on mount

      this.activateTab(0);

    }



    activateTab(index) {

      this.tabs.forEach((tab, i) => {

        tab.setAttribute('aria-selected', i === index ? 'true' : 'false');

      });



      // Anti-pattern: querySelectorAll on document — selects panels from all instances

      const panels = document.querySelectorAll('.featured-collection__panel');

      panels.forEach((panel, i) => {

        panel.hidden = i !== index;

      });

    }

  }



  customElements.define('featured-collection-tabs', FeaturedCollectionTabs);

</script>



<featured-collection-tabs data-section-id="{{ section.id }}">

  <div class="featured-collection__tab-list" role="tablist">

    {%- for block in section.blocks -%}

      <button

        class="featured-collection__tab-button"

        role="tab"

        aria-selected="{% if forloop.first %}true{% else %}false{% endif %}"

        {{ block.shopify_attributes }}

      >

        {{ block.settings.tab_label }}

      </button>

    {%- endfor -%}

  </div>



  <div class="featured-collection__panels">

    {%- for block in section.blocks -%}

      <div class="featured-collection__panel" {% unless forloop.first %}hidden{% endunless %}>

        {%- render 'product-card' for block.settings.collection.products as product -%}

      </div>

    {%- endfor -%}

  </div>

</featured-collection-tabs>

```

---

## ✅ Optimized Fix

```liquid

{%- comment -%}

  FIX: Replace document.querySelector with this.querySelector throughout

  the Web Component to correctly scope DOM queries to the current instance.



  Change 1: this.querySelector('.featured-collection__tab-list') searches

  only within this component's own subtree. When two instances exist,

  each instance's connectedCallback finds its own tab list — not the other's.



  Change 2: this.querySelectorAll('.featured-collection__tab-button') returns

  only buttons within this instance. There is no cross-instance contamination

  of the tabs NodeList.



  Change 3: All DOM queries inside activateTab also use this.querySelectorAll

  to scope panel selection to the current instance. The method is called with

  the correct `this` binding via arrow function callbacks.



  Change 4: document.querySelector is used only for the cart drawer reference

  — a genuinely document-global singleton element. This is the correct use

  of document.querySelector: elements that are intentionally shared across

  all section instances and exist exactly once in the document.

{%- endcomment -%}



<script>

  class FeaturedCollectionTabs extends HTMLElement {

    connectedCallback() {

      // this.querySelector: scoped to this component instance only

      this.tabList = this.querySelector('.featured-collection__tab-list');



      // this.querySelectorAll: returns only this instance's buttons

      this.tabs = this.querySelectorAll('.featured-collection__tab-button');



      // this.querySelector: scoped to this component's panels container

      this.panelsContainer = this.querySelector('.featured-collection__panels');



      this.tabs.forEach((tab, index) => {

        tab.addEventListener('click', () => this.activateTab(index));

      });



      // Activate the first tab on mount

      this.activateTab(0);

    }



    activateTab(index) {

      this.tabs.forEach((tab, i) => {

        tab.setAttribute('aria-selected', i === index ? 'true' : 'false');

      });



      // this.querySelectorAll: scoped to this instance's panels only

      const panels = this.querySelectorAll('.featured-collection__panel');

      panels.forEach((panel, i) => {

        panel.hidden = i !== index;

      });

    }

  }



  customElements.define('featured-collection-tabs', FeaturedCollectionTabs);

</script>



<featured-collection-tabs data-section-id="{{ section.id }}">

  <div class="featured-collection__tab-list" role="tablist">

    {%- for block in section.blocks -%}

      <button

        class="featured-collection__tab-button"

        role="tab"

        aria-selected="{% if forloop.first %}true{% else %}false{% endif %}"

        {{ block.shopify_attributes }}

      >

        {{ block.settings.tab_label }}

      </button>

    {%- endfor -%}

  </div>



  <div class="featured-collection__panels">

    {%- for block in section.blocks -%}

      <div class="featured-collection__panel" {% unless forloop.first %}hidden{% endunless %}>

        {%- render 'product-card' for block.settings.collection.products as product -%}

      </div>

    {%- endfor -%}

  </div>

</featured-collection-tabs>

```

---

## References

- [MDN — `Element.querySelector()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/querySelector)

- [MDN — `Document.querySelector()`](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelector)

- [MDN — Using custom elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements)

- [Shopify — Web Components in themes](https://shopify.dev/docs/storefronts/themes/best-practices/performance/javascript#web-components)

- [Shopify — Section rendering architecture](https://shopify.dev/docs/storefronts/themes/architecture/sections)

- [Shopify Dawn — Web Component scoping patterns](https://github.com/Shopify/dawn)
