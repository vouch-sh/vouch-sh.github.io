# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **vouch.sh** product site — a Hugo-based static site hosted via GitHub Pages at `vouch.sh`. It includes a landing page, documentation, and blog.

## Architecture

- **Static site generator**: Hugo (v0.142.0 extended)
- **Templating**: Hugo's Go templates in `layouts/`
- **Content**: Markdown files in `content/` with YAML front matter
- **Styling**: Hand-written CSS in `assets/css/main.css` (processed by Hugo Pipes)
- **No external dependencies**: No Node.js, no npm, no Tailwind build step

Key directories:
- `content/docs/` — Documentation pages (Getting Started, CLI Reference, AWS, SSH, EKS, GitHub, Docker, Cargo, CodeCommit, CodeArtifact, SCIM)
- `content/docs/applications/` — Framework-specific OIDC integration guides (13 guides across server, SPA, and native categories)
- `content/blog/` — Blog posts (markdown files)
- `layouts/` — Hugo templates
- `layouts/partials/` — Shared partials (e.g., `docs-sidebar-nav.html`)
- `layouts/shortcodes/` — Reusable shortcodes (e.g., `instance-url`)
- `assets/css/` — Stylesheets processed by Hugo Pipes
- `static/` — Files copied as-is to output (CNAME, robots.txt, images)

### Template hierarchy

All pages inherit from `layouts/_default/baseof.html`, which defines the HTML shell, nav header (driven by `menus.main` in `hugo.toml`), `{{ block "main" . }}` content area, and footer.

Layout resolution:
- **Homepage**: `layouts/index.html`
- **Docs list** (`/docs/`): `layouts/docs/list.html` — sidebar + card grid of child pages
- **Docs page**: `layouts/docs/single.html` — sidebar + article content
- **Applications list** (`/docs/applications/`): `layouts/docs/applications/list.html` — groups child pages into Server / SPA / Native sections using `where .Pages.ByWeight ".Params.category"` queries
- **Blog**: `layouts/blog/list.html` and `layouts/blog/single.html`
- **Everything else**: `layouts/_default/list.html` and `layouts/_default/single.html`

The `layouts/partials/docs-sidebar-nav.html` partial renders the docs sidebar. Pages are bucketed by `params.docsGroup` (see below), and within each group ordered `ByWeight`; sections (like Applications) render as collapsible groups, leaf pages as flat links.

## Content Patterns

Standard front matter fields:
- `title` — Page title (required)
- `description` — Used in meta tags and card summaries
- `weight` — Ordering *within the page's sidebar group* (lower = first); not global
- `subtitle` — Optional; rendered below the title on docs pages

Docs front matter adds:
- `params.docsGroup` — Which sidebar/landing group the page belongs to: `featured` ("Start Here": getting-started, rollout, startups), `aws` (everything that depends on the shared OIDC provider + IAM role), `integrations` (SSH, GitHub, Docker, Cargo, Kubernetes, SPIFFE, AI APIs), `admin`, or `reference`. Group order and labels live in `hugo.toml` (`params.sidebarGroups`) + `i18n/en.toml` (`sidebar_group_<key>`, `docs_group_<key>_title/desc` referenced by `layouts/docs/list.html`).

Integration guides follow a shared structure: a `{{</* tldr */>}}` box right after the intro (prereq chain, one-time admin action, per-developer command), `{{</* role admin */>}}` / `{{</* role developer */>}}` labels under step headings, actionable steps first, and "How it works" / comparison material demoted below them. `content/docs/rollout.md` is the team-rollout playbook these pages hang off; it owns the canonical offboarding section.

Application guide front matter adds:
- `params.category` — One of `server`, `spa`, or `native`; determines which section the card appears in on the applications list page
- `params.language` — Displayed as a badge on framework cards (e.g., "JavaScript", "Python", "Rust")

## Development

```bash
hugo server
```

This starts a local dev server with live reload at `http://localhost:1313/`.

To build for production:
```bash
hugo --gc --minify
```

Output goes to `public/`.

## Hugo Config

Key settings in `hugo.toml`:
- `markup.goldmark.renderer.unsafe = true` — Raw HTML is allowed in markdown content
- `markup.highlight.style = "monokai"` — Syntax highlighting theme
- `menus.main` — Top nav links (Docs, Blog, About, GitHub); the GitHub entry uses `params.external = true` for `target="_blank"`
- `params.usInstance` — Base URL for the Vouch US instance, used by the `instance-url` shortcode

