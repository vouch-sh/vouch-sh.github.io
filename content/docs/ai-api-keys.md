---
title: "Short-Lived Credentials for the Claude & OpenAI APIs"
linkTitle: "Claude & OpenAI APIs"
description: "Use Vouch as the OIDC issuer for Anthropic and OpenAI Workload Identity Federation, so local development and one-off scripts don't need a long-lived sk-ant- or sk- key."
weight: 17
subtitle: "Stop pasting sk-ant- and sk- keys into your shell for local development"
params:
  docsGroup: infra
---

Both [Anthropic](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) and [OpenAI](https://developers.openai.com/api/docs/guides/workload-identity-federation/aws) support **Workload Identity Federation (WIF)**: a workload presents a short-lived OIDC token from an issuer you operate, and the provider exchanges it for a minutes-long access token. WIF is workload-shaped, not workforce-shaped — there is no console login flow for an end user. It's a way to authenticate *something running somewhere* without giving it a static API key.

In production, that "something running somewhere" is usually a GitHub Actions runner, an AWS Lambda, or a GKE pod — and the right issuer is the native OIDC token those platforms already mint (see [Anthropic's](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) and [OpenAI's](https://developers.openai.com/api/docs/guides/workload-identity-federation/aws) WIF docs).

This page covers a narrower case: **local development and one-off scripts on a developer laptop**, where the alternative is pasting an `sk-ant-...` or `sk-...` key into a `.env` file and forgetting about it. Vouch is a standards-compliant OIDC issuer, so you can point Anthropic and OpenAI at it the same way they'd point at any other IdP, and exchange a Vouch-issued JWT for a short-lived provider token — no static key on disk.

```
vouch login → Vouch OIDC JWT → jwt-bearer exchange → short-lived provider token → Claude / OpenAI API
```

> Vouch must be reachable from the public internet so the provider can fetch the JWKS endpoint to validate token signatures. See [Architecture](/docs/architecture/) for network requirements.

---

## Claude API (Anthropic)

Anthropic's WIF accepts any standards-compliant OIDC issuer. You register Vouch as a **federation issuer**, point a **federation rule** at a **service account**, and exchange Vouch tokens for short-lived `sk-ant-oat01-...` tokens.

### Step 1 -- Register Vouch as a federation issuer <span class="role-label">Admin task</span>

In the [Claude Console](https://platform.claude.com/), go to **Settings → Workload identity → Issuers** and select **Create issuer**. Fill in the form:

- **Name** -- a lowercase identifier shown in rules and audit logs. `vouch` is a good default.
- **Issuer URL (iss claim)** -- `https://{{< instance-url >}}`. This must match the `iss` in Vouch-minted tokens exactly.
- **JWKS source** -- **OIDC discovery**. Vouch serves `/.well-known/openid-configuration` at the issuer URL, which advertises the `jwks_uri` Anthropic fetches signing keys from.
- **Discovery base URL** -- leave blank. Vouch publishes discovery at the issuer URL.
- **CA certificate (PEM)** -- leave blank. Vouch terminates TLS with a publicly trusted certificate.
- **Token validation → Enforce single-use tokens (JTI replay protection)** -- leave **on** (the default). Vouch mints a unique `jti` per ID token.
- **Token validation → Maximum token lifetime** -- leave at **1 hour** (the default). Vouch ID tokens are short-lived and well under this ceiling.

### Step 2 -- Create a service account and workspace <span class="role-label">Admin task</span>

Go to **Settings → Service accounts → Create service account** (for example, `local-dev`) and note its ID (`svac_...`).

`vouch setup anthropic` in Step 4 requires a workspace ID (`wrkspc_...`), and the organization's **Default** workspace does not expose one. Go to **Settings → Workspaces → Create workspace**, add the service account to it, and note the new workspace's ID.

### Step 3 -- Create a federation rule <span class="role-label">Admin task</span>

Back on **Workload identity → Federation rules**, select **Create rule**. The form has four sections:

**Basic info**

- **Rule name** + optional **Description**.
- **Issuer** -- pick the Vouch issuer registered in Step 1.

**Match configuration**

Keep the default **Pattern match** mode. Vouch tokens carry the logged-in identity in `sub` (the user's email); additional claims (`hd`, `email_verified`) are exact strings or booleans that **Additional claim conditions** handles natively.

- **Subject pattern** -- the developer's email, e.g. `developer@example.com`. Leave blank if you are matching on a claim condition only.
- **Expected audience** -- `https://{{< instance-url >}}`. Vouch's default `aud` is the issuer URL, so matching that string here means no extra `--audience` flag in Step 4. Anthropic enforces audience even though the field is labeled optional; mismatches are rejected with `jwt_audience_mismatch`.
- **Additional claim conditions** -- for a company-wide allow, set claim key `hd` and expected value `example.com` (your Vouch hosted-domain).

> **Avoid CEL expression mode.** In CEL the Expected audience field is hidden and not enforced, and the CEL evaluator handles `claims.aud` in ways that don't match Vouch's tokens reliably (`==` against a string-typed `aud` and `in` against a list-typed `aud` have both been observed to fail). Stay on Pattern match.

**Target**

- **Service account** -- the `svac_...` from Step 2.

**Authorization**

- **Workspaces** -- pick the workspace from Step 2. Leave "Enable in all workspaces" off.
- **OAuth scope** -- e.g. `workspace:developer`.
- **Token lifetime** -- 10 minutes is the default; shorter is better.

Note the rule ID (`fdrl_...`) -- you will need it in Step 4.

### Step 4 -- Get a token <span class="role-label">Developer task</span> {#claude-get-a-token}

> **Requires Vouch v2026.5.4 or later.** On earlier versions, drive the exchange directly with the [official Anthropic SDK's](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) `ANTHROPIC_FEDERATION_*` environment variables and `ANTHROPIC_IDENTITY_TOKEN_FILE`, kept current from the AWS-flow ID token returned by [`/v1/credentials/aws/token`](/docs/aws/).

Record the federation parameters once, then mint short-lived tokens on demand:

```bash
vouch setup anthropic \
  --federation-rule-id fdrl_... \
  --organization-id 00000000-0000-0000-0000-000000000000 \
  --service-account-id svac_... \
  --workspace-id wrkspc_...

vouch login                       # once per session
vouch credential anthropic        # prints a short-lived sk-ant-oat01-... token
```

No `--audience` flag is needed when the rule's Expected audience matches Vouch's default `aud` (the issuer URL). Pass `--audience <value>` only if you pinned the rule to a different audience.

`vouch setup anthropic` persists the federation parameters in `~/.vouch/config.json` so subsequent `vouch credential anthropic` invocations know which rule, service account, and workspace to use.

`vouch credential anthropic` mints a fresh OIDC ID token from your active Vouch session, exchanges it via the [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523) `jwt-bearer` grant against Anthropic's token endpoint, and caches the returned `sk-ant-oat01-...` until just before it expires — the same caching pattern as [`vouch credential aws`](/docs/cli-reference/) and `vouch credential k8s`. Use it inline:

```bash
curl -sS https://api.anthropic.com/v1/messages \
  -H "authorization: Bearer $(vouch credential anthropic)" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":1024,"messages":[{"role":"user","content":"Hello, Claude"}]}'
```

For scripts and SDKs that read `ANTHROPIC_AUTH_TOKEN` from the environment, two helpers wrap the credential call:

```bash
eval "$(vouch env --type anthropic)"        # exports ANTHROPIC_AUTH_TOKEN in the current shell
vouch exec --type anthropic -- python app.py  # runs a command with ANTHROPIC_AUTH_TOKEN set
```

Use `ANTHROPIC_AUTH_TOKEN`, not `ANTHROPIC_API_KEY` — the API-key variable is reserved for static `sk-ant-api...` keys and rejects federated `sk-ant-oat01-...` tokens. Make sure no `ANTHROPIC_API_KEY` is set in your shell profile or it will silently shadow the federated token.

---

## OpenAI API

OpenAI's WIF similarly exchanges an OIDC JWT for a short-lived OpenAI access token.

### Step 1 -- Register Vouch as the OIDC issuer <span class="role-label">Admin task</span>

In your OpenAI organization's workload identity settings, configure:

| Field | Value |
|---|---|
| Issuer URL | `https://{{< instance-url >}}` |
| Audience | The audience your workload requests (see [Audience matching](#audience-matching) below) |
| Subject mapping | Map the Vouch `sub` claim to the OpenAI service account |

### Step 2 -- Get a token <span class="role-label">Developer task</span>

> **Requires Vouch v2026.5.4 or later** (see the [Claude API note above](#claude-get-a-token) for the pre-v2026.5.4 SDK workaround).

```bash
vouch setup openai \
  --identity-provider-id wip_... \
  --service-account-id sa_... \
  --audience https://api.openai.com/v1   # set to whatever audience OpenAI configured

vouch login
vouch credential openai                  # prints a short-lived OpenAI access token
```

Use it inline the same way:

```bash
curl -sS https://api.openai.com/v1/responses \
  -H "authorization: Bearer $(vouch credential openai)" \
  -H "content-type: application/json" \
  -d '{"model":"gpt-5","input":"hello"}'
```

Ensure no static `OPENAI_API_KEY` is left in your shell profile to shadow federation.

---

## Audience matching {#audience-matching}

Vouch mints federation assertions with `aud` set to its issuer URL (`https://{{< instance-url >}}`) by default. The simplest setup is to match that exact string on the provider side, which is what the Anthropic walkthrough above does — no `--audience` flag, no override.

If a provider requires a specific audience (OpenAI configures the audience server-side when you register the issuer), pin that exact string on both sides:

- On the **rule** (Anthropic) or **identity provider** (OpenAI): set the audience to the value the provider expects.
- On **Vouch**: pass `--audience <same value>` to `vouch setup anthropic` / `vouch setup openai`.

Internally, Vouch mints audience-scoped tokens via [RFC 8707 resource indicators](https://datatracker.ietf.org/doc/html/rfc8707) or [RFC 8693 token exchange](/docs/applications/#service-to-service-m2m-authentication); the `--audience` flag is the surface for that.

## Beyond local development

For workloads that run somewhere other than a developer laptop, prefer the native OIDC issuer of the platform they run on — GitHub Actions, AWS, GCP, and Kubernetes all mint their own workload tokens that Anthropic and OpenAI accept directly. Vouch is the right issuer when the workload is *you*, at a terminal, running a script.

For an unattended job that does need to federate through Vouch (a scheduled task on a server you control, for example), use Vouch's [client-credentials flow](/docs/applications/#client-credentials-machine-to-machine) to obtain a Vouch token without an interactive login, then run the same federation exchange.

## Troubleshooting

**Provider cannot fetch JWKS / signature validation fails.** The issuer URL must be `https://{{< instance-url >}}` and reachable from the public internet. Confirm `https://{{< instance-url >}}/.well-known/openid-configuration` resolves and that the `jwks_uri` it advertises is reachable.

**Rule does not match.** The provider matches against the **ID token** that `vouch credential anthropic` / `vouch credential openai` mints internally (its `sub` is your email). Note that `vouch credential token` prints the RFC 9068 *access* token whose `sub` is a stable user UUID — useful for debugging Vouch-protected APIs, but not what the federation rule sees. The Authentication events tab in the Claude Console shows the decoded JWT for any failed exchange.

**Audience mismatch.** If the provider rejects the audience, mint an audience-scoped token (see [Audience matching](#audience-matching)) so `aud` equals the value the provider expects.

**A static key keeps winning.** `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` take precedence over federation. Unset them everywhere the workload runs (shell profile, `.env`, CI secrets).

## Related guides

- [Amazon Bedrock](/docs/bedrock/) -- access Claude models *through AWS* with Vouch STS credentials.
- [AWS](/docs/aws/) -- the OIDC federation pattern Vouch uses for AWS STS.
- [Architecture](/docs/architecture/) -- how Vouch issues and signs OIDC tokens.
