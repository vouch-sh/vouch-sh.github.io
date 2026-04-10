---
title: "Vanilla JavaScript"
description: "Integrate Vouch OIDC authentication into a plain JavaScript application using oidc-client-ts."
weight: 24
params:
  category: "spa"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

For browser-based applications, use the authorization code flow with PKCE. No client secret is needed. Vouch does not issue refresh tokens -- when the token expires, redirect the user to sign in again. oidc-client-ts can be loaded via CDN.

## Example

**[spa/vanilla-js](https://github.com/vouch-sh/examples/tree/main/spa/vanilla-js)** -- Complete working example with oidc-client-ts, PKCE, and hardware attestation claim extraction from the access token.
