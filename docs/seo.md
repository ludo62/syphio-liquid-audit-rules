# Methodology: Technical SEO & Semantic Integrity

Technical SEO in Shopify is about **Machine Readability**.

## 🤖 Semantic Precision

### 1. JSON-LD Integrity

Analyzing `product.liquid` to ensure Schema.org data is perfectly formed.

- **Syphio Check:** Ensuring `priceCurrency` and `availability` are pulled correctly from Liquid objects without syntax errors that break Google Search Console.

### 2. Hreflang & Internationalization

Ensuring that multi-market stores correctly map `request.path` in the `<head>` to prevent duplicate content penalties.
