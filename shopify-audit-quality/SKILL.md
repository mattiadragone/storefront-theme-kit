---
name: shopify-audit-quality
description: Use when auditing an existing Shopify theme repo for performance and accessibility quality. Runs Lighthouse score checks, JS bundle size, preload count, touch target sizes, focus indicators, and reduced-motion support. Does NOT build code — inspects existing files and reports findings.
---

# Shopify Quality Audit

Use this skill to inspect an existing theme for performance and accessibility quality issues. Run after the critical audit (`shopify-audit-critical`) or independently to investigate speed or a11y regressions.

## When to invoke

- Lighthouse performance score < 60 on product, collection, or home.
- Lighthouse accessibility score < 90 on any required page.
- After adding new sections, sliders, or media-heavy content.
- Before a Theme Store submission quality check.

## How to run

Execute each check block below against the repo root.

---

## CHECK 1 — Lighthouse score baseline

Run Lighthouse on the three required pages:

```bash
# Using Shopify's Lighthouse CI action (in CI) or locally:
npx lighthouse <store-url>/products/<handle> --output json --quiet | jq '.categories.performance.score, .categories.accessibility.score'
npx lighthouse <store-url>/collections/<handle> --output json --quiet | jq '.categories.performance.score, .categories.accessibility.score'
npx lighthouse <store-url> --output json --quiet | jq '.categories.performance.score, .categories.accessibility.score'
```

Weighted speed score formula:
```
speed_score = [(product × 31) + (collection × 33) + (home × 13)] / 77
```

Thresholds:
- Performance: ≥ 60 on all three pages (blocking for Theme Store)
- Accessibility: ≥ 90 on all three pages (blocking for Theme Store)

---

## CHECK 2 — JS bundle size

Minified JavaScript entry points must be ≤ 16 KB.

```bash
# Estimate minified sizes for main JS assets
for f in assets/global.js assets/theme.js assets/base.js; do
  [ -f "$f" ] && echo "$f: $(cat "$f" | wc -c) bytes unminified"
done

# Estimate minified size with esbuild (if installed)
esbuild assets/global.js --bundle --minify 2>/dev/null | wc -c
```

Severity: **HIGH** — large JS bundles increase parse time on low-end mobile, tanks INP.

---

## CHECK 3 — `{% stylesheet %}` bundle size

All section/block/snippet `{% stylesheet %}` content is concatenated into one global `section.css`. If the total exceeds 50 KB uncompressed, performance will degrade.

```bash
# Estimate total size of all {% stylesheet %} content
grep -rh "{% stylesheet %}" sections/ blocks/ snippets/ 2>/dev/null | wc -l
grep -rh "" sections/*.liquid blocks/*.liquid snippets/*.liquid 2>/dev/null \
  | awk '/\{% stylesheet %\}/{p=1} p{print} /\{% endstylesheet %\}/{p=0}' | wc -c
```

If > 50 KB: move cross-cutting rules to `assets/base.css` and remove duplicates.

Severity: **MEDIUM** — oversized bundle increases TTFB and parse cost.

---

## CHECK 4 — Preload count per template

Shopify recommends at most 2 `<link rel="preload">` hints per template. More than 2 compete for bandwidth.

```bash
# Count preload hints in layout and snippets
grep -rn 'rel="preload"\|preload: true' layout/ snippets/ sections/ 2>/dev/null | wc -l
grep -rn 'rel="preload"\|preload: true' layout/ snippets/ sections/ 2>/dev/null | head -20
```

If > 2 per template: keep only the LCP image and the critical CSS preload. Remove the rest.

Severity: **MEDIUM** — excess preloads compete for bandwidth and delay LCP.

---

## CHECK 5 — LCP image — eager loading and preload

The hero / first product image (LCP candidate) must be `loading="eager"` with a `fetchpriority="high"` preload in the layout `<head>`.

```bash
# Check for eager loading on first section's image
grep -n 'loading.*eager\|fetchpriority.*high\|rel="preload".*image' layout/theme.liquid snippets/*.liquid 2>/dev/null | head -10
```

Missing preload for LCP image = highest single source of poor LCP scores.

