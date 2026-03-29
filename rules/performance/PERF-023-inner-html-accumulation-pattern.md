# PERF-023 | `innerHTML +=` pattern in cart/section JavaScript

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Inline `<script>` blocks in cart drawer sections, cart page sections, and any section JavaScript that dynamically appends or updates DOM content

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

The `innerHTML +=` pattern — semantically equivalent to `element.innerHTML = element.innerHTML + newContent` — performs three distinct operations in sequence: it serializes the element's entire existing DOM subtree to an HTML string (the read of `element.innerHTML`), concatenates that string with the new HTML string, and then parses the combined string back into DOM nodes (the write of `element.innerHTML`). The serialization and re-parsing steps are not free — they are proportional to the size and depth of the existing subtree. For a Shopify cart drawer containing 5 line items, each rendered as a `<div>` with nested image, title, variant, price, and remove-button elements, the existing subtree comprises approximately 50–80 DOM nodes. Adding a 6th item via `innerHTML +=` destroys all 50–80 existing nodes and recreates them, even though their content has not changed.

The destruction and recreation of existing DOM nodes has compounding consequences beyond CPU cost. Every event listener attached to existing child elements is destroyed when those elements are removed from the DOM. A cart drawer that attaches `click` handlers to each line item's remove button will lose all 5 handlers when the 6th item is added via `innerHTML +=` — unless the handlers are explicitly re-attached after every mutation, which is both a performance regression and a source of bugs. CSS transitions tied to existing elements are also reset: an item that was mid-way through a fade-in animation will start over from zero when its DOM node is destroyed and recreated. In Chrome DevTools Performance traces, `innerHTML +=` operations on a cart drawer with 8 items show 15–30ms of "Recalculate Style" and "Layout" work triggered by the full subtree replacement — work that `insertAdjacentHTML()` or `replaceChildren()` would reduce to 2–5ms for the same content addition.

The correct replacement depends on the intent. For appending new content (adding a line item to the end of a list): `element.insertAdjacentHTML('beforeend', newContent)` inserts the parsed HTML at the specified position without touching any existing nodes. For replacing all content (re-rendering the entire cart after an update): `element.replaceChildren(...fragment.children)` or setting `element.innerHTML = newContent` (without the `+= ` accumulation) is appropriate — the full replacement is intentional and the existing nodes were already stale. The distinction matters: `innerHTML +=` is almost never the correct choice because any case where it is used is either an append (use `insertAdjacentHTML`) or a full replace (use `innerHTML =` without accumulation).

---

## Detection Logic (AST)

1. **Identify `HtmlRawElement[script]` nodes containing JavaScript source.** The AST walker collects `HtmlRawElement` nodes with `tagName === "script"` that have no `src` attribute (inline scripts) or have `type` values other than `"speculationrules"` or `"application/json"`. Extract the `RawMarkup` content for JavaScript analysis.

2. **Scan script content for `innerHTML +=` assignment patterns.** Within the extracted JavaScript text, apply pattern matching for the assignment operator sequence `innerHTML +=`. The pattern covers whitespace variants (`innerHTML+=`, `innerHTML +=`, `innerHTML  +=`). Also flag the equivalent explicit form: assignment where the right-hand side is a concatenation expression beginning with a member access on the same element reading `.innerHTML` (e.g., `el.innerHTML = el.innerHTML + `).

3. **Classify the surrounding context.** Inspect the variable name or expression on the left side of `innerHTML +=`. If the element being mutated is referenced by a selector containing `cart`, `drawer`, `line-item`, `cart-items`, or `cart-drawer`, classify as HIGH severity — cart DOM mutations with this pattern destroy and recreate the entire cart line item list. For other element contexts, classify as MEDIUM.

4. **Verify that the append is genuinely accumulative.** If the `innerHTML +=` pattern appears inside a loop (`for`, `while`, `forEach`, `map`), severity escalates to HIGH regardless of element context — loop-accumulation using `innerHTML +=` is O(n²) in both string concatenation and DOM parse work, as each iteration re-serializes and re-parses an increasingly large subtree. Include a note in the diagnostic recommending pre-building an HTML string in the loop and assigning it once outside the loop.

**Why regex fails here:** A regex matching `innerHTML +=` in raw script text cannot distinguish between commented-out code (a developer who noted the anti-pattern in a comment), string literals containing the text `innerHTML +=` (log messages, documentation strings), and actual DOM mutation code. The AST walker — even when operating on JavaScript as raw markup rather than a parsed JS AST — can apply heuristics based on surrounding code structure (presence of `;`, surrounding `function` or `class` keywords, proximity to `addEventListener` calls) to improve signal quality. For JavaScript AST-aware linters, the detection is exact: `AssignmentExpression` where `operator === "+="` and `left` is a `MemberExpression` with `property.name === "innerHTML"`.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: innerHTML += used to add cart line items and update the
  cart drawer, causing full subtree serialization and re-parse on every
  cart mutation.
{%- endcomment -%}

