---
title: "Angular (angular-auth-oidc-client)"
description: "Integrate Vouch OIDC authentication into an Angular single-page application using angular-auth-oidc-client."
weight: 23
params:
  category: "spa"
  language: "TypeScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[angular-auth-oidc-client](https://github.com/damienbod/angular-auth-oidc-client) is a certified OpenID Connect library for Angular. Key configuration:

- No client secret needed (public client with PKCE, enabled by default)
- Vouch does not issue refresh tokens — redirect the user to sign in again when the token expires
- Set `autoUserInfo: true` for automatic userinfo fetching
- Use `OidcSecurityService.checkAuth()` to get `{ isAuthenticated, userData, accessToken }`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode with `atob(token.split('.')[1])` after base64url character replacement

## Example

**[spa/angular](https://github.com/vouch-sh/examples/tree/main/spa/angular)** — Complete working example with angular-auth-oidc-client, PKCE, and hardware claim extraction.
