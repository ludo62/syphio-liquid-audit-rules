# LIQ-012 | `{% form %}` without `{% endform %}`

**Category:** Security

**Severity:** 🟠 HIGH

**CWE:** CWE-352 — Cross-Site Request Forgery

**OWASP:** A01:2021 — Broken Access Control

**Affects:** Any template using `{% form %}` for cart, contact, customer login, account creation, or product submission

**Added:** 2024-01 · **Last updated:** 2026-03

---

## Context

`{% form %}` is Shopify's Liquid block tag for generating HTML form elements. Its function is not purely presentational — the tag generates a CSRF authenticity token and injects it as a hidden input field (`<input type="hidden" name="authenticity_token" value="...">`), sets the correct `action` URL for the target Shopify endpoint, and sets `method="post"`. The authenticity token is a per-session, per-action cryptographic token that Shopify's server validates on every form submission. Without it, the server rejects the submission with a `422 Unprocessable Entity` response. This is the core CSRF protection mechanism for all Shopify storefront form actions — it prevents malicious third-party sites from submitting forms on behalf of authenticated customers without their knowledge.

When `{% endform %}` is missing, Shopify's Liquid parser does not close the form tag context. The consequence is twofold: the `<form>` HTML element's closing `</form>` tag is never emitted, leaving the form structurally unclosed in the DOM; and the authenticity token injection — which occurs at the `{% endform %}` boundary in some Liquid runtime implementations — is skipped entirely. An unclosed `<form>` element is interpreted by browsers according to the HTML5 parsing algorithm, which attempts to auto-close the form at the nearest valid closure point. This behavior is browser-specific and inconsistent — in some browsers, subsequent `<input>`, `<button>`, and `<select>` elements on the page that are outside the intended form scope are treated as form fields and included in the submission payload. This can cause customer shipping addresses, saved payment method tokens, or other sensitive form data to be submitted alongside unrelated form actions.

The security classification maps to CWE-352 (CSRF) because the missing `{% endform %}` effectively bypasses Shopify's CSRF token injection for the affected form, and to OWASP A01:2021 (Broken Access Control) because the server-side validation that controls which actions an unauthenticated or differently-authenticated session may perform is defeated. In practice, the immediate observed symptom is a 422 response on form submission — the customer cannot add to cart, cannot log in, cannot submit a contact form. In the worst case (missing `endform` on a login or account-creation form), the broken form boundary may cause subsequent form fields on the page to be included in the login POST request, potentially leaking data.

---

## Detection Logic (AST)

1. **Collect all `LiquidTag[name=form]` nodes.** The AST walker traverses the template tree and collects every `LiquidTag` where `name === "form"`. These are `LiquidTagOpen` nodes — Shopify's `{% form %}` is always a block tag and is never self-closing. The tag's `markup` contains the form type identifier (`'product'`, `'contact'`, `'customer_login'`, `'create_customer'`, `'recover_customer_password'`, `'cart'`).

2. **Apply stack-based open/close matching.** For each `LiquidTag[form]` encountered, a stack entry is pushed with the form type and source position. For each `LiquidTag[endform]` encountered, the stack is popped. After full template traversal, any remaining stack entries are unclosed `{% form %}` tags — each is a violation filed with the form type and source position.

3. **Inspect the form body for `authenticity_token` hidden input.** Within correctly closed `{% form %}...{% endform %}` blocks, the walker verifies that the generated output would include the CSRF token. This is a documentation check — Shopify's runtime injects the token automatically at render time, so the AST cannot observe it directly. However, if the developer has added a manual `<input type="hidden" name="authenticity_token">` within the block (a common anti-pattern copied from non-Shopify frameworks), this is flagged as a redundant token that may conflict with Shopify's injected token.

4. **Check for `<form>` HTML elements without Liquid `{% form %}` wrapper.** As a complementary check, the walker identifies `HtmlElement[tag=form]` nodes that do not have a `LiquidTag[form]` ancestor. Raw HTML `<form>` elements on Shopify storefronts submit without CSRF tokens — any submission to a Shopify endpoint will return 422. These are flagged as a separate HIGH-severity violation with a recommendation to use `{% form %}`.

**Why regex fails here:** A regex matching `{%\s*form\s+` and counting occurrences cannot determine whether the tag is inside a `{% comment %}` block (documentation) or inside a `{% if %}` branch that only executes for authenticated customers. The AST walker excludes `LiquidTag[comment]` descendants from the count and can inspect `LiquidTag[if]` condition logic to determine whether the form is conditionally rendered — providing context-aware violation reporting that regex cannot approximate.

---

## ❌ Anti-Pattern

