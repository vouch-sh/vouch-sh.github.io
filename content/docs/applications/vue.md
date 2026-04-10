---
title: "Vue (oidc-client-ts)"
description: "Integrate Vouch OIDC authentication into a Vue single-page application using oidc-client-ts."
weight: 21
params:
  category: "spa"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

For browser-based applications, use the authorization code flow with PKCE. No client secret is needed. Vouch does not issue refresh tokens -- when the token expires, redirect the user to sign in again.

## Example

**[spa/vue](https://github.com/vouch-sh/examples/tree/main/spa/vue)** -- Complete working example with oidc-client-ts UserManager, PKCE, and hardware attestation claim extraction from the access token.
