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

### Step 4 -- Get a token <span class="role-label">Developer task</span>

Record the federation parameters once, then mint short-lived tokens on demand:

```bash
vouch setup anthropic \
  --federation-rule-id fdrl_... \
  --organization-id 00000000-0000-0000-0000-000000000000 \
  --service-account-id svac_... \
  --workspace-id wrkspc_...

vouch login                       # YubiKey tap, once per session
vouch credential anthropic        # prints a short-lived sk-ant-oat01-... token
```

`vouch credential anthropic` runs the [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523) `jwt-bearer` exchange against Anthropic's token endpoint with your session token and caches the result, refreshing it as needed — the same pattern as [`vouch credential aws`](/docs/cli-reference/) and `vouch credential k8s`. Use it inline:

```bash
curl -sS https://api.anthropic.com/v1/messages \
  -H "authorization: Bearer $(vouch credential anthropic)" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":1024,"messages":[{"role":"user","content":"Hello, Claude"}]}'
```

Under the hood this is a standard token exchange, so non-Vouch SDK workloads can federate the same way using the official Anthropic SDK's `ANTHROPIC_FEDERATION_*` environment variables and `ANTHROPIC_IDENTITY_TOKEN_FILE`. Either way, ensure `ANTHROPIC_API_KEY` is unset — it takes precedence over federation and will silently shadow it.

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

### Step 2 -- Get a token <span class="role-label">Developer task</span>

Record the OpenAI federation parameters once, then mint tokens with `vouch credential openai`:

```bash
vouch setup openai --audience https://api.openai.com/v1   # plus org/service-account as prompted
vouch credential openai                                   # prints a short-lived OpenAI access token
```

As with Anthropic, ensure no static `OPENAI_API_KEY` is left in the environment to shadow federation.

---

## Use it with coding agents

Once `vouch setup anthropic` / `vouch setup openai` is configured, each CLI points at the matching `vouch credential` command and refreshes through it — no static keys and no token files to manage.

### Claude Code

Set Claude Code's `apiKeyHelper` to `vouch credential anthropic`. Claude Code re-runs the helper to refresh, so each request carries a current short-lived token:

```bash
claude config set apiKeyHelper "vouch credential anthropic"
```

Make sure `ANTHROPIC_API_KEY` is **unset** — it sits above the helper in the credential precedence and will silently shadow it.

### OpenAI Codex CLI

Codex CLI authenticates with `OPENAI_API_KEY`. Export it from `vouch credential openai` rather than a static key:

```bash
export OPENAI_API_KEY="$(vouch credential openai)"
codex "refactor this module"
```

Re-export before starting a long session, since the federated token is short-lived, and keep any static key out of `~/.codex` and your shell profile.

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