Severity: **HIGH** — LCP > 2.5s is the most common Theme Store performance failure.

---

## CHECK 6 — Touch target sizes

Primary interactive controls must be at least 44 × 44 px.

Manual check — look for these elements in layout and sections and verify CSS:

- Hamburger / menu toggle button
- Cart icon / cart drawer trigger
- Close buttons (modal, drawer, search)
- Navigation links in mobile menu
- Variant selector buttons / swatches
- Add-to-cart button
- Form submit buttons

```bash
# Find buttons / links that likely lack min-size CSS
grep -rn 'class=".*close\|class=".*hamburger\|class=".*cart.*icon\|class=".*nav.*toggle' sections/ layout/ snippets/ 2>/dev/null | head -20
```

Then search `assets/base.css` for `min-width: 44px` / `min-height: 44px` patterns to verify.

Severity: **HIGH** — controls smaller than 44 × 44 px fail Lighthouse tap-target check, reducing a11y score.

---

## CHECK 7 — Focus indicators

Every interactive element must have a visible `:focus-visible` outline with ≥ 3:1 contrast against its background.

```bash
# Check for focus-visible rule in base.css
grep -n "focus-visible\|:focus" assets/base.css 2>/dev/null

# Check for outline: none / outline: 0 without a replacement
grep -rn "outline:\s*none\|outline:\s*0" assets/ sections/ snippets/ 2>/dev/null | head -20
```

`outline: none` without a replacement `box-shadow` or border is a hard accessibility failure.

Severity: **HIGH** — missing focus indicators fail WCAG 2.1 AA and Lighthouse a11y.

---

## CHECK 8 — Reduced motion support

```bash
grep -rn "prefers-reduced-motion" assets/*.css sections/*.liquid snippets/*.liquid 2>/dev/null | head -10
```

The theme must have at least one `@media (prefers-reduced-motion: reduce)` rule that disables animations and transitions. Required by Theme Store.

Severity: **MEDIUM** — missing reduced-motion support fails Theme Store accessibility review.

---

## CHECK 9 — Font preload and `font-display: swap`

```bash
grep -n "font_face\|font-display\|font_url\|preload.*font" layout/theme.liquid snippets/*.liquid 2>/dev/null | head -10
```

All `font_face` calls should use `font_display: 'swap'` to prevent invisible text (FOIT) during load.

Severity: **MEDIUM** — missing `font-display: swap` causes FOIT, which contributes to poor LCP.

---

## CHECK 10 — `loading="lazy"` on below-the-fold images

```bash
# Find image_tag calls without lazy loading
grep -rn 'image_tag' sections/ snippets/ blocks/ 2>/dev/null \
  | grep -v "loading: 'lazy'\|loading: lazy" | head -20
```

All images below the fold should be lazy-loaded. Only the LCP candidate (first section, index == 1) should be `loading: 'eager'`.

Severity: **MEDIUM** — eager-loading below-the-fold images wastes bandwidth and delays LCP.

---

## Audit summary template

```
### Quality Audit Results — <theme name> — <date>

CHECK 1 Lighthouse scores:   product=<X> collection=<X> home=<X> | speed_score=<calc>
CHECK 2 JS bundle size:      <X KB> — PASS/FAIL (limit: 16 KB)
CHECK 3 stylesheet bundle:   <X KB> — PASS/FAIL (warn: 50 KB)
CHECK 4 Preload count:       <N> — PASS/FAIL (limit: 2 per template)
CHECK 5 LCP preload:         PRESENT / MISSING
CHECK 6 Touch targets:       PASS / FAIL — <elements to fix>
CHECK 7 Focus indicators:    PASS / FAIL — <outline: none violations>
CHECK 8 Reduced motion:      PRESENT / MISSING
CHECK 9 Font display swap:   PRESENT / MISSING
CHECK 10 Lazy load images:   PASS / FAIL — <N eager images below fold>

Priority fixes: <list ordered by Lighthouse impact>
```

## References

- https://shopify.dev/docs/storefronts/themes/best-practices/performance
- https://shopify.dev/docs/storefronts/themes/best-practices/accessibility
- https://web.dev/vitals/
- https://github.com/Shopify/lighthouse-ci-action
