# Methodology: Liquid Performance & Rendering Efficiency

Performance auditing in Syphio goes beyond Lighthouse scores. We analyze how Liquid code interacts with Shopify's server-side rendering (SSR) engine.

## 🧠 Core Concepts

### 1. The Liquid Heap Memory

Every `{% assign %}` or `{% capture %}` consumes memory on Shopify's servers.

- **Anti-pattern:** Re-assigning large objects (like `all_products`) inside a `{% for %}` loop.
- **Impact:** Increases **TTFB (Time to First Byte)** because the server struggles to allocate memory before sending HTML.

### 2. N+1 Query Problem (Nested Loops)

When a loop is nested inside another loop, complexity grows exponentially ($O(n^2)$).

- **Syphio Detection:** We look for `all_products` or `collections` lookups inside product loops.
- **Business Impact:** A 500ms delay in SSR can lead to a **7% drop in conversion**.

## 📊 Performance Scoring Logic

Rules in this category affect the **LCP (Largest Contentful Paint)**.

- **Critical:** Blocking scripts or unoptimized images (`img_url`).
- **High:** Excessive global object lookups.