```liquid
{%- comment -%}
  ANTI-PATTERN: {% form %} blocks without {% endform %}.

  Failure mode 1: Product form — missing endform.
  Shopify does not emit the closing </form> or CSRF token.
  Submission returns 422 Unprocessable Entity.
  Add-to-cart is completely broken. No error shown to customer.

  Failure mode 2: Contact form — missing endform.
  Same CSRF failure. Contact form submissions rejected server-side.
  Customer sees no confirmation, no error — silent failure.

  Failure mode 3: Unclosed <form> DOM structure — browser's HTML parser
  attempts to auto-close the form at an arbitrary boundary.
  Subsequent form fields on the page (search input, newsletter signup)
  may be interpreted as part of the unclosed product form's submission payload.
{%- endcomment -%}

<div class="product-page">
  <h1>{{ product.title }}</h1>
  <p>{{ product.price | money }}</p>

  {%- comment -%} Product form — endform missing -%}
  {% form 'product', product %}
    <select name="id" aria-label="{{ 'products.product.variant_selector' | t }}">
      {% for variant in product.variants %}
        <option
          value="{{ variant.id }}"
          {% if variant == product.selected_or_first_available_variant %}selected{% endif %}
        >
          {{ variant.title }} — {{ variant.price | money }}
        </option>
      {% endfor %}
    </select>

    <div class="quantity-selector">
      <label for="quantity">{{ 'products.product.quantity' | t }}</label>
      <input type="number" id="quantity" name="quantity" value="1" min="1">
    </div>

    <button type="submit" name="add">
      {{ 'products.product.add_to_cart' | t }}
    </button>

{%- comment -%} endform is missing here — CSRF token never injected -%}

<div class="contact-section">
  <h2>Contact us</h2>
  {%- comment -%} Contact form — also missing endform -%}
  {% form 'contact' %}
    <label for="contact-email">Email</label>
    <input type="email" id="contact-email" name="contact[email]" required>

    <label for="contact-message">Message</label>
    <textarea id="contact-message" name="contact[body]" required></textarea>

    <button type="submit">Send</button>

{%- comment -%} endform missing here too -%}
</div>
```

---

## ✅ Optimized Fix

```liquid
{%- comment -%}
  FIX: Close every {% form %} block with {% endform %}.

  Change 1: {% endform %} added immediately after the last form element.
  Shopify injects: closing </form> tag, CSRF authenticity_token hidden input,
  method="post" attribute, and the correct action URL for the endpoint.

  Change 2: form type strings match Shopify's documented form type identifiers.
  'product' requires the product object as second argument.
  'contact' requires no additional argument.

  Change 3: {% form %} error handling — Shopify populates form.errors after
  a failed submission. The errors object is only available within the
  {% form %}...{% endform %} block scope.
{%- endcomment -%}

<div class="product-page">
  <h1>{{ product.title }}</h1>
  <p>{{ product.price | money }}</p>

  {%- comment -%}
    Product form: Shopify injects authenticity_token, action="/cart/add",
    method="post". CSRF protection is active. endform closes the context.
  {%- endcomment -%}
  {% form 'product', product %}
    {% if form.errors %}
      <div class="form-errors" role="alert">
        {{ form.errors | default_errors }}
      </div>
    {% endif %}

    <select name="id" aria-label="{{ 'products.product.variant_selector' | t }}">
      {% for variant in product.variants %}
        <option
          value="{{ variant.id }}"
          {% if variant == product.selected_or_first_available_variant %}selected{% endif %}
          {% unless variant.available %}disabled{% endunless %}
        >
          {{ variant.title }} — {{ variant.price | money }}
        </option>
      {% endfor %}
    </select>

    <div class="quantity-selector">
      <label for="quantity">{{ 'products.product.quantity' | t }}</label>
      <input type="number" id="quantity" name="quantity" value="1" min="1">
    </div>

    <button type="submit" name="add" {% unless product.available %}disabled{% endunless %}>
      {{ 'products.product.add_to_cart' | t }}
    </button>
  {% endform %}
</div>

<div class="contact-section">
  <h2>{{ 'contact.form.title' | t }}</h2>
  {%- comment -%}
    Contact form: Shopify injects authenticity_token, action="/contact",
    method="post". endform is required for CSRF token injection.
  {%- endcomment -%}
  {% form 'contact' %}
    {% if form.posted_successfully? %}
      <p class="form-success">{{ 'contact.form.post_success' | t }}</p>
    {% else %}
      <label for="contact-email">{{ 'contact.form.email' | t }}</label>
      <input
        type="email"
        id="contact-email"
        name="contact[email]"
        value="{{ form.email }}"
        required
        autocomplete="email"
      >

      <label for="contact-message">{{ 'contact.form.message' | t }}</label>
      <textarea id="contact-message" name="contact[body]" required></textarea>

      <button type="submit">{{ 'contact.form.send' | t }}</button>
    {% endif %}
  {% endform %}
</div>
```

---

## References

- [CWE-352 — Cross-Site Request Forgery](https://cwe.mitre.org/data/definitions/352.html)
- [OWASP A01:2021 — Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [Shopify Liquid — `form` tag](https://shopify.dev/docs/api/liquid/tags/form)
- [Shopify — Form types and authenticity tokens](https://shopify.dev/docs/api/liquid/tags/form#form-form_type)
- [OWASP — CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
