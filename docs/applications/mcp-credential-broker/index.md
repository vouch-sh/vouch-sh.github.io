# MCP Credential Broker

> Build an MCP server that brokers AWS, GitHub, and SSH credentials through Vouch identity verification.

Source: https://vouch.sh/docs/applications/mcp-credential-broker/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

An MCP credential broker lets AI assistants obtain temporary cloud credentials on behalf of authenticated users. The broker validates the caller's Vouch access token, then calls the Vouch credential APIs to get:

- **AWS** -- Temporary STS credentials via `AssumeRoleWithWebIdentity`
- **GitHub** -- Installation access tokens scoped to the user's organization
- **SSH** -- Signed certificates tied to the user's identity

All credentials are short-lived and trace back to a hardware-verified human identity.

## Example

**[mcp/credential-broker](https://github.com/vouch-sh/examples/tree/main/mcp/credential-broker)** -- Complete working example extending the Python MCP remote server with AWS, GitHub, and SSH credential brokering tools.
