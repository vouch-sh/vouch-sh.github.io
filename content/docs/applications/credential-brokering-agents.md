---
title: "Credential Brokering Agents"
description: "Build CLI agents that authenticate with Vouch device flow and broker AWS, GitHub, and SSH credentials."
weight: 51
params:
  category: "agent"
  language: "Python"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

CLI agents and automation scripts can authenticate with Vouch using the [Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628) (no browser redirect needed), then use the Vouch credential brokering APIs to obtain temporary AWS credentials, GitHub tokens, or SSH certificates -- all tied to the user's hardware-backed identity.

## Examples

Working examples are available in the examples repository:

- **[native/python-agent-aws](https://github.com/vouch-sh/examples/tree/main/native/python-agent-aws)** -- CLI agent that brokers AWS STS credentials
- **[native/python-agent-github](https://github.com/vouch-sh/examples/tree/main/native/python-agent-github)** -- CLI agent that brokers GitHub installation tokens
- **[native/python-agent-multi](https://github.com/vouch-sh/examples/tree/main/native/python-agent-multi)** -- CLI agent that brokers AWS, GitHub, and SSH credentials
