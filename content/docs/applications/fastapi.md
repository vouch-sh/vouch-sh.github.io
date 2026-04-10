---
title: "FastAPI (Authlib)"
description: "Integrate Vouch OIDC authentication into a FastAPI application using Authlib."
weight: 7
params:
  category: "server"
  language: "Python"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

Authlib provides OAuth and OpenID Connect client support for Starlette-based applications with PKCE (`code_challenge_method='S256'`).

## Example

**[web/fastapi-authlib](https://github.com/vouch-sh/examples/tree/main/web/fastapi-authlib)** -- Complete working example with authorization code flow, PKCE, and hardware attestation claim extraction from the access token.
