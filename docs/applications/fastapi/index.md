# FastAPI (Authlib)

> Integrate Vouch OIDC authentication into a FastAPI application using Authlib.

Source: https://vouch.sh/docs/applications/fastapi/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[Authlib](https://authlib.org/) provides OAuth and OpenID Connect client support for Starlette-based applications. Key configuration:

- Register the provider with `server_metadata_url` and `code_challenge_method='S256'` for PKCE
- Add `SessionMiddleware` with a secret key before the auth middleware
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode the payload with base64url and padding adjustment
- Callback URL: `/callback`

## Example

**[web/fastapi-authlib](https://github.com/vouch-sh/examples/tree/main/web/fastapi-authlib)** — Complete working example with authorization code flow, PKCE, session middleware, and hardware claim extraction.
