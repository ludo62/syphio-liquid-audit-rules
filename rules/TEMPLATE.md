# Rule ID: [SYP-XXX] - [Short Title]

## 📝 Description

Provide a clear explanation of what this rule detects and why it is an anti-pattern.

## 🔴 Impact

- **Category:** [Performance | Security | EAA 2025 | SEO]
- **Severity:** [Critical | High | Medium | Low]
- **Business Risk:** Explain how this affects conversion, speed, or legal status.

## 🔍 AST Pattern

Describe the AST nodes involved (e.g., `LiquidTag[name="include"]`).

## ❌ Faulty Code

```liquid
{% comment %} Example of what triggers the warning {% endcomment %}
```

## ✅ Optimized Code

```liquid
{% comment %} How the code should look after fix {% endcomment %}
```

## 📚 References

- [Link to Shopify Dev Docs](https://shopify.dev/docs)
- [Link to Syphio Methodology](../docs/performance.md)
