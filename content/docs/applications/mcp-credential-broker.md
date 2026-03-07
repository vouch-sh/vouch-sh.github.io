---
title: "MCP Credential Broker"
description: "Build an MCP server that brokers AWS, GitHub, and SSH credentials through Vouch identity verification."
weight: 42
params:
  category: "mcp"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

This guide shows how to build an MCP server that brokers cloud credentials on behalf of authenticated users. AI assistants call MCP tools to get temporary AWS credentials, GitHub installation tokens, or signed SSH certificates -- all backed by the caller's Vouch OIDC identity.

The server uses the same FastMCP + `VouchTokenVerifier` pattern from the [MCP Remote Server (Python)]({{< ref "/docs/applications/mcp-server-py" >}}) guide. See that guide for the full token verification setup.

## Dependencies

```bash
pip install mcp PyJWT cryptography httpx
```

## Server Setup

The server uses the same `VouchTokenVerifier` and `AuthSettings` configuration as the basic MCP server. In addition to the verified claims, the credential broker stores the raw token so it can call Vouch credential APIs on behalf of the user:

```python
import os
import json
import contextvars
import jwt
from jwt import PyJWKClient
import httpx
from pydantic import AnyHttpUrl
from mcp.server.fastmcp import FastMCP
from mcp.server.auth.provider import AccessToken, TokenVerifier
from mcp.server.auth.settings import AuthSettings

VOUCH_ISSUER = os.environ.get('VOUCH_ISSUER', 'https://{{< instance-url >}}')
PORT = int(os.environ.get('PORT', '3000'))

jwks_client = PyJWKClient(f'{VOUCH_ISSUER}/oauth/jwks')
_current_claims = contextvars.ContextVar('current_claims', default=None)
_current_token = contextvars.ContextVar('current_token', default=None)


class VouchTokenVerifier(TokenVerifier):
    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            signing_key = jwks_client.get_signing_key_from_jwt(token)
            payload = jwt.decode(
                token, signing_key.key,
                algorithms=['ES256'], issuer=VOUCH_ISSUER,
                options={'verify_aud': False},
            )
            _current_claims.set(payload)
            _current_token.set(token)
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


mcp = FastMCP(
    'vouch-credential-broker',
    host='0.0.0.0', port=PORT, json_response=True,
    token_verifier=VouchTokenVerifier(),
    auth=AuthSettings(
        issuer_url=AnyHttpUrl(VOUCH_ISSUER),
        resource_server_url=AnyHttpUrl(f'http://localhost:{PORT}'),
        required_scopes=[],
    ),
)
```

## AWS Credentials Tool

Exchange the caller's Vouch session for temporary AWS credentials. The tool first obtains an AWS-specific ID token from Vouch, then calls AWS STS `AssumeRoleWithWebIdentity`:

```python
@mcp.tool(name='get-aws-credentials')
async def get_aws_credentials(role_arn: str) -> str:
    """Exchange the user's Vouch session for temporary AWS credentials."""
    token = _current_token.get()
    if not token:
        return json.dumps({'error': 'No authentication context'})

    async with httpx.AsyncClient() as client:
        vouch_resp = await client.get(
            f'{VOUCH_ISSUER}/v1/credentials/aws/token',
            headers={'Authorization': f'Bearer {token}'},
        )

    if vouch_resp.status_code != 200:
        return json.dumps({
            'error': 'Vouch AWS token request failed',
            'status': vouch_resp.status_code,
        })

    aws_id_token = vouch_resp.json()['id_token']

    async with httpx.AsyncClient() as client:
        sts_resp = await client.post(
            'https://sts.amazonaws.com/',
            data={
                'Action': 'AssumeRoleWithWebIdentity',
                'RoleArn': role_arn,
                'RoleSessionName': 'vouch-mcp',
                'WebIdentityToken': aws_id_token,
                'Version': '2011-06-15',
            },
        )

    # Parse STS XML response for credentials
    # Returns: AccessKeyId, SecretAccessKey, SessionToken, Expiration
```

## GitHub Token Tool

Get a GitHub installation token scoped to the caller's identity:

```python
@mcp.tool(name='get-github-token')
async def get_github_token(
    owner: str = '',
    repositories: list[str] | None = None,
) -> str:
    """Get a GitHub installation token via Vouch."""
    token = _current_token.get()
    if not token:
        return json.dumps({'error': 'No authentication context'})

    body = {}
    if owner:
        body['owner'] = owner
    if repositories:
        body['repositories'] = repositories

    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f'{VOUCH_ISSUER}/v1/credentials/github/token',
            headers={'Authorization': f'Bearer {token}'},
            json=body,
        )

    if resp.status_code != 200:
        return json.dumps({
            'error': 'GitHub token request failed',
            'status': resp.status_code,
        })

    return json.dumps(resp.json(), indent=2)
```

## SSH Certificate Tool

Sign an SSH public key with a Vouch-issued certificate:

```python
@mcp.tool(name='get-ssh-certificate')
async def get_ssh_certificate(public_key: str) -> str:
    """Sign an SSH public key with a Vouch-issued certificate."""
    token = _current_token.get()
    if not token:
        return json.dumps({'error': 'No authentication context'})

    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f'{VOUCH_ISSUER}/v1/credentials/ssh',
            headers={'Authorization': f'Bearer {token}'},
            json={'public_key': public_key},
        )

    if resp.status_code != 200:
        return json.dumps({
            'error': 'SSH certificate request failed',
            'status': resp.status_code,
        })

    return json.dumps(resp.json(), indent=2)
```

## How It Works

1. An MCP client authenticates with Vouch and sends the bearer token to the credential broker.
2. The broker verifies the token and stores both the claims and raw token in context variables.
3. When a tool is called, the broker uses the raw token to call Vouch credential APIs on behalf of the authenticated user.
4. Vouch returns short-lived, scoped credentials (AWS temporary credentials, GitHub installation tokens, or SSH certificates) tied to the user's hardware-backed identity.
5. The AI assistant receives the credentials and uses them for the requested operation -- no long-lived secrets are stored.
