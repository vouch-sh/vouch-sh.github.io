# A2A Agent (Python)

> Build an Agent-to-Agent (A2A) protocol agent secured with Vouch OIDC bearer token authentication.

Source: https://vouch.sh/docs/applications/a2a-agent/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

The [Agent-to-Agent (A2A)](https://a2a-protocol.org/) protocol defines how AI agents discover and communicate with each other. An A2A agent secured with Vouch requires bearer token authentication -- the Agent Card advertises the Vouch OIDC security scheme, and the agent validates access tokens against the Vouch JWKS endpoint.

## Example

**[a2a/python-agent](https://github.com/vouch-sh/examples/tree/main/a2a/python-agent)** -- Complete working example with A2A Python SDK, Agent Card with Vouch security scheme, and bearer token verification middleware.
