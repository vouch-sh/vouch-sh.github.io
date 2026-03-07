---
title: "MCP Remote Server (TypeScript)"
description: "Build an MCP remote server with Vouch OIDC bearer token authentication using the TypeScript SDK."
weight: 40
params:
  category: "mcp"
  language: "TypeScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets AI assistants call tools on remote servers. This guide shows how to build an MCP remote server that requires Vouch OIDC bearer tokens, using the official TypeScript SDK and [jose](https://github.com/panva/jose) for JWT verification.

## Dependencies

```bash
npm install express @modelcontextprotocol/sdk jose
```

## Protected Resource Metadata

Publish an [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) metadata endpoint so MCP clients can discover your authorization server automatically:

```typescript
import express from 'express';

const VOUCH_ISSUER = process.env.VOUCH_ISSUER || 'https://{{< instance-url >}}';
const PORT = parseInt(process.env.PORT || '3000');

const app = express();
app.use(express.json());

app.get('/.well-known/oauth-protected-resource', (_req, res) => {
  res.json({
    resource: process.env.VOUCH_AUDIENCE || `http://localhost:${PORT}`,
    authorization_servers: [VOUCH_ISSUER],
    bearer_methods_supported: ['header'],
    scopes_supported: ['openid', 'email'],
  });
});
```

## Bearer Token Verification

Verify Vouch JWTs using the JWKS endpoint. The middleware extracts the `email`, `sub`, and hardware attestation claims from the token:

```typescript
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(
  new URL(`${VOUCH_ISSUER}/oauth/jwks`)
);

interface AuthInfo {
  email: string;
  sub: string;
  hardwareVerified: boolean;
  hardwareAaguid: string | null;
  rawToken: string;
}

async function verifyToken(
  req: express.Request,
  res: express.Response,
  next: express.NextFunction,
) {
  const auth = req.headers.authorization;
  if (!auth?.startsWith('Bearer ')) {
    res.status(401).json({
      jsonrpc: '2.0',
      error: { code: -32001, message: 'Unauthorized' },
      id: null,
    });
    return;
  }

  const token = auth.slice(7);
  try {
    const { payload } = await jwtVerify(token, JWKS, {
      issuer: VOUCH_ISSUER,
    });
    (req as any).auth = {
      email: payload.email as string,
      sub: payload.sub as string,
      hardwareVerified: payload.hardware_verified as boolean,
      hardwareAaguid: (payload.hardware_aaguid as string) || null,
      rawToken: token,
    };
    next();
  } catch {
    res.status(401).json({
      jsonrpc: '2.0',
      error: { code: -32001, message: 'Invalid token' },
      id: null,
    });
  }
}
```

## MCP Server and Tools

Create an MCP server with tools that access the authenticated user's identity:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import {
  StreamableHTTPServerTransport
} from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { randomUUID } from 'node:crypto';

function createMcpServer(auth: AuthInfo) {
  const server = new McpServer({
    name: 'vouch-example',
    version: '1.0.0',
  });

  server.tool('whoami', 'Returns the authenticated user info', {},
    async () => ({
      content: [{
        type: 'text',
        text: JSON.stringify({
          email: auth.email,
          sub: auth.sub,
          hardware_verified: auth.hardwareVerified,
          hardware_aaguid: auth.hardwareAaguid,
        }, null, 2),
      }],
    }),
  );

  server.tool('sensitive-action',
    'Requires hardware key verification', {},
    async () => {
      if (!auth.hardwareVerified) {
        return {
          isError: true,
          content: [{
            type: 'text',
            text: JSON.stringify({
              error: 'hardware_key_required',
              message: 'This action requires hardware_verified=true.',
            }),
          }],
        };
      }
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({
            status: 'success',
            hardware_aaguid: auth.hardwareAaguid,
          }, null, 2),
        }],
      };
    },
  );

  return server;
}
```

## Session Management

Each authenticated session gets its own transport and MCP server instance:

```typescript
const transports = new Map<string, StreamableHTTPServerTransport>();

app.post('/mcp', verifyToken, async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string | undefined;
  let transport: StreamableHTTPServerTransport;

  if (sessionId && transports.has(sessionId)) {
    transport = transports.get(sessionId)!;
  } else {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      enableJsonResponse: true,
    });
    const server = createMcpServer((req as any).auth);
    await server.connect(transport);
    transport.onclose = () => {
      if (transport.sessionId) transports.delete(transport.sessionId);
    };
  }

  await transport.handleRequest(req, res, req.body);
  if (transport.sessionId && !transports.has(transport.sessionId)) {
    transports.set(transport.sessionId, transport);
  }
});

app.get('/mcp', verifyToken, async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string | undefined;
  if (!sessionId || !transports.has(sessionId)) {
    res.status(400).json({ error: 'Invalid session' });
    return;
  }
  await transports.get(sessionId)!.handleRequest(req, res);
});

app.delete('/mcp', verifyToken, async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string | undefined;
  if (sessionId && transports.has(sessionId)) {
    await transports.get(sessionId)!.handleRequest(req, res);
    transports.delete(sessionId);
  } else {
    res.status(400).json({ error: 'Invalid session' });
  }
});

app.listen(PORT);
```

## How It Works

1. The MCP client discovers the authorization server via the `/.well-known/oauth-protected-resource` endpoint (RFC 9728).
2. The client obtains a Vouch OIDC token and sends it as a `Bearer` token in the `Authorization` header.
3. The `verifyToken` middleware validates the JWT against the Vouch JWKS endpoint and extracts identity claims.
4. Each session gets its own `McpServer` instance with the authenticated user's identity bound to every tool call.
5. Tools can check `hardware_verified` to gate sensitive operations on hardware key attestation.
