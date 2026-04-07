# Vouch — Hardware-Backed Developer Credentials

[![Hugo](https://img.shields.io/badge/Hugo-0.142.0-ff4088?logo=hugo)](https://gohugo.io/)
[![Deploy](https://github.com/vouch-sh/vouch-sh.github.io/actions/workflows/hugo.yml/badge.svg)](https://github.com/vouch-sh/vouch-sh.github.io/actions/workflows/hugo.yml)
[![OpenID Certified](https://img.shields.io/badge/OpenID-Certified-green)](https://openid.net/certification/)

Open-source credential broker that issues short-lived SSH, AWS, GitHub, and Kubernetes credentials after FIDO2 hardware verification. One YubiKey tap, 8 hours of access.

**Live site:** [vouch.sh](https://vouch.sh)

## About

Vouch replaces long-lived secrets (SSH keys, AWS access keys, GitHub tokens, Docker credentials) with short-lived, cryptographically attested credentials. No credential is ever issued without proof of human presence via a FIDO2/WebAuthn security key.

After a single `vouch login`, credential helpers for SSH, AWS, GitHub, EKS, Docker, Cargo, CodeArtifact, and CodeCommit provide tokens on demand — transparently and without any long-lived secrets on disk.

This repository contains the Hugo source for the [vouch.sh](https://vouch.sh) product site, documentation, and blog.

## Development

**Prerequisites:** [Hugo](https://gohugo.io/installation/) v0.142.0 extended

Start a local dev server with live reload:

```bash
hugo server
```

Visit [http://localhost:1313](http://localhost:1313).

Build for production:

```bash
hugo --gc --minify
```

Output goes to `public/`.

## Project Structure

```
content/
  docs/               30+ documentation pages (AWS, SSH, EKS, GitHub, Docker, etc.)
  docs/applications/  24 OIDC integration guides (Rails, Next.js, FastAPI, Flutter, etc.)
  blog/               Blog posts
layouts/              Hugo Go templates and partials
assets/css/           Hand-written CSS (no build tools, no npm)
static/               Static files copied to output (CNAME, images, robots.txt)
```

## Deployment

Pushing to `main` triggers a [GitHub Actions workflow](.github/workflows/hugo.yml) that builds the site with Hugo and force-pushes to the `gh-pages` branch, which serves [vouch.sh](https://vouch.sh) via GitHub Pages.

## License

Apache-2.0 / MIT dual license. See the [main Vouch repository](https://github.com/vouch-sh/vouch) for source code.
