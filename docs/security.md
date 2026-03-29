# Methodology: Shopify Liquid Security & Data Integrity

Security in Shopify themes is often overlooked because Liquid is "server-side". However, XSS (Cross-Site Scripting) is the #1 vulnerability in themes.

## 🛡️ Attack Vectors

### 1. XSS in Script Contexts

The most common breach. Using `{{ product.title }}` inside a `<script>` tag without the `| json` filter allows an attacker to inject malicious JS via a product title.

- **Syphio AST Check:** `HtmlElement[name="script"]` → `LiquidVariable` (check for `json` or `escape` filters).

### 2. Customer Data Leakage

Accidentally exposing the `customer` object or internal `metafields` in the frontend source code.

- **Impact:** GDPR non-compliance and potential data theft.

## ⚖️ Severity Levels

- **🔴 Critical:** Remote Script Injection or Session Hijacking.
- **🟠 High:** Unintentional data exposure via Metafields.
