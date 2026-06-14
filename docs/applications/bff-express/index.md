# BFF Pattern (Express)

> Secure a single-page application with hardware-backed authentication using the Backend-for-Frontend pattern with Express and openid-client.

Source: https://vouch.sh/docs/applications/bff-express/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

The Backend-for-Frontend (BFF) pattern keeps all OAuth tokens on the server. The browser never sees access tokens or ID tokens — it authenticates via `HttpOnly`, `SameSite=Strict` session cookies instead. Key configuration:

- Client secret required (confidential client, since tokens are server-side)
- PKCE is automatic with [`openid-client`](https://github.com/panva/openid-client)
- Set `httpOnly: true`, `sameSite: 'strict'`, and `secure: true` (in production) on session cookies
- Proxy a `/api/me` endpoint so the frontend never handles tokens directly
- The hardware attestation claim (`hardware_verified`) is decoded server-side from the access token JWT

## Example

**[spa/bff-express](https://github.com/vouch-sh/examples/tree/main/spa/bff-express)** — Complete working example with Express BFF server, openid-client, PKCE, and proxied UserInfo endpoint.
