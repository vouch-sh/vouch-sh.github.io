# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **vouch.sh** landing page, hosted via GitHub Pages at `vouch.sh` (configured via CNAME). It is a static single-page site with no build process, no dependencies, and no JavaScript.

## Architecture

The entire site is a single `index.html` file with inline CSS and inline SVGs. There are no external resources—fonts, images, and styles are all embedded. The page serves as a region-selection gateway directing users to regional instances (currently only `us.vouch.sh` is active).

## Development

Open `index.html` directly in a browser to preview. There is no build step, linter, or test suite.

## Design System

The color scheme matches the vouch.sh server's Tailwind config:

- Background: `#0f1b2d`, Surface: `#1a2332`, Raised: `#243040`
- Accent: `#539fe5`, Accent hover: `#89bceb`
- Text: `#fafafa` (primary), `#888` (muted)
- Font stack: `ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`

## Deployment

Pushing to `main` triggers GitHub Pages deployment automatically. The `CNAME` file maps the custom domain `vouch.sh`.
