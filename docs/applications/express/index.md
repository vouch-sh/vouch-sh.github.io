# Express.js (openid-client)

> Integrate Vouch OIDC authentication into an Express.js application using openid-client.

Source: https://vouch.sh/docs/applications/express/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[openid-client](https://github.com/panva/openid-client) provides a certified OpenID Connect client for Node.js. Key configuration:

- Use `client.discovery()` for automatic issuer metadata, then `authorizationCodeGrant()` with PKCE
- Manual state, nonce, and PKCE code verifier management
- Store tokens in `express-session` with `saveUninitialized: false`
- Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are in the access token JWT ([RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068)) — decode with `Buffer.from(token.split('.')[1], 'base64url')`

## Example

**[web/express-openid](https://github.com/vouch-sh/examples/tree/main/web/express-openid)** — Complete working example with authorization code flow, PKCE, session management, and token introspection.
