# Flask (Authlib)

> Integrate Vouch OIDC authentication into a Flask application using Authlib.

Source: https://vouch.sh/docs/applications/flask/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[Authlib](https://authlib.org/) provides OAuth and OpenID Connect client support for Flask. Key configuration:

- Install [`authlib`](https://authlib.org/), `flask`, and `requests`
- Register the provider with `server_metadata_url` and `code_challenge_method='S256'` for PKCE
- Set `client_kwargs={'scope': 'openid email'}`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode the payload with base64url and padding adjustment
- Callback URL: `/callback`

## Example

**[web/flask-authlib](https://github.com/vouch-sh/examples/tree/main/web/flask-authlib)** — Complete working example with authorization code flow, PKCE, and hardware claim extraction.
