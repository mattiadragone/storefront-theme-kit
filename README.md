# claude-shopify-themes

A **Claude Code plugin** for building and auditing Shopify themes. Install it once and you get three
action skills plus a full Shopify-theme knowledge base — no manual copying into `.claude/skills/`.

The project separates two things that used to be tangled together:

- **`knowledge/`** — the *rules*. One file per Shopify area (sections, CSS, performance…). The single
  source of truth. No actions, just rules, examples, and thresholds.
- **`skills/`** — the *actions*. A small set of operation skills (build, audit, tooling) that **read from
  `knowledge/`** based on the operation in progress.

Change a rule once in `knowledge/`, and every skill that uses it stays correct.

---

## Architecture

```
claude-shopify-themes/
│
├── .claude-plugin/             ← PLUGIN manifest + marketplace catalog
│   ├── plugin.json             name, version, description
│   └── marketplace.json        lets users add this repo as a marketplace
│
├── knowledge/                  ← RULES (single source of truth)
│   ├── universal.md            cross-cutting rules — load for any task
│   ├── areas/                  sections · snippets · blocks · templates · layout · assets · config · locales
│   ├── languages/              liquid · css · javascript
│   └── quality/                performance · accessibility · design · theme-store
│
└── skills/                     ← ACTIONS (read from knowledge/)
    ├── shopify-build/          create or modify theme code
    ├── shopify-audit/          inspect an existing repo (read-only report)
    └── shopify-tooling/        CLI, theme-check, Prettier, version control, build pipelines
```

The skills read knowledge via `${CLAUDE_PLUGIN_ROOT}/knowledge/` when installed as a plugin, falling
back to `knowledge/` at the repo root when run from a clone.

### Why three skills

A skill is worth having only if it is a useful **action**. We keep the three that are:

| Skill | Action | What it does |
|---|---|---|
| **shopify-build** | *Create / edit code* | Routes to the right `knowledge/` files by the file type you touch, then writes conforming code. Designing a section/block happens here (it reads `quality/design.md`). |
| **shopify-audit** | *Inspect & report* | Read-only. Runs `shopify theme check` + targeted greps across three levels (critical → quality → submission). Each finding shows severity, location, the fix, **why it matters**, and a link to the knowledge file. |
| **shopify-tooling** | *Set up / run the toolchain* | Shopify CLI, theme-check config, Prettier, Theme Inspector, version control strategy, CI, build pipelines. |

Things that are **not** standalone actions live in `knowledge/` instead:

- **Design** → `knowledge/quality/design.md` (read by `shopify-build` when designing).
- **Optimize** → `shopify-audit` finds perf/a11y issues, `shopify-build` fixes them.
- **Submit** → the submission level of `shopify-audit` + `knowledge/quality/theme-store.md`.

---

## How it works in practice

1. You ask for something (e.g. "create a hero section").
2. The matching skill triggers (`shopify-build`).
3. The skill reads `knowledge/universal.md` plus the files for what you're touching
   (`areas/sections.md`, `languages/css.md`, `areas/locales.md`, `quality/design.md`).
4. It writes code that follows those rules.

For an audit, `shopify-audit` runs `theme check` and the level's checks, then reports findings with the
knowledge reference explaining each one.

---

## Example prompts

**Build**
- "Create a hero section with a heading, subheading, image, and CTA."
- "Add a testimonials block that nests inside the reviews section."
- "Refactor `snippets/product-card.liquid` to use responsive `image_tag`."
- "Make the cart drawer keyboard-accessible with focus trapping."

**Audit**
- "Audit the theme before I merge this branch."
- "Check this theme for Theme Store submission readiness."
- "Why did my Lighthouse score drop? Run the quality audit."

**Tooling**
- "Set up Shopify CLI and start a local dev server."
- "Add a theme-check config and a CI workflow that fails on errors."
- "Configure Prettier for Liquid and format all sections."

---

## Installation (Claude Code plugin)

Add this repo as a marketplace, then install the plugin:

```text
/plugin marketplace add mattiadragone/claude-shopify-themes
/plugin install claude-shopify-themes@mattiadragone
```

The skills become available namespaced under the plugin — Claude invokes them automatically when a task
matches, or you can call them explicitly:

```text
/claude-shopify-themes:shopify-build
/claude-shopify-themes:shopify-audit
/claude-shopify-themes:shopify-tooling
```

Update later with `/plugin marketplace update mattiadragone`.

### Try it locally without installing

```bash
git clone https://github.com/mattiadragone/claude-shopify-themes
claude --plugin-dir ./claude-shopify-themes
```

---

## Usage

Once installed, **just describe what you want** in a session inside your Shopify theme repo — the right
skill triggers automatically:

- *"Create a hero section with a heading and CTA"* → `shopify-build` writes the section, scoping CSS,
  adding `t:` keys and lazy-loaded images.
- *"Audit the theme before I merge"* → `shopify-audit` runs `shopify theme check` + its checks and reports
  each finding with the fix, why it matters, and the knowledge file to read.
- *"Set up Shopify CLI and a theme-check CI"* → `shopify-tooling` configures the toolchain.

You can also invoke a skill explicitly with `/claude-shopify-themes:shopify-build` (and `:shopify-audit`,
`:shopify-tooling`). The skills read the rules from `knowledge/` on demand, so the same project rules are
applied whether you build or audit.

---

## Skills reference

| Skill | Trigger | Reads from |
|---|---|---|
| `shopify-build` | Creating/editing any theme file or its Liquid/CSS/JS | `universal.md` + matching `areas/`, `languages/`, `quality/` files |
| `shopify-audit` | Inspecting an existing repo before merge/submission | `universal.md` + `quality/*` for the "why" |
| `shopify-tooling` | Setting up/running CLI, linting, CI, build pipelines | `quality/performance.md`, `quality/theme-store.md` |

## Knowledge reference

See [`knowledge/README.md`](knowledge/README.md) for the full index and the skill→knowledge routing
tables.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE) — Mattia Dragone

## Sources

- [Skeleton theme](https://github.com/Shopify/skeleton-theme)
- [Horizon rules reference](https://github.com/Shopify/horizon/tree/main/.cursor)
- [Shopify themes docs](https://shopify.dev/docs/storefronts/themes)
- [Theme Store requirements](https://shopify.dev/docs/storefronts/themes/store/requirements)
