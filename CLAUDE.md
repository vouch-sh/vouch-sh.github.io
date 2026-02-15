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
- `content/docs/` — Documentation pages (Getting Started, AWS, SSH, EKS, GitHub, Docker, Cargo, Applications, SCIM)
- `content/blog/` — Blog posts (markdown files)
- `layouts/` — Hugo templates (baseof.html, index.html, docs/, blog/)
- `layouts/shortcodes/` — Reusable shortcodes (e.g., `instance-url`)
- `assets/css/` — Stylesheets processed by Hugo Pipes
- `static/` — Files copied as-is to output (CNAME, robots.txt, images)

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

## Design System

Dark theme with the following palette:
- Background: `#0a0a0a`, Surface: `#141414`, Raised: `#1e1e1e`
- Accent: `#3ecf8e` (teal/green), Accent hover: `#5edba5`
- Text: `#e5e5e5` (primary), `#888` (muted), `#666` (dim)
- Border: `#2a2a2a`
- Font stack: `Inter`, system-ui, sans-serif (body); `JetBrains Mono`, monospace (code)

## Shortcodes

- `{{</* instance-url */>}}` — Renders the configured instance URL (default: `https://us.vouch.sh`). Configured via `params.usInstance` in `hugo.toml`.

## Deployment

Pushing to `main` triggers the GitHub Actions workflow (`.github/workflows/hugo.yml`) which builds Hugo and deploys to GitHub Pages. The `static/CNAME` file maps the custom domain `vouch.sh`.
