---
title: "React (react-oidc-context)"
description: "Integrate Vouch OIDC authentication into a React single-page application using react-oidc-context."
weight: 20
params:
  category: "spa"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

For browser-based applications, use the authorization code flow with PKCE. No client secret is needed. Vouch does not issue refresh tokens -- when the token expires, redirect the user to sign in again.

## Example

**[spa/react](https://github.com/vouch-sh/examples/tree/main/spa/react)** -- Complete working example with react-oidc-context, PKCE, and hardware attestation claim extraction from the access token.
