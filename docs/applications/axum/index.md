# Axum (openidconnect-rs)

> Integrate Vouch OIDC authentication into a Rust Axum application using openidconnect-rs.

Source: https://vouch.sh/docs/applications/axum/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[openidconnect-rs](https://crates.io/crates/openidconnect) provides a type-safe OpenID Connect client for Rust. Key configuration:

- Use `CoreProviderMetadata::discover_async()` for OIDC auto-discovery
- PKCE is automatic with `PkceCodeChallenge::new_random_sha256()`
- Define a custom claims struct implementing `AdditionalClaims` for type-safe access to Vouch-specific fields
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode the payload after token exchange
- Use tower-sessions for session management (use a persistent store in production)

## Example

**[web/axum-openidconnect](https://github.com/vouch-sh/examples/tree/main/web/axum-openidconnect)** — Complete working example with type-safe claims, PKCE, and hardware claim extraction.
