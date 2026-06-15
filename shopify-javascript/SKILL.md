---
name: shopify-javascript
description: Use when writing or modifying JS in assets/*.js or inside {% javascript %} blocks. Covers Custom Elements pattern (web components), the {% javascript %} per-section bundle, Shopify section editor events (shopify:section:load), defer loading order, pubsub pattern, and the no-minified-output Theme Store rule.
---

# Shopify JavaScript

## When to invoke

- Writing component JS as a Custom Element (web component) or vanilla.
- Adding behavior to a section via the inline `{% javascript %}` block.
- Listening to Shopify theme editor events (`shopify:section:load`, `shopify:block:select`, etc.).
- Debugging script load order or "swiper.swiper is undefined" race conditions.

## Theme Store JS rules

- **No minified first-party JS** in `assets/`. Reviewers want readable source.
- **Third-party vendor bundles** (Swiper, GSAP, etc.) ARE allowed minified.
- **No transpilation source maps** in production (`*.map`).
- **No app-dependent features** — the theme must function without any installed app.

## The `{% javascript %}` per-section block

Like `{% stylesheet %}`, the `{% javascript %}` block content is auto-bundled by Shopify into a `section.js` file loaded on every page. It runs ONCE per page load, not per section render.

```liquid
{% javascript %}
  class MySection extends HTMLElement {
    connectedCallback() {
      this.button = this.querySelector('button');
      this.button.addEventListener('click', this.onClick.bind(this));
    }
    onClick() { console.log('clicked'); }
  }
  customElements.define('my-section', MySection);
{% endjavascript %}
```

Pitfall: code inside `{% javascript %}` runs once. If multiple instances of the section exist on a page, each must initialize itself (via Custom Element `connectedCallback`, NOT via `document.querySelector('my-section').init()`).

## Custom Elements pattern (recommended)

Wrap each section's behavior in a Custom Element. Lifecycle:

```js
class ProductCard extends HTMLElement {
  constructor() { super(); }
  connectedCallback() {
    // Set up event listeners, observers, fetch initial state
    this.addToCartButton = this.querySelector('[data-add-to-cart]');
    this.addToCartButton?.addEventListener('click', this.onAdd.bind(this));
  }
  disconnectedCallback() {
    // Tear down listeners, intervals, observers
  }
  onAdd(event) {
    event.preventDefault();
    // ...
  }
}
customElements.define('product-card', ProductCard);
```

In the section markup:
```liquid
<product-card data-product-id="{{ product.id }}">
  <button data-add-to-cart>Add</button>
</product-card>
```

Benefits: each instance initializes itself; teardown on removal is automatic; works seamlessly with theme editor reloads.

## Shopify theme editor events

In the theme editor, sections / blocks are added/removed/selected without a full page reload. JS must respond to these events:

```js
document.addEventListener('shopify:section:load', (event) => {
  // event.detail.sectionId — the id of the section that just loaded
  // event.target — the section's DOM element
});

document.addEventListener('shopify:section:unload', (event) => { /* cleanup */ });
document.addEventListener('shopify:section:select', (event) => { /* focus / scroll */ });
document.addEventListener('shopify:section:deselect', (event) => { /* unfocus */ });
document.addEventListener('shopify:section:reorder', (event) => { /* re-init position-dependent code */ });

document.addEventListener('shopify:block:select', (event) => { /* highlight block */ });
document.addEventListener('shopify:block:deselect', (event) => { /* unhighlight */ });
```

If you use Custom Elements, most of this is handled automatically by `connectedCallback` / `disconnectedCallback` when the editor reinserts the section DOM.

## Defer loading order

Scripts loaded with `defer` execute in document order AFTER the HTML is parsed, BEFORE `DOMContentLoaded`.

```liquid
<!-- in layout/theme.liquid <head> -->
<script src="{{ 'constants.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'pubsub.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'global.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'swiper-element-bundle.js' | asset_url }}" defer="defer"></script>
<script src="{{ 'animations.js' | asset_url }}" defer="defer"></script>
```

Race condition pitfall: if `section.js` (bundled from `{% javascript %}` blocks) uses `swiperEl.swiper.…`, you depend on `swiper-element-bundle.js` having initialized the element first. Both are deferred → both execute before `DOMContentLoaded`, so wrapping section JS in:

```js
document.addEventListener('DOMContentLoaded', () => { /* … */ });
```

guarantees the bundle has registered `<swiper-container>` Custom Element by then.

## PubSub pattern for cross-component communication

A common theme pattern:

```js
// assets/pubsub.js
const subscribers = {};
export function subscribe(eventName, callback) {
  subscribers[eventName] ??= [];
  subscribers[eventName].push(callback);
  return () => { subscribers[eventName] = subscribers[eventName].filter(cb => cb !== callback); };
}
export function publish(eventName, data) {
  (subscribers[eventName] || []).forEach(cb => cb(data));
}
window.PubSub = { subscribe, publish };

// In section A:
PubSub.publish('cart:updated', { itemCount: 3 });

// In section B:
PubSub.subscribe('cart:updated', (data) => updateBubble(data.itemCount));
```

Pattern is used heavily in Dawn / Horizon for cart drawer ↔ cart icon updates.

## Fetch / async patterns

Use `fetch` with proper headers for the Shopify Cart API:

```js
async function addToCart(variantId, quantity) {
  const res = await fetch(`${window.Shopify.routes.root}cart/add.js`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
    body: JSON.stringify({ items: [{ id: variantId, quantity }] })
  });
  if (!res.ok) throw new Error(`Cart add failed: ${res.status}`);
  return res.json();
}
```

Always use `window.Shopify.routes.root` to support multi-language stores (prefixes like `/en`, `/fr`).

## Accessibility hooks

JavaScript that opens / closes modals / drawers must:

- Focus the first interactive element on open.
- Trap focus inside while open.
- Restore focus to the trigger on close.
- Respond to `Escape` key.
- Add `aria-expanded` to the trigger button.

See the `shopify-accessibility` skill for detailed component patterns.

## CRITICAL pitfalls

- **No `eval`, no `new Function(...)`** — Shopify CSP blocks them.
- **No global `var` in section JS** — `var foo = …` at the top of a `{% javascript %}` block leaks onto `window`. Use IIFE / Custom Element class scope.
- **No `document.write`** — Shopify storefronts block it.
- **No untrusted innerHTML** — escape user input with `escape` Liquid filter on the server, or use `textContent` on the client.

## References

- https://shopify.dev/docs/storefronts/themes/architecture/sections/section-assets
- https://shopify.dev/docs/storefronts/themes/architecture/sections/section-schema#javascript-tag
- https://shopify.dev/docs/storefronts/themes/best-practices/performance
- https://shopify.dev/docs/api/ajax
- https://developer.mozilla.org/docs/Web/Web_Components
