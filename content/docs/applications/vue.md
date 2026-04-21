---
title: "Vue (oidc-client-ts)"
description: "Integrate Vouch OIDC authentication into a Vue single-page application using oidc-client-ts."
weight: 21
params:
  category: "spa"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[oidc-client-ts](https://github.com/authts/oidc-client-ts) provides a `UserManager` for managing the OIDC lifecycle in Vue applications. Key configuration:

- No client secret needed (public client with PKCE, enabled by default)
- Vouch does not issue refresh tokens — redirect the user to sign in again when the token expires
- Configure `UserManager` with `authority`, `client_id`, `redirect_uri`, and `scope`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode with `atob(token.split('.')[1])` after base64url character replacement
- State persistence uses `sessionStorage` by default via `WebStorageStateStore`

## Example

**[spa/vue](https://github.com/vouch-sh/examples/tree/main/spa/vue)** — Complete working example with oidc-client-ts UserManager, PKCE, and hardware claim extraction.
