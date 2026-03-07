---
title: "A2A Agent (Python)"
description: "Build an Agent-to-Agent (A2A) protocol agent secured with Vouch OIDC bearer token authentication."
weight: 50
params:
  category: "agent"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

The [Agent-to-Agent (A2A)](https://google.github.io/A2A/) protocol defines how AI agents discover and communicate with each other. This guide shows how to build an A2A agent secured with Vouch OIDC bearer tokens using the [A2A Python SDK](https://github.com/google/A2A).

## Dependencies

```bash
pip install a2a-sdk PyJWT cryptography uvicorn
```

## Bearer Token Verification

Verify Vouch JWTs from incoming agent requests:

```python
import os
import json
import jwt
from jwt import PyJWKClient

VOUCH_ISSUER = os.environ.get('VOUCH_ISSUER', 'https://{{< instance-url >}}')
PORT = int(os.environ.get('PORT', '3000'))

jwks_client = PyJWKClient(f'{VOUCH_ISSUER}/oauth/jwks')


def verify_bearer_token(request) -> dict | None:
    auth = request.headers.get('authorization', '')
    if not auth.startswith('Bearer '):
        return None
    token = auth[7:]
    try:
        signing_key = jwks_client.get_signing_key_from_jwt(token)
        return jwt.decode(
            token, signing_key.key,
            algorithms=['ES256'], issuer=VOUCH_ISSUER,
            options={'verify_aud': False},
        )
    except Exception:
        return None
```

## Agent Card

The [Agent Card](https://google.github.io/A2A/#/documentation?id=agent-card) advertises your agent's capabilities and security requirements. The `securitySchemes` field tells callers to authenticate via Vouch OIDC:

```python
from a2a.types import (
    AgentCard, AgentCapabilities, AgentSkill,
)

agent_card = AgentCard(
    name='Vouch Identity Agent',
    description='An A2A agent secured with Vouch OIDC.',
    url=f'http://localhost:{PORT}',
    version='1.0.0',
    defaultInputModes=['text/plain'],
    defaultOutputModes=['text/plain'],
    capabilities=AgentCapabilities(streaming=False),
    skills=[
        AgentSkill(
            id='verify-identity',
            name='Verify Identity',
            description='Verifies the caller identity using their '
                        'Vouch OIDC token.',
            tags=['identity', 'security'],
            examples=['Who am I?', 'Verify my identity'],
        ),
    ],
    securitySchemes={
        'vouch_oidc': {
            'type': 'openIdConnect',
            'openIdConnectUrl':
                f'{VOUCH_ISSUER}/.well-known/openid-configuration',
        },
    },
    security=[{'vouch_oidc': []}],
)
```

## Agent Executor

Implement the `AgentExecutor` interface to handle incoming tasks:

```python
from a2a.server.agent_execution import AgentExecutor

class IdentityAgentExecutor(AgentExecutor):
    async def execute(self, context, event_queue):
        result = {
            'message': 'Identity verified via Vouch OIDC',
            'note': 'The caller was authenticated with a '
                    'hardware security key',
        }
        await event_queue.enqueue_event(
            context.create_text_artifact(json.dumps(result, indent=2))
        )

    async def cancel(self, context, event_queue):
        pass
```

## Auth Middleware

Wrap the A2A application with middleware that verifies bearer tokens on all endpoints except the Agent Card discovery endpoint:

```python
from a2a.server.apps import A2AStarletteApplication
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.tasks import InMemoryTaskStore
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

task_store = InMemoryTaskStore()
handler = DefaultRequestHandler(
    agent_executor=IdentityAgentExecutor(),
    task_store=task_store,
)
a2a_app = A2AStarletteApplication(
    agent_card=agent_card,
    http_handler=handler,
)


async def auth_middleware(request: Request, call_next):
    if request.url.path == '/.well-known/agent.json':
        return await call_next(request)

    claims = verify_bearer_token(request)
    if claims is None:
        return JSONResponse(
            {'error': 'Unauthorized',
             'message': 'Valid Vouch Bearer token required'},
            status_code=401,
            headers={'WWW-Authenticate': 'Bearer'},
        )
    request.state.auth = claims
    return await call_next(request)


app = a2a_app.build()
app.add_middleware(BaseHTTPMiddleware, dispatch=auth_middleware)
```

## Running the Agent

```python
import uvicorn

if __name__ == '__main__':
    uvicorn.run(app, host='0.0.0.0', port=PORT)
```

```bash
VOUCH_ISSUER=https://{{< instance-url >}} python agent.py
```

The agent starts on port 3000 with:

- `/.well-known/agent.json` -- Agent Card (unauthenticated, for discovery)
- All other endpoints require a valid Vouch bearer token

## How It Works

1. A calling agent fetches `/.well-known/agent.json` to discover the target agent's capabilities and security requirements.
2. The `securitySchemes` field tells the caller to obtain a Vouch OIDC token.
3. The caller sends requests with a `Bearer` token in the `Authorization` header.
4. The auth middleware verifies the JWT against the Vouch JWKS endpoint and attaches claims to the request state.
5. The `AgentExecutor` processes the task with the caller's verified identity available on the request.
