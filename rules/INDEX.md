# Syphio Rules Index (354)

> **The definitive database of Shopify Liquid anti-patterns.** > Rules are categorized by their primary business impact. Each rule is mapped to an AST pattern for automated detection.

---

## ⚡ Performance (100+ Rules)

Focus: LCP optimization, Liquid heap memory, and server-side rendering (TTFB).

| ID          | Rule Title                                                                | Severity  | Status |
| :---------- | :------------------------------------------------------------------------ | :-------: | :----: |
| **LIQ-001** | [Deprecated `include` usage (Cache Decoupling)](./performance/LIQ-001.md) |  🟠 High  |   ✅   |
| **LIQ-002** | [`img_url` to `image_url` migration](./performance/LIQ-002.md)            |  🟠 High  |   ✅   |
| **LIQ-005** | [Invariant `assign` in loops (Memory Churn)](./performance/LIQ-005.md)    | 🟡 Medium |   ✅   |
| _..._       | _[Remaining Performance Rules]_                                           |     -     |   ⏳   |

---

## 🔒 Security (80+ Rules)

Focus: XSS vectors, CSRF exposure, and data leakage in Liquid objects.

| ID          | Rule Title                                                        |  Severity   | Status |
| :---------- | :---------------------------------------------------------------- | :---------: | :----: |
| **SYP-201** | [Unsafe XSS in `<script>` context](./security/SYP-201.md)         | 🔴 Critical |   ✅   |
| **SYP-202** | [Leaking Customer data via global objects](./security/SYP-202.md) | 🔴 Critical |   ⏳   |
| _..._       | _[Remaining Security Rules]_                                      |      -      |   ⏳   |

---

## ♿ EAA 2025 & Accessibility (90+ Rules)

Focus: WCAG 2.1 AA compliance and mandatory EU Accessibility Law.

| ID          | Rule Title                                                        | Severity | Status |
| :---------- | :---------------------------------------------------------------- | :------: | :----: |
| **ACC-101** | [Missing ARIA labels in Search Forms](./accessibility/ACC-101.md) | 🟠 High  |   ⏳   |
| **ACC-102** | [Hardcoded focus traps in Drawers](./accessibility/ACC-102.md)    | 🟠 High  |   ⏳   |
| _..._       | _[Remaining Compliance Rules]_                                    |    -     |   ⏳   |

---

## 🔍 Technical SEO (84 Rules)

Focus: JSON-LD integrity, Hreflang logic, and Semantic HTML.

| ID          | Rule Title                                                  |  Severity   | Status |
| :---------- | :---------------------------------------------------------- | :---------: | :----: |
| **SEO-301** | [Malformed JSON-LD in `product.liquid`](./seo/SEO-301.md)   |   🟠 High   |   ⏳   |
| **SEO-302** | [Broken Hreflang logic in `theme.liquid`](./seo/SEO-302.md) | 🔴 Critical |   ⏳   |
| _..._       | _[Remaining SEO Rules]_                                     |      -      |   ⏳   |

---

## 🧪 Experimental / Merchant Logic

Focus: Broken conditions and silent failures.

| ID          | Rule Title                                                  | Severity  | Status |
| :---------- | :---------------------------------------------------------- | :-------: | :----: |
| **LOG-401** | [Empty `else` blocks in critical logic](./logic/LOG-401.md) | 🟡 Medium |   ⏳   |

---

[← Back to Home](../README.md)
