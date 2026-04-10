---
title: "Go (go-oidc)"
description: "Integrate Vouch OIDC authentication into a Go application using go-oidc and oauth2."
weight: 10
params:
  category: "server"
  language: "Go"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[go-oidc](https://github.com/coreos/go-oidc) and the standard [oauth2](https://pkg.go.dev/golang.org/x/oauth2) package provide OIDC support for Go. Key configuration:

- Initialize the provider with `oidc.NewProvider()` for auto-discovery
- Generate PKCE verifier with `oauth2.GenerateVerifier()` and `oauth2.S256ChallengeOption()`
- Manual state, nonce, and PKCE verifier management required
- Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are in the access token JWT — decode the payload into a struct after token exchange

## Example

**[web/go-oidc](https://github.com/vouch-sh/examples/tree/main/web/go-oidc)** — Complete working example with OIDC discovery, PKCE, and hardware claim extraction.
