# 🛡️ Syphio | Liquid Audit Standard & Rule Catalog

**Syphio** is a comprehensive, AST-based rule catalog designed to define the highest standards for Shopify Liquid development. This repository contains the formal specifications for security, performance, and accessibility (EAA 2025) audits.

> **Status:** This repository serves as the official documentation for the Syphio Audit Logic. It is intended for developers, architects, and security auditors looking to implement or understand advanced Shopify Theme analysis.

---

## 💎 The AST Advantage

Unlike legacy linters that rely on brittle Regex patterns, the Syphio standard is built on **Abstract Syntax Tree (AST) analysis**.

| Feature                  | Pattern Matching (Regex) |          Syphio (AST Logic)          |
| :----------------------- | :----------------------: | :----------------------------------: |
| **Logic Awareness**      |          ❌ No           |               ✅ Full                |
| **Scope Resolution**     |          ❌ No           | ✅ Yes (Detects origin of variables) |
| **Contextual Integrity** | ❌ Fails on nested tags  |        ✅ Maintains hierarchy        |
| **Accuracy**             | ⬆️ High False Positives  |             ⬇️ Near-zero             |

---

## 📚 Rule Catalog (Phase 1: Core Liquid)

This catalog is divided into specialized domains. Each rule includes a formal `Detection Logic`, `Anti-Pattern` examples, and `Optimized Fixes`.

### 🔹 [LIQ] Liquid: Syntax & Deprecated Tags

| ID          | Rule Name                         | Severity | Primary Impact     |
| :---------- | :-------------------------------- | :------: | :----------------- |
| **LIQ-019** | `\| escape` in `<script>` blocks  | 🔴 CRIT  | Security (XSS)     |
| **LIQ-022** | Deprecated `.css.liquid` Assets   | 🔴 CRIT  | Core Rendering     |
| **LIQ-023** | Missing Assets via `asset_url`    | 🟠 HIGH  | Performance (404s) |
| **LIQ-025** | `selected_variant` Fallback Logic | 🟠 HIGH  | SEO & Conversion   |

### 🔹 [ACC] Accessibility (EAA 2025 Compliance)

| ID          | Rule Name                              | Severity | Requirement           |
| :---------- | :------------------------------------- | :------: | :-------------------- |
| **ACC-001** | Missing `alt` on static/dynamic images | 🔴 CRIT  | WCAG 2.2 / EAA        |
| **ACC-002** | Unlinked Form Control Labels           | 🟠 HIGH  | Screen Reader Support |

---

## 📖 How to use this Catalog

Each rule file in the `/rules` directory follows a strict format:

1. **Context:** Why the rule exists and its business/technical impact.
2. **Detection Logic:** The technical steps required for an AST engine to identify the violation.
3. **Anti-Pattern:** Real-world examples of failing code.
4. **Optimized Fix:** The compliant, modern Shopify OS 2.0 solution.

---

## 🤝 Reference & Authorship

Syphio is maintained by [Ludo62](https://github.com/ludo62). This specification is part of a broader effort to standardize high-end Shopify Theme engineering.

---

© 2026 Syphio Audit Specifications.
