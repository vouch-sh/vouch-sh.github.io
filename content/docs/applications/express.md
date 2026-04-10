---
title: "Express.js (openid-client)"
description: "Integrate Vouch OIDC authentication into an Express.js application using openid-client."
weight: 3
params:
  category: "server"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

openid-client provides a certified OpenID Connect client for Node.js with manual PKCE, state, and nonce management.

## Example

**[web/express-openid](https://github.com/vouch-sh/examples/tree/main/web/express-openid)** -- Complete working example with authorization code flow, PKCE, and hardware attestation claim extraction from the access token.
