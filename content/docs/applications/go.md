---
title: "Go (go-oidc)"
description: "Integrate Vouch OIDC authentication into a Go application using go-oidc and oauth2."
weight: 10
params:
  category: "server"
  language: "Go"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

go-oidc and the standard oauth2 package provide OIDC discovery and PKCE support (`oauth2.GenerateVerifier()`).

## Example

**[web/go-oidc](https://github.com/vouch-sh/examples/tree/main/web/go-oidc)** -- Complete working example with OIDC discovery, PKCE, and hardware attestation claim extraction from the access token.
