---
title: "AWS Access Through IAM Identity Center"
linkTitle: "AWS Identity Center"
description: "Issue short-lived AWS credentials through IAM Identity Center permission sets, backed by a YubiKey -- no per-account OIDC provider and no role chaining."
weight: 4
subtitle: "Reach any assigned account and permission set directly, with a single hardware-backed login"
sitemap:
  priority: 0.7
params:
  docsGroup: infra
---

Vouch can issue AWS credentials **through AWS IAM Identity Center** (formerly AWS SSO) instead of through a per-account OIDC provider. A developer authenticates once with their YubiKey, and Vouch exchanges that hardware-backed session for an Identity Center token that reaches **every account and permission set the developer is assigned** -- directly, with no `AssumeRole` chaining and no per-role IAM trust policy.

This is a third AWS access pattern that sits alongside the two federation models covered elsewhere. All three run behind the **same** `vouch credential aws` and `vouch setup aws` commands; the mechanism is selected by your configuration, so adopting Identity Center only *adds config* -- your commands and existing profiles never change.

| Pattern | Guide | How access is granted |
|---|---|---|
| **Single account** | [AWS](/docs/aws/) | STS `AssumeRoleWithWebIdentity` into one role you deploy |
| **Multi-account (role chaining)** | [Multi-Account AWS](/docs/aws-multi-account/) | Federate into a hub role, then `AssumeRole` into spoke roles |
| **Identity Center** (this page) | -- | Identity Center permission-set assignments, reached via the SSO Portal |

**When to use this pattern:** you already run AWS IAM Identity Center with an **organization instance**, manage access as permission-set assignments, and want that existing access model to drive Vouch-issued credentials without deploying Vouch OIDC roles in every account.

> **Requires an organization instance of Identity Center.** The trusted-token-issuer exchange this page describes is only available on Identity Center **organization** instances, not account instances.

## How it works

1. The developer runs `vouch login` and authenticates with their YubiKey.
2. When AWS credentials are needed, Vouch obtains an Identity Center **bearer token**. If the SSO session is configured with a trusted token issuer (Step 1), Vouch exchanges a short-lived **RS256** OIDC token from the Vouch server for an Identity Center token via `sso-oidc:CreateTokenWithIAM` (no separate sign-in). Otherwise it falls back to the device token cached by `vouch aws login`.
3. Vouch calls the Identity Center **SSO Portal** `GetRoleCredentials` API for the chosen account and permission-set role, and returns temporary credentials (access key, secret key, session token) as AWS `credential_process` JSON.
4. The developer's AWS CLI, SDK, or Terraform session uses these credentials transparently.

Access is governed entirely by the user's Identity Center permission-set assignments -- there is no role trust policy for Vouch to maintain in each account.

---

## Step 1 -- Register Vouch as a trusted token issuer

<span class="role-label">Admin task</span>

Do this once, in the AWS account that holds your Identity Center **management role** (the role Vouch federates into). It wires Identity Center to accept Vouch-issued tokens.

1. **Register a trusted token issuer.** In the Identity Center console (or via the `sso-admin` API), add Vouch as a trusted token issuer with the issuer URL `https://{{< instance-url >}}`. Identity Center fetches Vouch's JWKS from `https://{{< instance-url >}}/oauth/jwks` to verify token signatures.
2. **Create a customer-managed OAuth 2.0 application.** Add a customer-managed application in Identity Center and note its **application ARN** (looks like `arn:aws:sso::111111111111:application/ssoins-xxxx/apl-xxxx`). You will put this in `identity_center_application_arn`.
3. **Set the application's Aud claim.** On the application's trusted-token-issuer configuration, set the expected **Aud** (audience) claim to a value you choose, e.g. `vouch-identity-center`. You will put the same value in `identity_center_audience`; Vouch requests it as the `audience` on the RS256 token so Identity Center accepts the exchange.
4. **Enable the portal scope.** Grant the application the **`sso:account:access`** scope so the exchanged token can drive the SSO Portal (`ListAccounts` / `ListAccountRoles` / `GetRoleCredentials`).
5. **Allow the exchange.** In the application's resource (authentication) policy, grant your Vouch **management role** permission to call **`sso-oauth:CreateTokenWithIAM`**. This is the SigV4 caller Vouch uses to perform the exchange.

<div class="checkpoint">
<p><strong>You are done with the AWS-side setup when...</strong></p>
<ul>
  <li>Vouch is registered as a trusted token issuer pointing at <code>https://{{< instance-url >}}</code>.</li>
  <li>A customer-managed application exists, with its Aud claim set and the <code>sso:account:access</code> scope enabled.</li>
  <li>The management role may call <code>sso-oauth:CreateTokenWithIAM</code> on that application.</li>
  <li>You have written down the application ARN and the Aud value -- developers need both in Step 2.</li>
</ul>
</div>

---

## Step 2 -- Configure the SSO session

<span class="role-label">Developer task</span>

Identity Center connection details (start URL, region, scopes) are read from the `[sso-session]` block in `~/.aws/config`, exactly as for `vouch aws login`. Vouch stores the two extra Identity Center values in its own config file at `$XDG_CONFIG_HOME/vouch/config.json` (`~/.config/vouch/config.json` by default), keyed by SSO session name:

