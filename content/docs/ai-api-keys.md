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
- **Standards-based** -- both providers accept any standards-compliant OIDC issuer. Vouch publishes [OIDC discovery](https://{{< instance-url >}}/.well-known/openid-configuration) (which advertises the `jwks_uri` where signing keys are served), issues ES256-signed JWTs the provider verifies, and supports [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) token exchange and [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) resource indicators when a provider requires a specific `aud`. The [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523) `jwt-bearer` grant runs at the provider's token endpoint, not Vouch's.

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

In the [Claude Console](https://platform.claude.com/), go to **Settings → Workload identity → Issuers** and select **Create issuer**. Fill in the form:

- **Name** -- a lowercase identifier shown in rules and audit logs. `vouch` is a good default.
- **Issuer URL (iss claim)** -- `https://{{< instance-url >}}`. This must match the `iss` in Vouch-minted tokens exactly.
- **JWKS source** -- **OIDC discovery**. Vouch serves `/.well-known/openid-configuration` at the issuer URL, which advertises the `jwks_uri` Anthropic fetches signing keys from.
- **Discovery base URL** -- leave blank. Vouch publishes discovery at the issuer URL.
- **CA certificate (PEM)** -- leave blank. Vouch terminates TLS with a publicly trusted certificate.
- **Token validation → Enforce single-use tokens (JTI replay protection)** -- leave **on** (the default). Vouch mints a unique `jti` per ID token, so single-use enforcement is safe and prevents replay if a JWT leaks.
- **Token validation → Maximum token lifetime** -- leave at **1 hour** (the default). Vouch ID tokens are short-lived and well under this ceiling; tighten only if policy demands shorter.

### Step 2 -- Create a service account and workspace <span class="role-label">Admin task</span>

Go to **Settings → Service accounts → Create service account** (for example, `coding-agent` or `ci-deploy`) and note its ID (`svac_...`).

`vouch setup anthropic` in Step 4 requires a workspace ID (`wrkspc_...`), and the organization's **Default** workspace does not expose one. If you haven't already, go to **Settings → Workspaces → Create workspace**, add the service account to it, and note the new workspace's ID.

### Step 3 -- Create a federation rule <span class="role-label">Admin task</span>

Back on **Workload identity → Federation rules**, select **Create rule**. The form has four sections:

**Basic info**

- **Rule name** + optional **Description**.
- **Issuer** -- pick the Vouch issuer registered in Step 1.

**Match configuration**

Keep the default **Pattern match** mode. Vouch ID tokens carry the human identity in `sub` as the user's email, and the additional Vouch-specific claims (`hd`, `email_verified`, `hardware_verified`, `hardware_aaguid`) are exact strings or booleans that the Pattern match form's **Additional claim conditions** field handles natively.

- **Subject pattern** -- the user's exact email, e.g. `developer@example.com`. Leave blank if you are matching on a claim condition only (see below).
- **Expected audience** -- `https://{{< instance-url >}}`. Vouch's default `aud` is the issuer URL, so matching that string here means no extra `--audience` flag in Step 4. Anthropic enforces audience even though the field is labeled optional; tokens whose `aud` does not match are rejected with `jwt_audience_mismatch`.
- **Additional claim conditions** -- for a company-wide allow, set claim key `hd` and expected value `example.com` (your Vouch hosted-domain). To additionally require a hardware-backed token, add `hardware_verified` = `true`.

> **Avoid CEL expression mode** for AI provider federation. In CEL the Expected audience field is hidden and not enforced, and the CEL evaluator handles `claims.aud` in ways that don't match Vouch's tokens reliably (`==` against a string-typed `aud` and `in` against a list-typed `aud` have both been observed to fail). Stay on Pattern match — every Vouch claim worth matching on is a string or boolean and works in **Additional claim conditions**.

**Target**

- **Service account** -- the `svac_...` from Step 2.

**Authorization**

- **Workspaces** -- pick the workspace(s) the service account should act in. Leave "Enable in all workspaces" off unless you genuinely want every current and future workspace included.
- **OAuth scope** -- e.g. `workspace:developer`. Sets the scope carried on Anthropic tokens minted by this rule.
- **Token lifetime** -- 10 minutes is the default; choose a value matching the workload (shorter is better).

Two common shapes, both Pattern match, both targeting the service account from Step 2, with `Expected audience: https://{{< instance-url >}}` and scope `workspace:developer`:

- **Single developer** -- Subject pattern: `developer@example.com`.
- **Company-wide allow** -- Subject pattern: blank. Additional claim conditions: `hd` = `example.com`.

Note the rule ID (`fdrl_...`) -- you will need it in Step 4.

### Step 4 -- Get a token <span class="role-label">Developer task</span> {#claude-get-a-token}

> **Requires Vouch v2026.5.4 or later.** The `vouch setup anthropic` / `vouch setup openai` and `vouch credential anthropic` / `vouch credential openai` commands ship in Vouch **v2026.5.4**. On earlier versions, drive the federation exchange directly with the [official Anthropic SDK's](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) `ANTHROPIC_FEDERATION_*` environment variables and `ANTHROPIC_IDENTITY_TOKEN_FILE`, kept current from the AWS-flow ID token returned by [`/v1/credentials/aws/token`](/docs/aws/).

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

No `--audience` flag is needed when the rule's Expected audience matches Vouch's default `aud` (the issuer URL `https://{{< instance-url >}}`). Pass `--audience <value>` only if you pinned the rule to a different audience.

`vouch setup anthropic` persists the federation parameters in `~/.vouch/config.json` and also auto-configures Claude Code's credential helper (see [Use it with coding agents](#use-it-with-coding-agents) below). Pass `--force` to overwrite an existing `apiKeyHelper` if Claude Code is already configured.

`vouch credential anthropic` mints a fresh OIDC ID token from your active Vouch session (no extra YubiKey tap), uses it as the assertion in the [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523) `jwt-bearer` exchange against Anthropic's token endpoint, and caches the returned `sk-ant-oat01-...` until just before it expires — the same caching pattern as [`vouch credential aws`](/docs/cli-reference/) and `vouch credential k8s`. Use it inline:

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

> **Requires Vouch v2026.5.4 or later** (see the [Claude API note above](#claude-get-a-token) for the pre-v2026.5.4 SDK workaround).

Record the OpenAI federation parameters once, then mint tokens with `vouch credential openai`:

```bash
vouch setup openai \
  --identity-provider-id wip_... \
  --service-account-id sa_... \
  --audience https://api.openai.com/v1   # set to whatever audience OpenAI configured

vouch login                              # YubiKey tap, once per session
vouch credential openai                  # prints a short-lived OpenAI access token
```

`vouch setup openai` persists the federation parameters and also auto-writes a Codex provider block at `~/.codex/config.toml` (see [Use it with coding agents](#use-it-with-coding-agents) below). Pass `--force` to switch the top-level Codex `model_provider` away from another provider already in place.

As with Anthropic, ensure no static `OPENAI_API_KEY` is left in the environment to shadow federation.

---

## Use it with coding agents

Once `vouch setup anthropic` / `vouch setup openai` is configured, each CLI points at the matching `vouch credential` command and refreshes through it — no static keys and no token files to manage.

### Claude Code

`vouch setup anthropic` auto-configures Claude Code for you — it merges `~/.claude/settings.json` to set both `apiKeyHelper` (→ `vouch credential anthropic`) and `env.CLAUDE_CODE_API_KEY_HELPER_TTL_MS = "3000000"` (≈50 min, comfortably under the ~1 h provider token lifetime), so each request carries a current short-lived token and Claude Code actually re-invokes the helper before it expires. If Claude Code already has a different `apiKeyHelper`, the command refuses to overwrite — re-run with `--force` to replace it.

Without the TTL env var Claude Code caches the first token until it expires and never refreshes it, so don't set `apiKeyHelper` by hand unless you also set `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` to a value below the minted token's lifetime.

Make sure `ANTHROPIC_API_KEY` is **unset** — it sits above the helper in the credential precedence and will silently shadow it.

### OpenAI Codex CLI

`vouch setup openai` auto-configures Codex for you — it merges `~/.codex/config.toml` to add a `[model_providers.vouch]` block with a refreshing auth command, and flips the top-level `model_provider` to `vouch`:

```toml
model_provider = "vouch"

[model_providers.vouch]
name = "vouch"
base_url = "https://api.openai.com/v1"
wire_api = "chat"

[model_providers.vouch.auth]
command = "vouch"
args = ["credential", "openai"]
timeout_ms = 5000
refresh_interval_ms = 300000   # re-run before the ~1 h token expires
```

Codex invokes the auth command with no stdin and reads the token from stdout — exactly what `vouch credential openai` prints. This `auth` block cannot be combined with `env_key` or `requires_openai_auth` on the same provider.

The `[model_providers.vouch]` block is owned by `vouch setup openai` and overwritten silently — re-run the setup command to pick up a moved `vouch` binary or a new refresh interval. The top-level `model_provider` is shared with your other providers, so changing it away from a non-`vouch` value requires `--force`.

For a one-off invocation you can instead export the token directly, but it is read only at startup and will not refresh mid-session:

```bash
export OPENAI_API_KEY="$(vouch credential openai)"   # one-off; no refresh
codex "refactor this module"
```

Keep any static key out of `~/.codex` and your shell profile.

### Opencode

[Opencode](https://opencode.ai/) reads provider credentials from the environment / `.env` file at startup, from the auth store written by `opencode auth login`, or from substitutions in `opencode.json`. The simplest path is to export the federated token before launching:

```bash
export ANTHROPIC_API_KEY="$(vouch credential anthropic)"
# or, for OpenAI-backed models:
# export OPENAI_API_KEY="$(vouch credential openai)"
opencode
```

You can also pin the substitution in `opencode.json` so the same config works across shells:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "anthropic": {
      "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" }
    }
  }
}
```

Both paths read the token **at launch** — Opencode does not have a command-backed auth provider with proactive refresh like Codex's `model_providers.*.auth`, so a session that outlives the minted token will fail mid-conversation. For long sessions, re-launch Opencode, or write an [Opencode plugin](https://opencode.ai/docs/plugins) that hooks `account.update` to shell out to `vouch credential anthropic` / `vouch credential openai` on demand. Keep any static `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` out of your shell profile so it does not shadow the federated token.

Because each agent now runs as a hardware-verified identity, you get per-developer (or per-agent) usage attribution and an audit trail instead of a shared key.

---

## Audience matching {#audience-matching}

Vouch mints federation assertions with `aud` set to its issuer URL (`https://{{< instance-url >}}`) by default. The simplest setup is to match that exact string on the provider side, which is what the Anthropic walkthrough above does — no `--audience` flag, no override.

Standards-purists may notice this puts the *issuer* in `aud` rather than the *recipient*. That's a deliberate tradeoff: Vouch's federation assertions are single-use (`jti` replay protection), 1-hour-capped, and only ever presented point-to-point to the provider's token endpoint in exchange for the provider's own short-lived access token. The audience-binding scenario this argument is meant to defend against — replay of a bearer token to a different audience — doesn't exist for these assertions.

If a provider requires a specific audience (OpenAI configures the audience server-side when you register the issuer), pin that exact string on both sides:

- On the **rule** (Anthropic) or **identity provider** (OpenAI): set the audience to the value the provider expects.
- On **Vouch**: pass `--audience <same value>` to `vouch setup anthropic` / `vouch setup openai`.

Internally, Vouch mints audience-scoped tokens via [RFC 8707 resource indicators](https://datatracker.ietf.org/doc/html/rfc8707) or [RFC 8693 token exchange](/docs/applications/#service-to-service-m2m-authentication); the `--audience` flag is the surface for that.

## Interactive developers vs. headless agents

The `vouch login` → `vouch credential` flow above is for an **interactive developer**: the YubiKey tap happens at the start of the session, and every minted token traces back to that human.

An **unattended agent** — a CI job or a long-running service — has no human to tap a key, so it cannot use the FIDO2 login flow. Give it its own non-interactive identity instead:

- Use Vouch's [client-credentials (machine-to-machine) flow](/docs/applications/#client-credentials-machine-to-machine) to obtain a Vouch token without a hardware tap, then run the same federation exchange against the provider.
- Issue scoped tokens with an agent-specific `sub` claim and short lifetimes, and target a dedicated service account / federation rule per agent so its access is limited to what it needs.

For agents that act **on behalf of** a human (preserving the human-identity chain), see [Credential Brokering for Agents](/docs/applications/credential-brokering-agents/).

## Troubleshooting

**Provider cannot fetch JWKS / signature validation fails.** The issuer URL must be `https://{{< instance-url >}}` and reachable from the public internet. Confirm `https://{{< instance-url >}}/.well-known/openid-configuration` resolves and that the `jwks_uri` it advertises is reachable.

**Rule does not match.** The provider matches against the **ID token** that `vouch credential anthropic` / `vouch credential openai` mints internally (its `sub` is your email). Note that `vouch credential token` prints the RFC 9068 *access* token whose `sub` is a stable user UUID — useful for debugging Vouch-protected APIs, but not what the federation rule sees. The Authentication events tab in the Claude Console shows the decoded JWT for any failed exchange — use it to compare what Vouch minted against what the rule expected.

**Audience mismatch.** If the provider rejects the audience, mint an audience-scoped token (see [Audience matching](#audience-matching)) so `aud` equals the value the provider expects.

**A static key keeps winning.** `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` take precedence over federation. Unset them everywhere the workload runs (container env, CI secrets, shell profiles).

## Related guides

- [Amazon Bedrock](/docs/bedrock/) -- access Claude models *through AWS* with Vouch STS credentials.
- [AWS](/docs/aws/) -- the OIDC federation pattern Vouch uses for AWS STS.
- [Credential Brokering for Agents](/docs/applications/credential-brokering-agents/) -- scoped tokens for automated agents.
- [Architecture](/docs/architecture/) -- how Vouch issues and signs OIDC tokens.
