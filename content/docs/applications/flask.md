---
title: "Flask (Authlib)"
description: "Integrate Vouch OIDC authentication into a Flask application using Authlib."
weight: 6
params:
  category: "server"
  language: "Python"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

Authlib provides OAuth and OpenID Connect client support for Flask with PKCE (`code_challenge_method='S256'`).

## Example

**[web/flask-authlib](https://github.com/vouch-sh/examples/tree/main/web/flask-authlib)** -- Complete working example with authorization code flow, PKCE, and hardware attestation claim extraction from the access token.