```json
{
  "aws": {
    "sso_sessions": {
      "my-sso": {
        "management_role": "arn:aws:iam::111111111111:role/VouchManagement",
        "identity_center_application_arn": "arn:aws:sso::111111111111:application/ssoins-xxxx/apl-xxxx",
        "identity_center_audience": "vouch-identity-center"
      }
    }
  }
}
```

- `management_role` -- the role Vouch federates into via `AssumeRoleWithWebIdentity`; it must be allowed to call `sso-oauth:CreateTokenWithIAM` (Step 1).
- `identity_center_application_arn` -- the customer-managed application ARN from Step 1.
- `identity_center_audience` -- the Aud value you configured on that application.

Both `identity_center_application_arn` and `identity_center_audience` must be set together. When they are, a single `vouch login` is enough -- Vouch performs the token exchange for you. If they are **omitted**, the Identity Center path still works, but you must run `vouch aws login` first so Vouch can use the cached device token instead.

---

## Step 3 -- Set up AWS profiles

<span class="role-label">Developer task</span>

`vouch setup aws` writes profiles into `~/.aws/config` whose `credential_process` calls `vouch credential aws`. With Identity Center configured on the session, it has three modes.

### Interactive -- one account

Run `vouch setup aws` with no `--role` and no `--discover`. Vouch prompts you to pick an account and a permission-set role from those you are assigned, then writes a single profile:

```bash
vouch setup aws
# → Select an AWS account: …
# → Select a role: …
```

### Discover everything

Write a profile for **every** account and permission set you can access:

```bash
vouch setup aws --discover --region us-east-1
```

When the session is configured for Identity Center, `--discover` uses the SSO Portal and writes one `vouch-sso-<account>-<role>` profile per assignment. (Without Identity Center config, `--discover` falls back to STS role-chaining discovery -- see [Multi-Account AWS](/docs/aws-multi-account/).) Pass `--prefix <NAME>` to change the profile-name prefix.

### By hand

Both modes above write the same shape of profile, so you can also add or edit one directly. In Identity Center mode the `credential_process` invokes `vouch credential aws` with `--account` and `--role`:

```ini
[profile prod-admin]
credential_process = vouch credential aws --sso-session "my-sso" --account 111111111111 --role "AdministratorAccess"
region = us-east-1
```

> **Note:** In Identity Center mode, `--role` is the **permission-set name** (e.g. `AdministratorAccess`), not a role ARN. Both `--account` and `--role` are required, because `credential_process` runs non-interactively. (Without `--account`, `vouch credential aws --role <ARN>` uses the STS web-identity path instead.)

---

## Step 4 -- Test

<span class="role-label">Developer task</span>

```bash
# Authenticate (a single hardware-backed login when the trusted token issuer is configured)
vouch login

# If you did NOT configure identity_center_application_arn / _audience, sign in to Identity Center first:
# vouch aws login

# Check your identity
aws sts get-caller-identity --profile prod-admin
```

<div class="checkpoint">
<p><strong>You are done when...</strong></p>
<ul>
  <li><code>aws sts get-caller-identity --profile prod-admin</code> returns an assumed-role identity in the expected account.</li>
  <li>A real AWS command (e.g. <code>aws s3 ls --profile prod-admin</code>) succeeds with no static credentials in <code>~/.aws/credentials</code>.</li>
</ul>
</div>

---

## Session tags and attribution

The Identity Center path carries the same [session tags](/docs/aws/#session-tags) as the STS web-identity path -- `vouch:Email`, `vouch:Domain`, and, when an AI coding agent is detected, `vouch:AccessType=ai` and `vouch:Agent=<name>`. Use them in IAM policy conditions via `aws:PrincipalTag` for attribute-based access control and CloudTrail attribution.

---

## Troubleshooting

### `CreateTokenWithIAM` is denied

- Confirm the Vouch **management role** is granted `sso-oauth:CreateTokenWithIAM` in the customer-managed application's resource policy (Step 1.5).
- Confirm `management_role` in `config.json` matches that role and that Vouch can federate into it (its trust policy trusts the Vouch OIDC provider, as in [AWS / Step 3](/docs/aws/#step-3--deploy-a-vouch-role)).

### The token is rejected (invalid audience)

- The value in `identity_center_audience` must exactly match the **Aud** claim configured on the customer-managed application (Step 1.3).

### No accounts or roles are listed

- Ensure the customer-managed application has the **`sso:account:access`** scope enabled (Step 1.4).
- Verify the developer has permission-set assignments in Identity Center.

### `SSO session expired or missing`

- You have not configured the trusted-token-issuer fields, so Vouch is using the device-token fallback. Run `vouch aws login`, or add `identity_center_application_arn` and `identity_center_audience` to the session (Step 2).

---

## Related guides

- [AWS](/docs/aws/) -- Single-account OIDC federation with STS `AssumeRoleWithWebIdentity`.
- [Multi-Account AWS Strategy](/docs/aws-multi-account/) -- Role chaining across an AWS Organization.
- [CLI Reference](/docs/cli-reference/) -- Full flags for `vouch credential aws` and `vouch setup aws`.