<script>
  class CartDrawer extends HTMLElement {
    constructor() {
      super();
      this.cartItemsEl = this.querySelector('.cart-drawer__items');
      this.cartFooterEl = this.querySelector('.cart-drawer__footer');
    }

    // Anti-pattern: innerHTML += destroys and recreates all existing
    // line item nodes when appending a new item. 5 items = 50-80 DOM
    // nodes destroyed and recreated to add 1 new item.
    async addLineItem(variantId, quantity) {
      const response = await fetch(`${window.Shopify.routes.root}cart/add.js`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ id: variantId, quantity })
      });
      const lineItem = await response.json();

      // Destroys all remove-button event listeners attached to existing items
      this.cartItemsEl.innerHTML += this.renderLineItem(lineItem);

      // Updates footer — same innerHTML += problem, destroys existing footer nodes
      const cart = await (await fetch(`${window.Shopify.routes.root}cart.js`)).json();
      this.cartFooterEl.innerHTML += `<span class="cart-count-update">${cart.item_count}</span>`;
    }

    renderLineItem(item) {
      return `
        <div class="cart-item" data-key="${item.key}">
          <img src="${item.image}" alt="${item.title}" loading="lazy">
          <div class="cart-item__details">
            <p class="cart-item__name">${item.product_title}</p>
            <p class="cart-item__variant">${item.variant_title}</p>
            <p class="cart-item__price">${Shopify.formatMoney(item.final_line_price)}</p>
          </div>
          <button class="cart-item__remove" data-key="${item.key}">Remove</button>
        </div>
      `;
    }
  }

  customElements.define('cart-drawer', CartDrawer);
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Replace innerHTML += with insertAdjacentHTML for appends and
  innerHTML = (without accumulation) for full replacements.

  Change 1: insertAdjacentHTML('beforeend', html) inserts new HTML at the
  end of the element's existing content without touching any existing DOM
  nodes. The existing line items and their event listeners are preserved
  exactly. Parse cost is proportional to the new content only, not the
  existing subtree.

  Change 2: For the footer update (full content replacement, not append),
  innerHTML = without += is used. The footer is replaced entirely because
  the cart count data has changed for all items, not just the new one.
  This is intentional — the distinction from += is that this is a known
  full replacement, not an accumulation.

  Change 3: Event delegation on the cart items container (listening on the
  parent rather than individual remove buttons) means event listeners survive
  any innerHTML replacement of child content. This eliminates the listener
  leak that innerHTML += was causing.

  Change 4: For bulk re-renders (full cart refresh), build the complete HTML
  string in one pass and assign it once — never accumulate inside a loop.
{%- endcomment -%}

<script>
  class CartDrawer extends HTMLElement {
    constructor() {
      super();
      this.cartItemsEl = this.querySelector('.cart-drawer__items');
      this.cartFooterEl = this.querySelector('.cart-drawer__footer');
      // Event delegation: listeners on parent survive child DOM changes
      this.cartItemsEl.addEventListener('click', this.handleItemClick.bind(this));
    }

    async addLineItem(variantId, quantity) {
      const response = await fetch(`${window.Shopify.routes.root}cart/add.js`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ id: variantId, quantity })
      });
      const lineItem = await response.json();

      // insertAdjacentHTML: no existing nodes destroyed, no listener loss
      this.cartItemsEl.insertAdjacentHTML('beforeend', this.renderLineItem(lineItem));

      const cart = await (await fetch(`${window.Shopify.routes.root}cart.js`)).json();
      // Full replacement (not accumulation) — intentional, correct
      this.cartFooterEl.innerHTML = this.renderFooter(cart);
    }

    handleItemClick(event) {
      const removeBtn = event.target.closest('.cart-item__remove');
      if (removeBtn) this.removeLineItem(removeBtn.dataset.key);
    }

    renderLineItem(item) {
      return `
        <div class="cart-item" data-key="${item.key}">
          <img src="${item.image}" alt="${item.title}" loading="lazy" width="80" height="80">
          <div class="cart-item__details">
            <p class="cart-item__name">${item.product_title}</p>
            <p class="cart-item__variant">${item.variant_title || ''}</p>
            <p class="cart-item__price">${Shopify.formatMoney(item.final_line_price)}</p>
          </div>
          <button class="cart-item__remove" data-key="${item.key}" aria-label="Remove ${item.product_title}">
            Remove
          </button>
        </div>
      `;
    }

    renderFooter(cart) {
      return `<p class="cart-drawer__total">Total: ${Shopify.formatMoney(cart.total_price)}</p>`;
    }
  }

  customElements.define('cart-drawer', CartDrawer);
</script>
```

---

## References

- [MDN — `Element.insertAdjacentHTML()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/insertAdjacentHTML)
- [MDN — `Element.innerHTML`](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML)
- [MDN — `Element.replaceChildren()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/replaceChildren)
- [web.dev — Avoid large, complex layouts and layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing)
- [Shopify — Building performant theme JavaScript](https://shopify.dev/docs/storefronts/themes/best-practices/performance/javascript)
