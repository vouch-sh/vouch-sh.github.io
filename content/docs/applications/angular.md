---
title: "Angular (angular-auth-oidc-client)"
description: "Integrate Vouch OIDC authentication into an Angular single-page application using angular-auth-oidc-client."
weight: 23
params:
  category: "spa"
  language: "TypeScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

For browser-based applications, use the authorization code flow with PKCE. No client secret is needed. Vouch does not issue refresh tokens -- when the token expires, redirect the user to sign in again.

## Example

**[spa/angular](https://github.com/vouch-sh/examples/tree/main/spa/angular)** -- Complete working example with angular-auth-oidc-client, PKCE, and automatic userinfo fetching.
