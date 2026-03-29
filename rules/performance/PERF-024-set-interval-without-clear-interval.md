# PERF-024 | `setInterval` without `clearInterval` in Web Component

**Category:** Performance

**Severity:** 🟠 HIGH

**Affects:** Inline `<script>` blocks and JavaScript files defining Web Components (`customElements.define`) used in Shopify sections — particularly slideshow, countdown timer, and announcement bar sections

**Added:** 2026-03 · **Last updated:** 2026-03

---

## Context

Shopify's Theme Editor calls `connectedCallback` on a Web Component when the component is added to the DOM, and calls `disconnectedCallback` when the component is removed. During theme customization, every setting change that causes a section to re-render triggers a `disconnectedCallback` + `connectedCallback` cycle on all Web Components within that section — the section is removed and re-injected into the DOM with the updated settings. If `setInterval` is started in `connectedCallback` without a corresponding `clearInterval` in `disconnectedCallback`, each re-render cycle adds a new active interval timer while the previous one continues running. After 10 setting changes — a routine session for a merchant configuring a slideshow's timing, colors, and content — 10 simultaneous interval callbacks are executing, each driving the same slideshow or countdown logic against a DOM that only one of them can correctly reference.

The consequences of timer accumulation are measurable and progressive. Each additional stale interval consumes CPU time at the interval period (e.g., a 3-second slideshow cycle executing 10 simultaneous callbacks consumes CPU every 3 seconds for all 10 timers). Stale timers reference the component instance via closure — those closures hold references to the component's DOM subtree, preventing garbage collection of the entire component tree even after `disconnectedCallback` has been called. In the Chrome DevTools Memory tab, leaked Web Component instances appear as detached DOM trees that retain their full subtree in memory indefinitely. For a slideshow with 8 slides and associated image elements, each leaked instance can retain 2–5MB of DOM and associated image data in memory. After 20 Theme Editor interactions, the memory retention becomes noticeable as increased tab memory usage and GC pressure.

The fix is a single stored reference pattern: assign the return value of `setInterval` to `this._intervalId` in `connectedCallback`, and call `clearInterval(this._intervalId)` as the first statement in `disconnectedCallback`. This is not optional boilerplate — it is the required cleanup contract for any resource acquired in `connectedCallback`. The same pattern applies to `setTimeout` (use `this._timeoutId` + `clearTimeout`), `requestAnimationFrame` (use `this._rafId` + `cancelAnimationFrame`), and `addEventListener` on elements outside the component's own subtree (use `removeEventListener` with the same handler reference in `disconnectedCallback`). Shopify's Web Component lifecycle documentation and the Dawn reference theme explicitly implement this pattern in all timer-using components.

---

## Detection Logic (AST)

1. **Identify Web Component class definitions with `connectedCallback` methods.** The AST walker scans `HtmlRawElement[script]` node content for class definitions that contain a `connectedCallback` method. This includes both `class Foo extends HTMLElement` patterns and anonymous class expressions passed to `customElements.define(...)`. Collect the class body as a text region for further analysis.

2. **Detect `setInterval` calls within `connectedCallback`.** Within the identified `connectedCallback` method body, scan for `setInterval(` call expressions. Flag any `setInterval` call that is present in `connectedCallback`. The return value assignment is inspected in the next step.

3. **Verify that the interval ID is stored on `this`.** For each `setInterval` call found in `connectedCallback`, inspect whether its return value is assigned to a `this.*` property (e.g., `this._intervalId =`, `this.intervalId =`, `this._timer =`). If the return value is not stored, or is stored in a local variable (not on `this`), it cannot be accessed in `disconnectedCallback` — flag as HIGH severity.

4. **Verify that `clearInterval` is called in `disconnectedCallback` with the stored ID.** Scan the class body for a `disconnectedCallback` method. Within it, look for a `clearInterval(this._intervalId)` call (or whatever property name was used in step 3). If `disconnectedCallback` is absent from the class entirely, emit HIGH severity — there is no cleanup path at all. If `disconnectedCallback` is present but does not call `clearInterval` with the stored ID, emit HIGH severity. If `clearInterval` is called with a different variable that does not match the stored ID, emit MEDIUM advisory.

**Why regex fails here:** A regex matching `setInterval` without `clearInterval` in the same file produces false positives for files that correctly call `clearInterval` in a method that the regex does not associate with the same class's `disconnectedCallback`. Determining that a `clearInterval` call is in the `disconnectedCallback` of the same class as the `setInterval` call in `connectedCallback` requires tracking class boundaries, method membership, and the `this` property used to store the interval ID — three levels of scope analysis that require AST traversal or a JavaScript parser, not string pattern matching.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: setInterval started in connectedCallback without clearInterval
  in disconnectedCallback. Each Theme Editor setting change re-renders the
  section and adds a new timer without removing the old one.
{%- endcomment -%}

