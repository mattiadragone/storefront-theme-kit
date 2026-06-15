---
name: shopify-audit-submission
description: Use when preparing a Shopify theme for Theme Store submission. Runs a full compliance checklist: required templates, section group structure, @app/@theme block support, code rules (no SCSS, no minified source, no obfuscation), color scheme count, font picker variants, demo store requirements, and Partner Program compliance.
---

# Shopify Submission Audit

Use this skill to run a complete Theme Store submission checklist against an existing theme repo. Run after `shopify-audit-critical` and `shopify-audit-quality` — all issues from those audits must be resolved first.

## When to invoke

- Preparing a new theme for first Theme Store submission.
- After a major update to verify continued eligibility.
- When Shopify reviewers reject a submission and you need to identify all gaps.

## Checklist — run each section in order

---

## SECTION 1 — Required templates

```bash
# Check for all required template files
required=(
  "templates/index.json"
  "templates/product.json"
  "templates/collection.json"
  "templates/list-collections.json"
  "templates/blog.json"
  "templates/article.json"
  "templates/page.json"
  "templates/cart.json"
  "templates/search.json"
  "templates/404.json"
  "templates/password.json"
  "templates/gift_card.liquid"
  "templates/customers/login.liquid"
  "templates/customers/register.liquid"
  "templates/customers/account.liquid"
  "templates/customers/order.liquid"
  "templates/customers/addresses.liquid"
  "templates/customers/activate_account.liquid"
  "templates/customers/reset_password.liquid"
  "layout/password.liquid"
  "sections/header-group.json"
  "sections/footer-group.json"
)
for f in "${required[@]}"; do
  [ -f "$f" ] && echo "PRESENT: $f" || echo "MISSING: $f"
done
```

All must be PRESENT. Any MISSING = blocking rejection.

---

## SECTION 2 — Sections Everywhere (OS 2.0)

Header and footer must be section groups, not hardcoded sections.

```bash
# Verify header-group.json and footer-group.json exist and have correct type
cat sections/header-group.json 2>/dev/null | grep '"type"'
cat sections/footer-group.json 2>/dev/null | grep '"type"'
```

Expected: `"type": "header"` and `"type": "footer"` respectively.

```bash
# Check layout/theme.liquid uses {% sections %} not {% section %} for header/footer
grep -n "{% sections\|{% section " layout/theme.liquid
```

Should see `{% sections 'header-group' %}` and `{% sections 'footer-group' %}`, NOT `{% section 'header' %}`.

---

## SECTION 3 — `@theme` and `@app` block support

Every section that accepts blocks must include both `@theme` and `@app` in the blocks array.

```bash
# Find sections with a blocks array but missing @theme or @app
grep -l '"blocks"' sections/*.liquid | while read f; do
  has_theme=$(grep -c '"@theme"' "$f" || true)
  has_app=$(grep -c '"@app"' "$f" || true)
  preset=$(grep -c '"presets"' "$f" || true)
  [ "$preset" -gt 0 ] && [ "$has_theme" -eq 0 ] && echo "MISSING @theme: $f"
  [ "$preset" -gt 0 ] && [ "$has_app" -eq 0 ] && echo "MISSING @app: $f"
done
```

Sections that appear in the theme editor (have presets) MUST accept `@app` blocks. Sections without presets (main-\* sections) are exempt.

---

## SECTION 4 — Code rules

```bash
# No .scss source files
find assets/ -name "*.scss" -o -name "*.sass" 2>/dev/null && echo "FAIL: SCSS files found" || echo "PASS: No SCSS/SASS"

# No .css.map or .js.map files
find assets/ -name "*.map" 2>/dev/null && echo "FAIL: sourcemap files found" || echo "PASS: No sourcemaps"

# No node_modules or package.json at repo root (should not be committed)
[ -d "node_modules" ] && echo "FAIL: node_modules committed" || echo "PASS: No node_modules"

# Check for minified first-party CSS (lines > 500 chars in assets/)
awk 'length > 500 {print FILENAME": line "NR" ("length" chars)"; found=1} END{exit !found}' assets/*.css 2>/dev/null \
  && echo "FAIL: Possible minified CSS" || echo "PASS: CSS appears readable"

# Check for obfuscated JS (eval, atob, fromCharCode patterns in first-party assets)
grep -rn 'eval(\|atob(\|fromCharCode(' assets/global.js assets/theme.js assets/base.js 2>/dev/null \
  && echo "FAIL: Possible obfuscation" || echo "PASS: No obfuscation patterns"
```

