# Syphio AST Node Specification (v1.0)

This document defines the Abstract Syntax Tree (AST) nodes used by the Syphio engine to analyze Shopify Liquid source code.

## 🌳 Node Definitions

### 1. `LiquidTag`

Any logic delimited by `{% ... %}`.

- **`name`**: The tag identifier (e.g., `if`, `for`, `assign`, `render`).
- **`markup`**: The raw content inside the tag.
- **`is_block`**: Boolean. True if it requires an `end` tag.
- **`children`**: Array of nodes contained within a block tag.

### 2. `LiquidVariable`

Any output delimited by `{{ ... }}`.

- **`expression`**: The object being accessed (e.g., `product.title`).
- **`filters`**: Array of `Filter` nodes applied to the variable.

### 3. `HtmlElement`

Standard HTML tags parsed in context with Liquid.

- **`tag_name`**: The HTML tag (e.g., `script`, `img`, `div`).
- **`attributes`**: Key-value pairs of HTML attributes.
- **`context`**: The rendering context (e.g., `head`, `body`, `json-ld`).

---

_Note: This schema ensures that all 354 rules follow a unified structural logic._