<script>
  class SlideshowComponent extends HTMLElement {
    connectedCallback() {
      this.slides = Array.from(this.querySelectorAll('.slideshow__slide'));
      this.currentIndex = 0;

      // Anti-pattern: interval ID not stored; cannot be cleared later.
      // Each connectedCallback call (Theme Editor re-render) adds a new
      // timer. After 10 re-renders, 10 timers fire simultaneously.
      setInterval(() => {
        this.currentIndex = (this.currentIndex + 1) % this.slides.length;
        this.goToSlide(this.currentIndex);
      }, this.dataset.speed || 3000);

      // Anti-pattern: event listener added without corresponding removeEventListener.
      // After each re-render, another listener accumulates on document.
      document.addEventListener('keydown', (e) => {
        if (e.key === 'ArrowRight') this.nextSlide();
        if (e.key === 'ArrowLeft') this.prevSlide();
      });
    }

    goToSlide(index) {
      this.slides.forEach((slide, i) => {
        slide.classList.toggle('is-active', i === index);
      });
    }

    nextSlide() {
      this.currentIndex = (this.currentIndex + 1) % this.slides.length;
      this.goToSlide(this.currentIndex);
    }

    prevSlide() {
      this.currentIndex = (this.currentIndex - 1 + this.slides.length) % this.slides.length;
      this.goToSlide(this.currentIndex);
    }
  }

  customElements.define('slideshow-component', SlideshowComponent);
</script>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Store interval ID on this and clear it in disconnectedCallback.
  Also fix the event listener leak with a stored bound handler reference.

  Change 1: this._intervalId stores the setInterval return value.
  clearInterval(this._intervalId) in disconnectedCallback cancels the timer
  before the component is removed from the DOM. Each Theme Editor re-render
  clears the previous timer before starting a new one — timer count stays at 1.

  Change 2: this._keydownHandler stores a bound reference to the keyboard
  handler. The same reference is passed to both addEventListener and
  removeEventListener, which is required for correct listener removal.
  Arrow functions assigned inline cannot be removed — a new function
  reference is created on each addEventListener call.

  Change 3: disconnectedCallback clears all acquired resources in order:
  timer first, then event listener. The order matches reverse acquisition
  order to avoid state inconsistencies during teardown.

  Change 4: pauseOnHover pattern uses the same stored-ID approach — the
  interval is cleared on mouseenter and restarted on mouseleave, with
  the new ID stored back to this._intervalId each time.
{%- endcomment -%}

<script>
  class SlideshowComponent extends HTMLElement {
    connectedCallback() {
      this.slides = Array.from(this.querySelectorAll('.slideshow__slide'));
      this.currentIndex = 0;
      this._speed = parseInt(this.dataset.speed, 10) || 3000;

      // Store interval ID on this so disconnectedCallback can clear it
      this._intervalId = setInterval(() => {
        this.currentIndex = (this.currentIndex + 1) % this.slides.length;
        this.goToSlide(this.currentIndex);
      }, this._speed);

      // Store bound handler reference — required for removeEventListener
      this._keydownHandler = this.handleKeydown.bind(this);
      document.addEventListener('keydown', this._keydownHandler);
    }

    disconnectedCallback() {
      // Clear timer — prevents stale timer accumulation on Theme Editor re-renders
      clearInterval(this._intervalId);

      // Remove listener using the same bound reference — prevents listener leak
      document.removeEventListener('keydown', this._keydownHandler);
    }

    handleKeydown(event) {
      if (event.key === 'ArrowRight') this.nextSlide();
      if (event.key === 'ArrowLeft') this.prevSlide();
    }

    goToSlide(index) {
      this.slides.forEach((slide, i) => {
        slide.classList.toggle('is-active', i === index);
      });
    }

    nextSlide() {
      this.currentIndex = (this.currentIndex + 1) % this.slides.length;
      this.goToSlide(this.currentIndex);
    }

    prevSlide() {
      this.currentIndex = (this.currentIndex - 1 + this.slides.length) % this.slides.length;
      this.goToSlide(this.currentIndex);
    }
  }

  customElements.define('slideshow-component', SlideshowComponent);
</script>
```

---

## References

- [MDN — `setInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setInterval)
- [MDN — `clearInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearInterval)
- [MDN — `HTMLElement.connectedCallback`](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements#connectedcallback)
- [MDN — `HTMLElement.disconnectedCallback`](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements#disconnectedcallback)
- [Shopify — Web Components in themes](https://shopify.dev/docs/storefronts/themes/best-practices/performance/javascript#web-components)
- [Shopify Dawn — SlideshowComponent implementation](https://github.com/Shopify/dawn)
