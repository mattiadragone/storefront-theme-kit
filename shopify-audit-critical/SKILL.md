---
name: shopify-audit-critical
description: Use when auditing an existing Shopify theme repo for critical correctness errors. Runs severity-1 checks: CSS scoping leaks, missing t: prefixes, missing block.shopify_attributes, undeferred scripts, images without alt or dimensions, and theme-check violations. Does NOT build code — inspects existing files.
---

# Shopify Critical Audit

Use this skill to inspect an existing theme repo and surface blocking errors before a review or Theme Store submission. Run these checks before any code review or QA pass.

## When to invoke

- Before merging a feature branch to main in a Shopify theme repo.
- Before opening a Theme Store submission.
- After a major refactor to catch regressions.
- When a merchant reports visual or editor bugs.

## How to run

Execute each check block below against the repo root. All commands assume `bash` and that `sections/`, `blocks/`, `snippets/`, `assets/` exist in the current directory.

---

## CHECK 1 — CSS scoping leaks

Every selector inside `{% stylesheet %}` must be scoped. Unscoped selectors leak into the global `section.css` bundle loaded on every page.

```bash
# Extract all {% stylesheet %} content and find potentially unscoped top-level selectors
grep -rn "{% stylesheet %}" sections/ blocks/ snippets/ 2>/dev/null | while IFS=: read file line rest; do
  awk "NR==$line,/\{%[- ]* endstylesheet [-%]\}/" "$file"
done | grep -E "^\s*(html|body|\*|\.[-a-z_]+\s*\{|[a-z]+\[)" | head -30
```

Severity: **BLOCKING** — unscoped CSS causes cross-page visual regressions.

Manual rule: open each `{% stylesheet %}` block and confirm every CSS selector starts with the section/block root class or a custom-element tag name unique to that file.

---

## CHECK 2 — Missing `t:` prefixes in schema strings

Every `label`, `info`, `name`, and preset `name` in section/block schema JSON must use `t:` prefix.

```bash
# Find schema label/info/name fields with raw strings (not t: prefix)
grep -rn '"label"\|"info"\|"name"\|"content"' sections/ blocks/ config/ 2>/dev/null \
  | grep -v '"t:' \
  | grep -v '//\|#\|schema_name\|theme_name\|theme_author\|theme_documentation\|theme_support\|theme_version' \
  | grep -v '"default"\|"id"\|"type"\|"value"' \
  | head -40
```

Severity: **BLOCKING** — raw strings display in the editor instead of translated text; Theme Store rejects these.

---

## CHECK 3 — Missing `block.shopify_attributes`

Every block's root element must render `{{ block.shopify_attributes }}`. Without it the block is invisible in the editor.

```bash
# Find block files that lack block.shopify_attributes
for f in blocks/*.liquid; do
  grep -q "block.shopify_attributes" "$f" || echo "MISSING: $f"
done
```

Severity: **BLOCKING** — blocks are not selectable in the theme editor.

---

## CHECK 4 — Scripts without `defer`

All non-critical `<script src="...">` tags must have `defer="defer"`. Blocking scripts delay LCP and INP.

```bash
# Find script tags without defer
grep -rn '<script src=' layout/ sections/ snippets/ templates/ 2>/dev/null \
  | grep -v 'defer' \
  | grep -v '<!--' \
  | head -20
```

Severity: **HIGH** — parser-blocking scripts directly lower Lighthouse performance score.

---

## CHECK 5 — Images without `alt` attribute

All `<img>` tags and `image_tag` filter calls must include an `alt` attribute.

```bash
# Find img tags without alt
grep -rn '<img' sections/ snippets/ blocks/ templates/ 2>/dev/null \
  | grep -v 'alt=' | head -20

# Find image_tag calls without alt
grep -rn 'image_tag' sections/ snippets/ blocks/ templates/ 2>/dev/null \
  | grep -v 'alt:' | head -20
```

Severity: **HIGH** — missing alt drops Lighthouse a11y score below the 90 threshold required by Theme Store.

---

## CHECK 6 — Images without explicit `width` and `height`

Images need explicit `width` and `height` to prevent CLS (Cumulative Layout Shift).

```bash
# Find image_tag calls without explicit width/height dimensions
grep -rn 'image_tag' sections/ snippets/ blocks/ 2>/dev/null \
  | grep -v 'width:.*height:\|widths:' | head -20
```

Severity: **HIGH** — missing dimensions cause CLS > 0.1, a Core Web Vitals failure.

---

## CHECK 7 — `{% include %}` (deprecated)

`{% include %}` leaks parent scope and is deprecated. Replace with `{% render %}`.

```bash
grep -rn "{%[- ]* include " sections/ snippets/ blocks/ templates/ layout/ 2>/dev/null | head -20
```

Severity: **MEDIUM** — theme-check flags this; Theme Store may reject.

---

## CHECK 8 — `theme-check` (Shopify CLI)

Run Shopify's own static analysis tool:

```bash
shopify theme check
```

Or with specific severity:

```bash
shopify theme check --fail-level error
```

`theme-check` catches: missing translation keys, deprecated tags, missing schema, invalid setting types, and more. Fix all `error` severity items before submission.

Severity: **ALL** — `theme-check` errors are blocking; warnings should be reviewed.

---

## Audit summary template

After running all checks, report findings in this format:

```
### Critical Audit Results — <theme name> — <date>

CHECK 1 CSS scoping: PASS / FAIL — <N unscoped selectors found in: file.liquid>
CHECK 2 t: prefixes:  PASS / FAIL — <N raw strings in: file, file>
CHECK 3 block attrs:  PASS / FAIL — <files missing block.shopify_attributes>
CHECK 4 defer:        PASS / FAIL — <N blocking scripts>
CHECK 5 img alt:      PASS / FAIL — <N images without alt>
CHECK 6 dimensions:   PASS / FAIL — <N images without width/height>
CHECK 7 include:      PASS / FAIL — <N deprecated includes>
CHECK 8 theme-check:  PASS / FAIL — <N errors, N warnings>

Blocking issues: <list>
Recommended fixes before merge/submission: <list>
```

## References

- https://shopify.dev/docs/storefronts/themes/tools/theme-check
- https://shopify.dev/docs/storefronts/themes/architecture/sections
- https://shopify.dev/docs/storefronts/themes/architecture/blocks/theme-blocks
