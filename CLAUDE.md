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

The `layouts/partials/docs-sidebar-nav.html` partial renders the docs sidebar. It iterates `docsSection.Pages.ByWeight`, rendering sections (like Applications) as collapsible groups and leaf pages as flat links.

## Content Patterns

Standard front matter fields:
- `title` — Page title (required)
- `description` — Used in meta tags and card summaries
- `weight` — Controls ordering in sidebar nav and card grids (lower = first)
- `subtitle` — Optional; rendered below the title on docs pages

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

## Deployment

Pushing to `main` triggers the GitHub Actions workflow (`.github/workflows/hugo.yml`) which builds with Hugo v0.142.0 (pinned in the workflow) and force-pushes the built output to the `gh-pages` branch. The `static/CNAME` file maps the custom domain `vouch.sh`.
