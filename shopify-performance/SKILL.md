---
name: shopify-performance
description: Use when optimizing a theme for Lighthouse, Core Web Vitals, or page-load speed. Covers image lazy loading, font-display strategies, defer vs async scripts, Liquid loops cost, the global section.css/section.js bundle size, image_tag responsive widths, and predictive search load patterns.
---

# Shopify Performance

## When to invoke

- Lighthouse performance score < 60 on product / collection / home.
- Slow LCP / FID / CLS reported in Web Vitals or Google Search Console.
- Investigating a slow page after a feature was added.
- Auditing CSS / JS bundle sizes before submission.

## Theme Store performance threshold

Lighthouse performance ≥ 60 (mobile + desktop) on:

- Home page
- Product page (a typical product)
- Collection page (a typical collection)

Tested with the Shopify Theme Inspector and `lighthouse-ci`.

## Core Web Vitals targets

| Metric | Good | Needs improvement | Poor |
|---|---|---|---|
| LCP (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## Images (the #1 LCP factor)

### Use `image_tag` with responsive `widths` and `sizes`

```liquid
{{ product.featured_image | image_tag:
   widths: '375,550,750,1100,1500',
   sizes: '(min-width: 1200px) 50vw, (min-width: 750px) 70vw, 100vw',
   loading: 'lazy',
   alt: product.title | escape
}}
```

`widths` generates a `srcset`. `sizes` tells the browser which width to pick for the current viewport.

### Lazy-load below the fold

```liquid
{{ image | image_tag: loading: 'lazy', fetchpriority: 'low' }}
```

`loading="eager"` only for the LCP candidate (typically the hero / first product image above the fold).

### Preload the LCP image

```liquid
{%- if section.index == 1 -%}
  <link rel="preload" as="image" href="{{ image | image_url: width: 1100 }}" imagesrcset="…" imagesizes="…">
{%- endif -%}
```

### Use modern formats automatically

`image_url` automatically serves WebP / AVIF when the browser supports it. No special config needed.

## Fonts

Use Shopify's font system for theme fonts:

```liquid
{{ settings.font_body | font_face: font_display: 'swap' }}
{{ settings.font_heading | font_face: font_display: 'swap' }}

<link rel="preconnect" href="https://fonts.shopifycdn.com">
```

`font-display: swap` shows fallback text immediately while custom font loads (avoids FOIT).

Preload only the most critical font weight:

```liquid
<link rel="preload" as="font" href="{{ settings.font_body | font_url }}" type="font/woff2" crossorigin>
```

## CSS

### Inline critical CSS in the layout `<head>`

Cleanest Shopify pattern: keep `base.css` small (variables + above-the-fold reset), load it from `layout/theme.liquid` `<head>` with normal `<link>` (no defer for CSS — render-blocking is required to avoid FOUC).

Lazy-load template-specific CSS conditionally:

```liquid
{%- if template contains 'product' -%}
  {{ 'template-product.css' | asset_url | stylesheet_tag }}
{%- endif -%}
```

### Section `{% stylesheet %}` global bundle

ALL `{% stylesheet %}` content across sections / blocks / snippets is bundled into one `section.css` file loaded everywhere. Bigger bundle → slower TTFB. Keep section stylesheets minimal and scope rules to avoid duplicates.

Audit:

```bash
# Estimated total size of all {% stylesheet %} content
grep -h -A 999 '{%- stylesheet -%}' sections/*.liquid snippets/*.liquid blocks/*.liquid 2>/dev/null \
  | sed '/{%- endstylesheet -%}/,$d' | wc -c
```

If > 50 KB, consider moving cross-cutting rules to `base.css` instead.

## JavaScript

### Defer non-critical JS

```liquid
<script src="{{ 'global.js' | asset_url }}" defer="defer"></script>
```

Never use `async` for theme JS that has ordering dependencies. `defer` preserves document order.

### Don't ship inline `<script>` blocks

`<script>` (no `src`) blocks the parser. Use `{% javascript %}` (auto-deferred section bundle) for section-scoped JS.

### Tree-shake third-party libraries

If you only use Swiper's core, don't bundle the full Swiper-element-bundle. Build a smaller version (third-party libs are exempt from the no-minified rule).

### Loading order matters

Shopify defer execution order matches document order. Put framework / library scripts before component scripts:

```liquid
<script src="{{ 'pubsub.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'global.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'swiper-element-bundle.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'animations.js' | asset_url }}" defer="defer"></script>
```

## Liquid performance

### Avoid heavy work inside `for` loops

```liquid
{# BAD — hits the DB for every iteration #}
{% for product in collection.products %}
  {% assign related = product.metafields.custom.related.value %}
{% endfor %}

{# GOOD — fetch once, iterate #}
{% liquid
  assign all_related = collection.products | map: 'metafields.custom.related.value'
%}
```

### Limit collection / product iterations

```liquid
{% for product in collection.products limit: 12 %}
```

The Shopify Theme Inspector for Chrome shows Liquid render times per template — use it to find the slowest loops.

### Cache repeated lookups in variables

```liquid
{%- liquid
  assign current_variant = product.selected_or_first_available_variant
  assign price_class = product.price_varies ? 'price--varies' : 'price--fixed'
-%}
```

## CLS prevention

- All images have explicit `width` and `height` attributes (NOT just CSS).
- Reserve space for above-the-fold lazy content via `min-height` or `aspect-ratio`.
- Avoid injecting content above existing content after page load.

```liquid
{{ image | image_tag: widths: '...', sizes: '...', loading: 'lazy', width: image.width, height: image.height }}
```

## Predictive search

If you use `<predictive-search>`, the request fires on every keystroke. Use:

- `debounce` of 250–300ms in JS.
- Cache last-result keyword to avoid redundant fetches.

```js
let lastQuery = '';
input.addEventListener('input', debounce((e) => {
  if (e.target.value === lastQuery) return;
  lastQuery = e.target.value;
  fetch(`/search/suggest.json?q=${e.target.value}`);
}, 300));
```

## Tools

- **Shopify Theme Inspector for Chrome** — Liquid render profiling per section.
- **Lighthouse / PageSpeed Insights** — global page audit.
- **Web Vitals Chrome extension** — live CWV metrics during dev.
- **Chrome Performance tab** — flame chart for INP / long tasks.
- **`lighthouse-ci`** in CI — automate Lighthouse runs on PR.

## Common quick wins

1. Reduce image sizes (most themes load 2 MB+ of hero images).
2. Defer all non-critical JS.
3. Inline LCP image preload.
4. Remove unused fonts / font weights.
5. Cut down on the `section.css` global bundle (scope, deduplicate, move to base.css).
6. Use `font-display: swap`.
7. Set `loading="lazy"` on below-the-fold images.
8. Replace external embed scripts with native HTML where possible.

## References

- https://shopify.dev/docs/storefronts/themes/best-practices/performance
- https://shopify.dev/docs/storefronts/themes/best-practices/performance
- https://shopify.dev/docs/storefronts/themes/tools/theme-inspector
- https://web.dev/vitals/
- https://shopify.dev/docs/api/liquid/filters/image_tag
