---
title: "MCP Remote Server (Python)"
description: "Build an MCP remote server with Vouch OIDC bearer token authentication using the Python FastMCP SDK."
weight: 41
params:
  category: "mcp"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets AI assistants call tools on remote servers. This guide shows how to build an MCP remote server with Vouch OIDC authentication using the Python [FastMCP](https://github.com/modelcontextprotocol/python-sdk) SDK, which has built-in support for OAuth-based auth and RFC 9728 Protected Resource Metadata.

## Dependencies

```bash
pip install mcp PyJWT cryptography
```

## Token Verifier

Implement the `TokenVerifier` interface to validate Vouch JWTs using the JWKS endpoint. A `contextvars.ContextVar` makes the authenticated claims available to tool handlers:

```python
import os
import json
import contextvars
import jwt
from jwt import PyJWKClient
from mcp.server.auth.provider import AccessToken, TokenVerifier

VOUCH_ISSUER = os.environ.get('VOUCH_ISSUER', 'https://{{< instance-url >}}')

jwks_client = PyJWKClient(f'{VOUCH_ISSUER}/oauth/jwks')
_current_claims = contextvars.ContextVar('current_claims', default=None)


class VouchTokenVerifier(TokenVerifier):
    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            signing_key = jwks_client.get_signing_key_from_jwt(token)
            payload = jwt.decode(
                token,
                signing_key.key,
                algorithms=['ES256'],
                issuer=VOUCH_ISSUER,
                options={'verify_aud': False},
            )
            _current_claims.set(payload)
            return AccessToken(
                token=token,
                client_id=payload.get('sub'),
                scopes=(
                    payload.get('scope', '').split()
                    if isinstance(payload.get('scope'), str)
                    else []
                ),
            )
        except Exception:
            return None
```

## Server Configuration

FastMCP handles RFC 9728 metadata and bearer token extraction automatically when you provide `AuthSettings` and a `TokenVerifier`:

```python
from pydantic import AnyHttpUrl
from mcp.server.fastmcp import FastMCP
from mcp.server.auth.settings import AuthSettings

PORT = int(os.environ.get('PORT', '3000'))

mcp = FastMCP(
    'vouch-example',
    host='0.0.0.0',
    port=PORT,
    json_response=True,
    token_verifier=VouchTokenVerifier(),
    auth=AuthSettings(
        issuer_url=AnyHttpUrl(VOUCH_ISSUER),
        resource_server_url=AnyHttpUrl(f'http://localhost:{PORT}'),
        required_scopes=[],
    ),
)
```

## Tool Registration

Register tools that access the authenticated user's claims through the context variable:

```python
@mcp.tool()
async def whoami() -> str:
    """Returns the authenticated user info from the Vouch OIDC token."""
    claims = _current_claims.get()
    if claims:
        return json.dumps({
            'email': claims.get('email', 'unknown'),
            'sub': claims.get('sub', 'unknown'),
            'hardware_verified': claims.get('hardware_verified', False),
        }, indent=2)
    return json.dumps({'error': 'No authentication context'}, indent=2)


@mcp.tool(name='sensitive-action')
async def sensitive_action() -> str:
    """Requires hardware key verification."""
    claims = _current_claims.get()
    if not claims or not claims.get('hardware_verified', False):
        return json.dumps({
            'error': 'hardware_key_required',
            'message': 'This action requires hardware_verified=true.',
        }, indent=2)
    return json.dumps({
        'status': 'success',
        'hardware_aaguid': claims.get('hardware_aaguid'),
    }, indent=2)
```

## Running the Server

```python
if __name__ == '__main__':
    mcp.run(transport='streamable-http')
```

```bash
VOUCH_ISSUER=https://{{< instance-url >}} python server.py
```

The server starts on port 3000 with:

- `/.well-known/oauth-protected-resource` -- RFC 9728 metadata pointing MCP clients to Vouch
- `/mcp` -- Streamable HTTP transport endpoint (POST, GET, DELETE)

## How It Works

1. FastMCP publishes Protected Resource Metadata so MCP clients discover Vouch as the authorization server.
2. The client obtains a Vouch OIDC token and sends it as a `Bearer` token in the `Authorization` header.
3. `VouchTokenVerifier` validates the JWT against the Vouch JWKS endpoint and stores the claims in a context variable.
4. Tool handlers read `_current_claims` to access the authenticated user's identity, including `hardware_verified` for gating sensitive operations.
