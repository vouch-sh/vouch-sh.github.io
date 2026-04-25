# ASP.NET Core (OpenID Connect)

> Integrate Vouch OIDC authentication into an ASP.NET Core application using the built-in OpenID Connect middleware.

Source: https://vouch.sh/docs/applications/aspnet-core/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

ASP.NET Core includes built-in OpenID Connect middleware. Key configuration:

- Requires .NET 10.0 or later with [`Microsoft.AspNetCore.Authentication.OpenIdConnect`](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/configure-oidc-web-authentication)
- Enable PKCE with `options.UsePkce = true`
- Set `GetClaimsFromUserInfoEndpoint = true` and `SaveTokens = true`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode with `JsonDocument.Parse()` after base64url decoding with padding adjustment

## Example

**[web/aspnet-core](https://github.com/vouch-sh/examples/tree/main/web/aspnet-core)** — Complete working example with OIDC middleware, PKCE, and hardware claim extraction.
