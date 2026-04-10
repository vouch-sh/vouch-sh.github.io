---
title: "ASP.NET Core (OpenID Connect)"
description: "Integrate Vouch OIDC authentication into an ASP.NET Core application using the built-in OpenID Connect middleware."
weight: 11
params:
  category: "server"
  language: "C#"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

ASP.NET Core includes built-in OpenID Connect middleware with PKCE support (`options.UsePkce = true`). Requires .NET 10.0 or later.

## Example

**[web/aspnet-core](https://github.com/vouch-sh/examples/tree/main/web/aspnet-core)** -- Complete working example with OIDC middleware, PKCE, and hardware attestation claim extraction from the access token.
