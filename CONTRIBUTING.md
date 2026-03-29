# Contributing to Syphio Rules

Thank you for interest in improving the Shopify Liquid audit ecosystem. To maintain "Senior Engineer" standards, all contributions must follow these guidelines.

## 🧪 Rule Submission Process

1. **Use the Template:** Every new rule must be based on [`rules/TEMPLATE.md`](./rules/TEMPLATE.md).
2. **AST-First Logic:** We do not accept simple Regex rules. You must describe the target **AST Node** (e.g., `LiquidTag`, `HtmlElement`).
3. **Business Impact:** Every rule must explain _why_ a merchant should care (Conversion, Security, or Law).

## 📂 Structure

- Accessibility rules → `/rules/accessibility/`
- Checkout rules → `/rules/checkout/`
- CI/CD rules → `/rules/ci-cd/`
- GDPR rules → `/rules/gdpr/`
- Hydrogen rules → `/rules/hydrogen/`
- Integrations rules → `/rules/integrations/`
- JavaScript rules → `/rules/javascript/`
- Liquid rules → `/rules/liquid/`
- Markets rules → `/rules/markets/`
- Metafields rules → `/rules/metafields/`
- Performance rules → `/rules/performance/`
- Schema rules → `/rules/schema/`
- Security rules → `/rules/security/`
- SEO rules → `/rules/seo/`

## 🛠️ Definition of Done

- [ ] Rule follows the standard Markdown format.
- [ ] Faulty and Optimized code examples are provided.
- [ ] Business Impact Score (PIS) is justified.