---

## SECTION 5 — No hardcoded English strings in templates

```bash
# Find Liquid output that is NOT using the t filter or a variable
grep -rn "{{[^}]*}}" sections/ snippets/ templates/ 2>/dev/null \
  | grep "'[A-Z][a-z]" \
  | grep -v "| t\b\|| escape\|| image_tag\|| asset_url\|| money\|| date\|| handle\|| size" \
  | head -20
```

All user-visible strings must use the `t` filter. Hardcoded English strings cause raw-text failures in non-English stores.

---

## SECTION 6 — Color schemes

Theme Store requires at least 4 color schemes, each with matching foreground/background pairs.

```bash
# Count color_scheme_group entries in settings_schema.json
grep -c '"color_scheme_group"' config/settings_schema.json 2>/dev/null
grep -c '"id"' config/settings_data.json 2>/dev/null

# Check settings_data.json for existing color scheme entries
grep -A2 '"color_schemes"' config/settings_data.json 2>/dev/null | head -20
```

Must have ≥ 4 presets defined. Also verify each scheme has readable foreground/background contrast (≥ 4.5:1).

---

## SECTION 7 — Font picker variants

Every `font_picker` setting must have bold, italic, and bold-italic variants available in the default font family.

```bash
grep -rn '"font_picker"' config/settings_schema.json sections/*.liquid 2>/dev/null | head -10
```

Check each font picker's `"default"` value. Then verify the family supports `_n7` (bold), `_i4` (italic), `_i7` (bold-italic). Test in the Shopify Font Explorer if unsure.

---

## SECTION 8 — Accelerated checkout

Cart page must display accelerated checkout buttons.

```bash
grep -rn "payment_button\|shop_pay\|accelerated" templates/cart.json sections/*.liquid 2>/dev/null | head -10
```

The main cart section must include `{{ form | payment_button }}` inside a `{% form 'cart' %}` block.

---

## SECTION 9 — Gift card

```bash
# gift_card.liquid must exist and contain key elements
grep -n "gift_card\|code\|expiry\|download" templates/gift_card.liquid 2>/dev/null | head -10
```

Must display: card image, gift card code, expiration date, and download link.

---

## SECTION 10 — Predictive search

```bash
grep -rn "predictive-search\|search/suggest\|PredictiveSearch" snippets/ sections/ assets/ 2>/dev/null | head -10
```

Header search must use predictive search (custom element or fetch-based implementation).

---

## SECTION 11 — Partner Program compliance

Manual checks — no automated script can cover these:

- [ ] No code obfuscation (eval, atob, base64-encoded logic in theme JS)
- [ ] No search engine cloaking (different markup for bots vs users)
- [ ] No hidden keyword stuffing or invisible links
- [ ] No mock customer data or fake orders in templates
- [ ] No app-dependent features (theme works with zero apps installed)
- [ ] No dark patterns (hidden costs, auto-selected paid add-ons, fake urgency)
- [ ] Third-party library LICENSE files present in `assets/` or repo root

---

## SECTION 12 — Demo store requirements

Manual check:

- [ ] At least one preset per section type with demo content
- [ ] No Lorem Ipsum
- [ ] No generic stock photo placeholders
- [ ] Real-looking product names, prices, and descriptions
- [ ] Realistic blog posts and images

---

## Full submission checklist — summary

```
### Submission Audit Results — <theme name> — <version> — <date>

SECTION 1  Required templates:     PASS / FAIL — <missing files>
SECTION 2  Section groups:         PASS / FAIL
SECTION 3  @theme + @app blocks:   PASS / FAIL — <sections missing>
SECTION 4  Code rules:             PASS / FAIL — <SCSS / minified / obfuscated>
SECTION 5  Hardcoded strings:      PASS / FAIL — <N raw strings>
SECTION 6  Color schemes (≥4):     PASS / FAIL — <N found>
SECTION 7  Font picker variants:   PASS / FAIL
SECTION 8  Accelerated checkout:   PASS / FAIL
SECTION 9  Gift card:              PASS / FAIL
SECTION 10 Predictive search:      PASS / FAIL
SECTION 11 Partner compliance:     PASS / FAIL — <issues>
SECTION 12 Demo store:             PASS / FAIL

Blocking issues (must fix before submitting):
<list>

Recommended improvements:
<list>
```

## References

- https://shopify.dev/docs/storefronts/themes/store/requirements
- https://shopify.dev/docs/storefronts/themes/store/review-process/submit-theme
- https://www.shopify.com/partners/terms
