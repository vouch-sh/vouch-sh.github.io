---
title: "BFF Pattern (Express)"
description: "Secure a single-page application with hardware-backed authentication using the Backend-for-Frontend pattern with Express and openid-client."
weight: 25
params:
  category: "spa"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

The Backend-for-Frontend (BFF) pattern keeps all OAuth tokens on the server. The browser never sees access tokens or ID tokens -- it authenticates via `HttpOnly`, `SameSite=Strict` session cookies instead.

## Example

**[spa/bff-express](https://github.com/vouch-sh/examples/tree/main/spa/bff-express)** -- Complete working example with Express BFF server, openid-client, PKCE, and proxied UserInfo endpoint.
