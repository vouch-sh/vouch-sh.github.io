---
title: "React (react-oidc-context)"
description: "Integrate Vouch OIDC authentication into a React single-page application using react-oidc-context."
weight: 20
params:
  category: "spa"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[react-oidc-context](https://github.com/authts/react-oidc-context) wraps [oidc-client-ts](https://github.com/authts/oidc-client-ts) in a React context provider. Key configuration:

- No client secret needed (public client with PKCE, enabled by default)
- Vouch does not issue refresh tokens — redirect the user to sign in again when the token expires
- Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are in the access token JWT — decode with `atob(token.split('.')[1])` after base64url character replacement
- Access the token via `auth.user.access_token`

## Example

**[spa/react](https://github.com/vouch-sh/examples/tree/main/spa/react)** — Complete working example with react-oidc-context, PKCE, and hardware claim extraction from the access token.