## Design System

Dark theme using CSS custom properties defined in `assets/css/main.css`:
- `--bg` / `--bg-surface` / `--bg-raised` / `--bg-code` — Background hierarchy
- `--accent` / `--accent-hover` / `--accent-muted` — Teal/green accent (`#3ecf8e`)
- `--text` / `--text-muted` / `--text-dim` — Text hierarchy
- `--border` / `--border-light` — Borders
- `--font-sans` / `--font-mono` — Font stacks (Inter / JetBrains Mono)
- `--max-width` / `--content-width` / `--sidebar-width` / `--nav-height` — Layout dimensions

## Shortcodes

- `{{</* instance-url */>}}` — Renders the configured instance URL (default: `https://us.vouch.sh`). Configured via `params.usInstance` in `hugo.toml`.
- `{{</* tldr */>}}...{{</* /tldr */>}}` — Accent-bordered quick-start box at the top of integration guides (`.tldr` in main.css; title via `tldr_title` i18n key). Inner content is markdown.
- `{{</* role admin */>}}` / `{{</* role developer */>}}` — Pill label under a heading marking one-time admin work vs. per-developer work (`role_admin`/`role_developer` i18n keys).
- `{{</* session-note */>}}` — One-line "run `vouch login`" reminder (`session_note` i18n key); replaces repeated 8-hour-session boilerplate.

## Internationalization (i18n)

The site runs in Hugo multilingual mode with English (`en`) as the only language
today. It is **translation-ready**: all reusable UI strings are extracted, so a
new language requires no template edits.

Where translatable text lives:
- **Reusable UI strings** (nav, footer, sidebar labels, breadcrumbs, buttons,
  aria-labels, JS button states, blog meta, applications/docs section headings)
  → `i18n/en.toml`, referenced in templates with `{{ i18n "key" }}`. Nav menu
  names map via each menu entry's `identifier` in `hugo.toml`; sidebar group
  labels resolve to `sidebar_group_<key>`.
- **Page-specific prose** → content front matter and bodies. The homepage copy
  lives in `content/_index.md` front matter (`hero`, `problem`, `howItWorks`,
  `integrations`, `agents`, `openSource`, `regions`); `layouts/index.html`
  renders it via `.Params`.

To add a language (e.g. Japanese), no template changes are needed:
1. Add a block to `hugo.toml`: `[languages.ja]` with `label`, `locale`, `weight`
   (English stays at `/`; new languages render under `/ja/`).
2. Copy `i18n/en.toml` → `i18n/ja.toml` and translate the `other` values.
3. Add translated content as sibling files: `content/_index.ja.md`,
   `content/docs/aws.ja.md`, etc. Untranslated pages fall back to English.
4. When ≥2 languages exist, add a language-switcher partial (iterate
   `.Translations`) and include it in `layouts/_default/baseof.html`.

Not yet i18n-keyed (revisit when adding a language): the JSON-LD FAQ block in
`layouts/partials/structured-data.html` (mirrors `content/docs/faq.md`) and
internal link paths in `layouts/index.html` (use `relLangURL` for per-language
prefixes).

## Deployment

Pushing to `main` triggers the GitHub Actions workflow (`.github/workflows/hugo.yml`) which builds with Hugo v0.142.0 (pinned in the workflow) and force-pushes the built output to the `gh-pages` branch. The `static/CNAME` file maps the custom domain `vouch.sh`.

## Stream Timeout Prevention

1. Do each numbered task ONE AT A TIME. Complete one task fully,
   confirm it worked, then move to the next.
2. Never write a file longer than ~150 lines in a single tool call.
   If a file will be longer, write it in multiple append/edit passes.
3. Start a fresh session if the conversation gets long (20+ tool calls).
   The error gets worse as the session grows.
4. Keep individual grep/search outputs short. Use flags like
   --include and -l (list files only) to limit output size.
5. If you do hit the timeout, retry the same step in a shorter form.
   Don't repeat the entire task from scratch.

## Committing Changes

- When committing, write the message from the user's description — do NOT run `git diff` or `git log` to read all changes first.
- Only run `git status` to see which files are staged, then commit with the message the user provided.
- This avoids large diff output that causes stream idle timeouts.
- After completing each discrete task or feature, commit immediately before moving to the next task. Keep commits small and frequent.