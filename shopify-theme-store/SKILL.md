---
name: shopify-theme-store
description: Use when auditing a theme for Shopify Theme Store submission or reviewing whether changes break Theme Store eligibility. Covers required templates, performance thresholds (Lighthouse 60+/90+), browser/device matrix, accelerated checkout, gift cards, search, sections-everywhere, and the no-app-dependency / unique-design rules.
---

# Shopify Theme Store Requirements

## When to invoke

- Auditing a theme before initial Theme Store submission.
- Reviewing whether a feature change risks losing Theme Store eligibility.
- Building a checklist for QA before a major release.
- Answering "is this allowed for a Theme Store theme?" questions.

## Hard thresholds

| Area | Threshold |
|---|---|
| Lighthouse performance | ≥ 60 on product, collection, home (mobile + desktop) |
| Lighthouse accessibility | ≥ 90 on product, collection, home (mobile + desktop) |
| Color contrast (body text) | ≥ 4.5:1 |
| Color contrast (large text 18pt+) | ≥ 3:1 |

If a change drops Lighthouse performance below 60 on any of those three pages, the submission will be rejected. Run Lighthouse CI before PR.

## Required templates

Every Theme Store theme MUST include:

- `templates/index.json`
- `templates/product.json` (default)
- `templates/collection.json` (default)
- `templates/list-collections.json`
- `templates/blog.json`
- `templates/article.json`
- `templates/page.json`
- `templates/cart.json`
- `templates/search.json`
- `templates/404.json`
- `templates/password.json` (paired with `layout/password.liquid`)
- `templates/gift_card.liquid` (LIQUID, not JSON)
- Customer area: `templates/customers/login.liquid`, `register.liquid`, `account.liquid`, `order.liquid`, `addresses.liquid`, `activate_account.liquid`, `reset_password.liquid`

## Required features

### Sections Everywhere (OS 2.0)

Every JSON template must allow Custom Liquid blocks (`{ "type": "@app" }` and section blocks system). Header and footer MUST be section groups (`sections/header-group.json`, `sections/footer-group.json`), not hardcoded.

### Accelerated checkout buttons

Cart page must display the Shop Pay / Apple Pay / Google Pay button group via:

```liquid
{{ form | payment_button }}
```

### Gift cards

Must support purchasing AND redeeming gift cards. `templates/gift_card.liquid` must display the card image, code, expiration, and download link.

### Faceted search filtering

Collection and search pages must support faceted filters using the `filter` Liquid object. Filters must be keyboard navigable.

### Predictive search

Header search modal must use predictive-search Liquid endpoint (`/search/suggest.json` or `<predictive-search>` custom element).

### Product recommendations

Product page must include related-products section using `recommendations` Liquid object.

### Rich media

Product galleries must support image, video, 3D model, and external video (YouTube, Vimeo).

### Multi-language and multi-currency

- All visible text uses translation keys.
- Currency selector if store supports multiple markets.
- Locale selector if store supports multiple languages.

### Newsletter signup

Must include a newsletter form using `{% form 'customer' %}`.

### Shop Pay features

- Shop Pay accelerated checkout button.
- Login with Shop on customer login page.
- Shop avatar in header (`shop-user-avatar` element).
- Shop Pay Installments display on product price.

## Browser / device support

### Desktop

- Safari — latest 2 versions
- Chrome — latest 3 versions
- Firefox — latest 3 versions
- Edge — latest 2 versions

### Mobile

- iOS Safari — latest
- Chrome Mobile (Android) — latest
- Samsung Internet — latest

### Webviews

- Instagram in-app browser
- Facebook in-app browser
- Pinterest in-app browser

Avoid features that fail in older Safari (e.g., `flex gap` requires the Safari fallback in `assets/base.css`).

## Code / architecture rules

- Native CSS only — no Sass / SCSS source files.
- No minified first-party CSS / JS (third-party vendor libs exempt).
- Protocol-relative URLs for external CDN references (`//cdn.example.com/...`).
- No app-dependent features — theme functions without any installed app.
- No mock data, no "fake" customer / order info in templates.
- Third-party libraries must include LICENSE files.

## Settings / customization

- Setting labels in clear, merchant-friendly language.
- Settings grouped logically with header separators.
- At least 4 color schemes, each with matching foreground / background.
- Every font picker must support bold, italic, and bold-italic variants in the chosen family.

## Design uniqueness

- Cannot reuse another Theme Store theme's design (cosmetic-only changes rejected).
- Must demonstrate architectural innovation — new section types, new layout systems, distinct interaction patterns.
- Designs must hold up across varying content lengths and as new sections / settings are added.

## Demo store requirement

Each preset must have at least one demo store using authentic content:

- No Lorem Ipsum
- No stock photos that look generic
- Real-looking product names, prices, descriptions
- Realistic blog posts and images

## Documentation / support

- FAQ section in the theme documentation site.
- Functional support contact form.
- Support response within 2 business days.
- Public changelog of theme updates.

## Deceptive code — Partner Program compliance

These rules are compliance requirements under the Shopify Partner Program Agreement and API Terms of Service. Violations can result in removal from the Theme Store.

### No code obfuscation

Do not obfuscate theme code. Obfuscation transforms readable code into deliberately hard-to-parse output. There is no legitimate reason to obfuscate a theme — it hides behavior from Shopify review, harms performance, and is explicitly prohibited.

Standard minification (removing whitespace and comments) is separate and fine for third-party libs. First-party theme code must be readable.

### No search engine cloaking

Do not include code that presents different content to search engine crawlers than to human visitors. Cloaking is prohibited even when the intent is to improve Lighthouse or PageSpeed scores — manipulating what Google sees can result in removal from the Theme Store and Google penalties for the merchant's store.

### No search engine manipulation

Do not include code that manipulates search rankings through deceptive techniques — hidden keyword stuffing, invisible links, fake redirects, or other black-hat SEO tactics.

### General compliance principle

Act with integrity and in the best interests of the merchants using the theme. Regularly review compliance with the [Partner Program Agreement](https://www.shopify.com/partners/terms) and the [API Terms of Service](https://www.shopify.com/legal/api-terms).

## Submission checklist (run before submitting)

1. Lighthouse mobile + desktop ≥ 60 (performance) and ≥ 90 (a11y) on home, product, collection.
2. All required templates present.
3. All sections support `@theme` and `@app` blocks.
4. Header and footer are section groups.
5. Gift card purchase + redeem flow works.
6. Accelerated checkout buttons render on cart.
7. Predictive search works.
8. Product recommendations render.
9. Newsletter form submits.
10. No minified / SCSS files in `assets/`.
11. All visible strings are translation keys (no `t:` raw output, no hardcoded English).
12. At least 4 color schemes.
13. Tested in Safari, Chrome, Firefox, Edge, iOS Safari, Android Chrome.
14. Theme works with all apps uninstalled.
15. Demo store uses authentic content, no placeholders.
16. No obfuscated code, no cloaking, no SEO manipulation.

## References

- https://shopify.dev/docs/storefronts/themes/store/requirements
- https://shopify.dev/docs/storefronts/themes/store/review-process/submit-theme
- https://shopify.dev/docs/storefronts/themes/best-practices/performance
- https://shopify.dev/docs/storefronts/themes/best-practices/accessibility
