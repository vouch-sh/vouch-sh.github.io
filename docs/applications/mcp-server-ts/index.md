# MCP Remote Server (TypeScript)

> Build an MCP remote server with Vouch OIDC bearer token authentication using the TypeScript SDK.

Source: https://vouch.sh/docs/applications/mcp-server-ts/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets AI assistants call tools on remote servers. This guide covers building an MCP remote server that requires Vouch OIDC bearer tokens, using the official TypeScript SDK and [`jose`](https://github.com/panva/jose) for JWT verification.

The server validates access tokens against the Vouch JWKS endpoint and extracts the hardware attestation claim (`hardware_verified`) from the JWT payload. Tools can gate sensitive operations on hardware key attestation.

## Example

**[mcp/remote-server-ts](https://github.com/vouch-sh/examples/tree/main/mcp/remote-server-ts)** -- Complete working example with bearer token verification, [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) Protected Resource Metadata, and per-user MCP server instances.
