---
name: shopify-locales
description: Use when adding or modifying translations in locales/*.json. Covers the four file types (default, storefront, schema, default.schema), the t: prefix in schemas, the difference between merchant-facing and storefront-facing strings, and the fallback chain.
---

# Shopify Locales

## When to invoke

- Adding a new visible string to a section, snippet, block, or template.
- Adding a label / info / preset name to a section schema (requires a key in `*.default.schema.json`).
- Translating an existing theme to a new language.
- Decoding a missing `t:` key error in the theme editor.

## The four file types

`locales/` contains four flavors of JSON files per language:

- **`<lang>.default.json`** — storefront-facing strings (cart, product, search, etc.) for the default theme language. One file marked default per theme. Example: `locales/en.default.json`.
- **`<lang>.json`** — storefront strings for an additional language. Example: `locales/it.json`, `locales/fr.json`.
- **`<lang>.default.schema.json`** — merchant-facing strings shown in the theme editor (setting labels, section names, info text). One default per theme.
- **`<lang>.schema.json`** — merchant-facing strings for an additional editor language.

Strict rule: storefront strings (`*.json`) and editor strings (`*.schema.json`) are SEPARATE. A `t:` key from `en.default.json` is NOT visible to the editor sidebar; a key from `en.default.schema.json` is NOT visible in storefront rendering.

## Using translation keys

### Storefront-facing (in Liquid templates)

```liquid
{{ 'cart.general.title' | t }}
{{ 'products.product.add_to_cart' | t }}
{{ 'sections.header.search' | t }}
```

The `t` filter looks up the key in the active language file (e.g., `en.default.json` for English).

With dynamic values:

```liquid
{{ 'cart.general.total_count' | t: count: cart.item_count }}
```

The locale file value uses `{{ count }}` interpolation:

```json
{
  "cart": {
    "general": {
      "total_count": "{{ count }} items in cart"
    }
  }
}
```

### Merchant-facing (in section / block schemas)

Use `t:` prefix INSIDE JSON values:

```json
{
  "name": "t:names.featured_collection",
  "settings": [
    {
      "type": "text",
      "id": "heading",
      "label": "t:settings.heading",
      "info": "t:settings.heading_info",
      "default": "Featured collection"
    }
  ],
  "presets": [
    { "name": "t:names.featured_collection" }
  ]
}
```

Each `t:` key MUST exist in `en.default.schema.json`:

```json
{
  "names": {
    "featured_collection": "Featured collection"
  },
  "settings": {
    "heading": "Heading",
    "heading_info": "Shown above the collection grid"
  }
}
```

If a key is missing, the editor displays the raw `t:` string — bad UX and Theme Store rejection.

## File structure

JSON nested by category. Common top-level keys in `en.default.json`:

```json
{
  "general": {
    "accessibility": { "..." },
    "search": { "..." },
    "social": { "..." }
  },
  "products": { "..." },
  "cart": { "..." },
  "templates": {
    "search": { "..." },
    "cart": { "..." }
  },
  "sections": {
    "header": { "..." },
    "footer": { "..." }
  },
  "customer": { "..." },
  "blogs": { "..." }
}
```

For schemas (`en.default.schema.json`), the common top level is:

```json
{
  "names": { },
  "settings": { },
  "settings_schema": { },
  "blocks": { }
}
```

## Pluralization

Use one / few / many / other plural buckets:

```json
{
  "cart": {
    "general": {
      "item_count": {
        "one": "{{ count }} item",
        "other": "{{ count }} items"
      }
    }
  }
}
```

In Liquid:

```liquid
{{ 'cart.general.item_count' | t: count: cart.item_count }}
```

## Locale fallback chain

Storefront resolution order:

1. Active language file (`<lang>.json`)
2. Default storefront language (`<lang>.default.json` for the default language)
3. English fallback (`en.default.json`)
4. Raw key string

If a key is missing in all languages, the storefront shows the raw `cart.general.title` text. Theme Store reviewers check for this and reject themes with raw keys visible.

## Required default keys (Theme Store)

A Theme Store-bound theme must include translations for:

- All storefront text (cart, product, search, account, etc.).
- Every section, block, and theme setting label.
- Accessibility strings (`general.accessibility.*`) for screen readers.

Run `theme-check` with the `MissingTemplate` and `TranslationKeyExists` checks enabled to catch missing keys.

## References

- https://shopify.dev/docs/storefronts/themes/architecture/locales
- https://shopify.dev/docs/api/liquid/filters/translate
- https://shopify.dev/docs/storefronts/themes/markets/multiple-currencies-languages
