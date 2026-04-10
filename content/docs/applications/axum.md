---
title: "Axum (openidconnect-rs)"
description: "Integrate Vouch OIDC authentication into a Rust Axum application using openidconnect-rs."
weight: 9
params:
  category: "server"
  language: "Rust"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

openidconnect-rs provides a type-safe OpenID Connect client for Rust with PKCE support (`PkceCodeChallenge::new_random_sha256()`).

## Example

**[web/axum-openidconnect](https://github.com/vouch-sh/examples/tree/main/web/axum-openidconnect)** -- Complete working example with type-safe claims, PKCE, and hardware attestation claim extraction from the access token.
