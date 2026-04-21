---
title: "MCP Remote Server (Python)"
description: "Build an MCP remote server with Vouch OIDC bearer token authentication using the Python FastMCP SDK."
weight: 41
params:
  category: "mcp"
  language: "Python"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets AI assistants call tools on remote servers. [FastMCP](https://github.com/PrefectHQ/fastmcp) has built-in support for OAuth-based auth and [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) Protected Resource Metadata.

The server validates access tokens against the Vouch JWKS endpoint and extracts the hardware attestation claim (`hardware_verified`) from the JWT payload. Tools can gate sensitive operations on hardware key attestation.

## Example

**[mcp/remote-server-py](https://github.com/vouch-sh/examples/tree/main/mcp/remote-server-py)** -- Complete working example with FastMCP, bearer token verification, and hardware claim extraction.
