---
title: "Replace AI Provider API Keys with Short-Lived Credentials"
linkTitle: "Claude & OpenAI APIs"
description: "Authenticate to the Claude API and OpenAI API with short-lived, hardware-backed tokens via Workload Identity Federation — no more long-lived sk-ant- or sk- keys."
weight: 17
subtitle: "Use Vouch as the OIDC issuer for Anthropic and OpenAI Workload Identity Federation"
params:
  docsGroup: infra
---

AI provider API keys are long-lived bearer secrets. An `sk-ant-...` or `sk-...` key copied into a CI secret, an agent's environment, or a developer's `.env` file never expires and grants full access until someone notices and rotates it. Both [Anthropic](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) and [OpenAI](https://developers.openai.com/api/docs/guides/workload-identity-federation/aws) now support **Workload Identity Federation (WIF)**: instead of a static key, a workload presents a short-lived OIDC token from an identity provider you operate, and the provider exchanges it for an access token that expires in minutes.

Vouch is a standards-compliant, [FAPI 2.0–certified](/docs/architecture/) OIDC issuer, so it can be that identity provider. After a single FIDO2 verification, developers and agents exchange a Vouch-issued token for a short-lived Claude or OpenAI credential — no static keys to mint, store, rotate, or leak.

```
YubiKey tap → FIDO2 → Vouch OIDC JWT → jwt-bearer exchange → short-lived provider token → Claude / OpenAI API
```

## Why this matters

- **Every API call has a verified identity** -- no shared keys or service-account secrets for human users.
- **No static credentials to leak** -- tokens expire in minutes instead of never; there is nothing long-lived to exfiltrate from CI, an agent, or a laptop.
- **Per-user and per-agent attribution** -- usage and cost map back to the hardware-verified identity that the token was minted for.
- **Standards-based** -- both providers accept any standards-compliant OIDC issuer. Vouch publishes [OIDC discovery](https://{{< instance-url >}}/.well-known/openid-configuration) and [JWKS](https://{{< instance-url >}}/.well-known/jwks.json), and implements [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523) (JWT bearer), [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) (token exchange), and [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) (audience-restricted tokens).

## How it works

1. The developer runs `vouch login` and authenticates with their YubiKey.
2. Vouch issues a short-lived OIDC ID token signed with **ES256**. The identity is carried in the `sub` claim (the developer's email).
3. The workload presents that JWT to the AI provider's token endpoint using the **RFC 7523 `jwt-bearer`** grant.
4. The provider validates the signature against the Vouch JWKS endpoint, checks `iss` / `exp` / audience and the claims against the federation rule you configured, and returns a short-lived access token.
5. The workload calls the API with that token and refreshes it before it expires.

> Vouch must be reachable from the public internet so the provider can fetch the JWKS endpoint to validate token signatures. See [Architecture](/docs/architecture/) for network requirements.

---

## Claude API (Anthropic)

Anthropic's Workload Identity Federation accepts any standards-compliant OIDC issuer. You register Vouch as a **federation issuer**, point a **federation rule** at a **service account**, and your workloads exchange Vouch tokens for short-lived `sk-ant-oat01-...` tokens.

### Step 1 -- Register Vouch as a federation issuer <span class="role-label">Admin task</span>

In the [Claude Console](https://platform.claude.com/), go to **Settings → Workload identity → Issuers** and select **Create issuer**. Choose the **generic OIDC** preset and set:

| Field | Value |
|---|---|
| Issuer URL | `https://{{< instance-url >}}` |
| JWKS source | `discovery` -- Vouch serves `/.well-known/openid-configuration` at the issuer URL |

### Step 2 -- Create a service account <span class="role-label">Admin task</span>

Go to **Settings → Service accounts → Create service account** (for example, `coding-agent` or `ci-deploy`). Add it to each workspace it should act in, and note its ID (`svac_...`).

### Step 3 -- Create a federation rule <span class="role-label">Admin task</span>

Back on **Workload identity → Federation rules**, select **Create rule**, choose the Vouch issuer, and match on the Vouch token's `sub` claim (the user or agent email). Target the service account from Step 2 and set a token lifetime. Note the rule ID (`fdrl_...`).

A rule matching a single developer's email looks like:

```
Match:   subject_prefix = "developer@example.com"
Target:  svac_...   (service account)
Scope:   workspace:developer
```

### Step 4 -- Exchange the token <span class="role-label">Developer task</span>

After `vouch login`, print the raw session JWT with [`vouch credential token`](/docs/cli-reference/#vouch-credential-token) and exchange it at Anthropic's token endpoint:

```bash
JWT=$(vouch credential token)

ACCESS_TOKEN=$(curl -sS https://api.anthropic.com/v1/oauth/token \
  -H "content-type: application/json" \
  -d '{
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "assertion": "'"$JWT"'",
    "federation_rule_id": "fdrl_...",
    "organization_id": "00000000-0000-0000-0000-000000000000",
    "service_account_id": "svac_...",
    "workspace_id": "wrkspc_..."
  }' | jq -r .access_token)

curl -sS https://api.anthropic.com/v1/messages \
  -H "authorization: Bearer $ACCESS_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":1024,"messages":[{"role":"user","content":"Hello, Claude"}]}'
```

In production, the official Anthropic SDKs run this exchange and refresh loop for you when you set the federation environment variables (`ANTHROPIC_FEDERATION_RULE_ID`, `ANTHROPIC_ORGANIZATION_ID`, `ANTHROPIC_SERVICE_ACCOUNT_ID`, `ANTHROPIC_WORKSPACE_ID`) and point `ANTHROPIC_IDENTITY_TOKEN_FILE` at a file kept populated with a fresh Vouch token. Make sure `ANTHROPIC_API_KEY` is unset — it takes precedence over federation and will silently shadow it.

---

## OpenAI API

OpenAI's Workload Identity Federation similarly exchanges an OIDC JWT for a short-lived OpenAI access token, so Vouch slots in as the issuer.

### Step 1 -- Register Vouch as the OIDC issuer <span class="role-label">Admin task</span>

In your OpenAI organization's workload identity settings, configure:

| Field | Value |
|---|---|
| Issuer URL | `https://{{< instance-url >}}` |
| Audience | The audience your workload requests (see [Audience matching](#audience-matching) below) |
| Subject mapping | Map the Vouch `sub` claim (user/agent email) to the OpenAI service account |

### Step 2 -- Exchange the token <span class="role-label">Developer task</span>

Obtain the Vouch JWT with `vouch credential token` and let the OpenAI SDK exchange it for a short-lived OpenAI access token, which it then sends as the bearer credential on each request. As with Anthropic, ensure no static `OPENAI_API_KEY` is left in the environment to shadow federation.

---

## Use it with coding agents

Both flagship coding CLIs read their provider credentials from the environment, so they pick up federated tokens the same way any SDK does.

### Claude Code

Claude Code is built on the Anthropic SDK and honors the same credential precedence. The most self-contained option is Claude Code's `apiKeyHelper` setting — a script that performs the exchange and prints the resulting token, which Claude Code re-runs to refresh:

```bash
#!/usr/bin/env bash
# ~/.vouch/anthropic-token.sh — set as apiKeyHelper in Claude Code settings
JWT=$(vouch credential token)
curl -sS https://api.anthropic.com/v1/oauth/token \
  -H "content-type: application/json" \
  -d '{"grant_type":"urn:ietf:params:oauth:grant-type:jwt-bearer","assertion":"'"$JWT"'","federation_rule_id":"fdrl_...","organization_id":"...","service_account_id":"svac_...","workspace_id":"wrkspc_..."}' \
  | jq -r .access_token
```

Alternatively, set the `ANTHROPIC_FEDERATION_*` environment variables and `ANTHROPIC_IDENTITY_TOKEN_FILE` (kept current with `vouch credential token`), and let the bundled SDK handle the exchange. Either way, confirm `ANTHROPIC_API_KEY` is unset so it does not shadow federation.

### OpenAI Codex CLI

Codex CLI authenticates with `OPENAI_API_KEY`. Wrap your session so the variable holds a freshly exchanged OpenAI access token rather than a static key:

```bash
export OPENAI_API_KEY=$(vouch-openai-token)   # script that exchanges the Vouch JWT
codex "refactor this module"
```

Re-run the exchange (or use a wrapper that refreshes on expiry) for long sessions, since the federated token is short-lived. Keep the static key out of `~/.codex` and shell profiles.

Because each agent now runs as a hardware-verified identity, you get per-developer (or per-agent) usage attribution and an audit trail instead of a shared key.

---

## Audience matching {#audience-matching}

Vouch's default token `aud` claim is the Vouch server URL. Anthropic audience matching is optional — a rule can match on `sub` alone — so the default works for Claude. When a provider requires the token's `aud` to be a specific value, mint an audience-scoped token using Vouch's [RFC 8707 resource indicators](https://datatracker.ietf.org/doc/html/rfc8707) or [RFC 8693 token exchange](/docs/applications/#service-to-service-m2m-authentication), and register that exact value as the expected audience on the provider side.

## Agents and delegation

For automated agents that call these APIs on behalf of a user, issue scoped tokens with an agent-specific `sub` claim and short lifetimes, and target a dedicated service account / federation rule per agent. This keeps the human-identity chain intact while limiting each agent to only what it needs. See [Credential Brokering for Agents](/docs/applications/credential-brokering-agents/).

## Troubleshooting

**Provider cannot fetch JWKS / signature validation fails.** The issuer URL must be `https://{{< instance-url >}}` and reachable from the public internet. Confirm `https://{{< instance-url >}}/.well-known/openid-configuration` and `/.well-known/jwks.json` resolve.

**Rule does not match.** Decode your Vouch token (`vouch credential token | cut -d. -f2 | base64 -d`) and confirm the `sub`/claims satisfy the federation rule's match conditions exactly.

**Audience mismatch.** If the provider rejects the audience, mint an audience-scoped token (see [Audience matching](#audience-matching)) so `aud` equals the value the provider expects.

**A static key keeps winning.** `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` take precedence over federation. Unset them everywhere the workload runs (container env, CI secrets, shell profiles).

## Related guides

- [Amazon Bedrock](/docs/bedrock/) -- access Claude models *through AWS* with Vouch STS credentials.
- [AWS](/docs/aws/) -- the OIDC federation pattern Vouch uses for AWS STS.
- [Credential Brokering for Agents](/docs/applications/credential-brokering-agents/) -- scoped tokens for automated agents.
- [Architecture](/docs/architecture/) -- how Vouch issues and signs OIDC tokens.
