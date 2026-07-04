---
title: "Replace AWS Access Keys with Short-Lived Credentials"
linkTitle: "AWS"
description: "Stop distributing long-lived AWS access keys. Use OIDC federation to get temporary STS credentials backed by a YubiKey."
weight: 2
subtitle: "Configure AWS IAM to trust Vouch as an OIDC identity provider"
sitemap:
  priority: 0.8
params:
  docsGroup: infra
---

Vouch eliminates static AWS access keys. You configure AWS to trust Vouch as an OIDC identity provider, and developers get temporary STS credentials -- valid for up to 1 hour -- after authenticating with their YubiKey. Every API call is tied to a verified human identity in CloudTrail.

## How it works

`vouch login` authenticates the developer with their YubiKey, and the Vouch server issues a short-lived OIDC ID token carrying their email in the `sub` claim. When the developer runs an AWS command, the CLI calls **AWS STS AssumeRoleWithWebIdentity** with that token; AWS validates it against the Vouch server and returns temporary credentials valid for up to 1 hour. Because the token is scoped to the authenticated user and short-lived, credentials cannot be shared or reused after expiry.

## Step 1 -- Register the Vouch OIDC provider

<span class="role-label">Admin task</span>

Vouch uses **exactly one OIDC provider per organization**. Register it once -- in your single AWS account, or in the **management account** if you use AWS Organizations. An administrator does this before any user can assume a role.

> For background on OIDC identity providers in AWS, see [Creating OpenID Connect (OIDC) identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) in the AWS documentation.

{{< tabs >}}
{{< tab "AWS CLI" >}}
```bash
aws iam create-open-id-connect-provider \
  --url "https://{{< instance-url >}}" \
  --client-id-list "https://{{< instance-url >}}"
```
{{< /tab >}}
{{< tab "CloudFormation" >}}
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Vouch OIDC Identity Provider

Resources:
  VouchOIDCProvider:
    Type: "AWS::IAM::OIDCProvider"
    Properties:
      Url: "https://{{< instance-url >}}"
      ClientIdList:
        - "https://{{< instance-url >}}"
```
{{< /tab >}}
{{< tab "Terraform" >}}
```hcl
resource "aws_iam_openid_connect_provider" "vouch" {
  url            = "https://{{< instance-url >}}"
  client_id_list = ["https://{{< instance-url >}}"]
}
```
{{< /tab >}}
{{< /tabs >}}

> **Note:** AWS fetches the JWKS from `https://{{< instance-url >}}/oauth/jwks` at runtime to verify token signatures. A `ThumbprintList` is no longer required -- AWS obtains the root CA thumbprint automatically.

---

## Choose your setup

<span class="role-label">Admin task</span>

How your team accesses AWS decides what you deploy next. This is the same question the `vouch setup aws` wizard asks each developer -- **"How do you access AWS?"** -- so admins and developers stay aligned:

| Your AWS layout | What to deploy | Guide |
|---|---|---|
| **Single account — one IAM role** | One role in this account | Continue to Step 2 below |
| **Multiple accounts — a management role that assumes into member roles** | A hub role here, plus a spoke role per account | [Role chaining](/docs/aws-multi-account/) |
| **Identity Center (SSO) — permission sets** | Register Vouch as a trusted token issuer | [Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) |

If you have an Organization, everything anchors in the management account. Single-account setup continues below.

---

## Step 2 -- Deploy a role

<span class="role-label">Admin task</span>

This is the role developers federate into with `vouch login`. Its **trust policy** is the same no matter what the role can do; only the **permissions policy** changes -- and *what the role is allowed to do is your team's decision.* Deploy the shared trust policy below, then attach a permissions policy one of two ways.

The `*@example.com` condition limits role assumption to anyone with a verified email in your domain (see [Tips for restricting access](#tips-for-restricting-access) for narrower patterns).

### Shared trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/{{< instance-url >}}"
      },
      "Action": [
        "sts:AssumeRoleWithWebIdentity",
        "sts:SetSourceIdentity",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "{{< instance-url >}}:aud": "https://{{< instance-url >}}"
        },
        "StringLike": {
          "{{< instance-url >}}:sub": "*@example.com",
          "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
        }
      }
    }
  ]
}
```

The `sub` and `sts:RoleSessionName` conditions bind the session to the authenticated user; see [Tips for restricting access](#tips-for-restricting-access) for what each does.

### Managed policy

Attach an AWS-managed policy. Start with `ReadOnlyAccess` and broaden to exactly what your team needs -- don't reach for `PowerUserAccess` by default. See AWS's [job-function managed policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html) for options.

{{< tabs >}}
{{< tab "CloudFormation" >}}
```yaml
Resources:
  VouchDeveloperRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: VouchDeveloper
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub "arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/{{< instance-url >}}"
            Action:
              - "sts:AssumeRoleWithWebIdentity"
              - "sts:SetSourceIdentity"
              - "sts:TagSession"
            Condition:
              StringEquals:
                "{{< instance-url >}}:aud": "https://{{< instance-url >}}"
              StringLike:
                "{{< instance-url >}}:sub": "*@example.com"
                "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
      ManagedPolicyArns:
        # Start safe. Attach the permissions your team needs.
        - !Sub "arn:${AWS::Partition}:iam::aws:policy/ReadOnlyAccess"
```
{{< /tab >}}
{{< tab "Terraform" >}}
```hcl
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  aws_partition  = data.aws_partition.current.partition
  aws_account_id = data.aws_caller_identity.current.account_id
}

resource "aws_iam_role" "vouch_developer" {
  name = "VouchDeveloper"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:${local.aws_partition}:iam::${local.aws_account_id}:oidc-provider/{{< instance-url >}}"
        }
        Action = [
          "sts:AssumeRoleWithWebIdentity",
          "sts:SetSourceIdentity",
          "sts:TagSession",
        ]
        Condition = {
          StringEquals = {
            "{{< instance-url >}}:aud" = "https://{{< instance-url >}}"
          }
          StringLike = {
            "{{< instance-url >}}:sub" = "*@example.com"
            "sts:RoleSessionName"      = "${{{< instance-url >}}:sub}"
          }
        }
      }
    ]
  })

  # Start safe. Attach the permissions your team needs.
  managed_policy_arns = ["arn:${local.aws_partition}:iam::aws:policy/ReadOnlyAccess"]
}
```
{{< /tab >}}
{{< /tabs >}}

### Explicit actions

Attach a custom inline policy listing only the actions your team needs. Use the same trust policy and role definition as the managed-policy example; replace the `ManagedPolicyArns` block with an inline policy. Example -- read/write a specific S3 bucket and read CloudWatch logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-data",
        "arn:aws:s3:::my-app-data/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:GetLogEvents",
        "logs:FilterLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

In CloudFormation, use [`Policies`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-iam-role.html#cfn-iam-role-policies) on the role; in Terraform, use [`aws_iam_role_policy`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy).

<div class="checkpoint">
<p><strong>You are done with role deployment when...</strong></p>
<ul>
  <li>The role uses the shared trust policy with your email domain in the <code>sub</code> condition.</li>
  <li>You attached a permissions policy your team is comfortable with (start with <code>ReadOnlyAccess</code>).</li>
  <li>You have the role ARN written down -- developers need it in Step 3.</li>
</ul>
</div>

---

## Step 3 -- Configure the Vouch CLI

<span class="role-label">Developer task</span>

Each developer runs the wizard -- it asks the same **"How do you access AWS?"** question and writes the AWS profile for you:

```bash
vouch setup aws
```

For single-account access you can skip the prompts by passing the role directly:

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

This command accepts the following flags:

- `--role` -- The ARN of the IAM role from Step 2. Omit all flags to launch the interactive wizard.
- `--profile` -- The AWS profile name to write credentials to (default: `vouch`; additional profiles auto-name as `vouch-2`, `vouch-3`, etc.).

For multi-account setups, `vouch setup aws` also accepts `--management-role`, `--identity-center-application`, `--region`, and `--discover` -- see [Multi-Account AWS](/docs/aws-multi-account/).

The command writes a `credential_process` entry into `~/.aws/config` so that the AWS CLI and SDKs automatically call `vouch credential aws` whenever credentials are needed:

```ini
[profile vouch]
credential_process = vouch credential aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

> **Note:** If you need a specific region for this profile, add a `region` line manually (e.g., `region = us-east-1`).

---

## Step 4 -- Test

<span class="role-label">Developer task</span>

Verify that everything is working:

```bash
# Make sure you are logged in
vouch login

# Check your identity
aws sts get-caller-identity --profile vouch
```

Expected output:

```json
{
  "UserId": "AROA...:alice@example.com",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@example.com"
}
```

Try running a command against a real AWS service:

```bash
aws s3 ls --profile vouch
```

<div class="checkpoint">
<p><strong>You are done when...</strong></p>
<ul>
  <li><code>aws sts get-caller-identity --profile vouch</code> returns an assumed-role ARN for the expected AWS account.</li>
  <li>The role session name matches the authenticated Vouch user.</li>
  <li>A real AWS command succeeds without static credentials in <code>~/.aws/credentials</code>.</li>
</ul>
</div>

---

## Console access

Open the AWS Management Console directly from the CLI without entering credentials in a browser:

```bash
vouch aws console
```

This uses your active Vouch session to obtain temporary STS credentials, exchanges them for a federation sign-in token, and opens the console in your default browser. Pass `--role` to specify a role, or omit it to use the role from your configured AWS profile. For [IAM Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) access, pass `--account <id> --permission-set <name>` (add `--idc-application <arn>` when more than one instance is configured); use `--via <management-role-arn>` to select a management role when multiple organizations are configured.

---

## Tips for restricting access

### Restrict by email address

Limit role assumption to specific users by adding an email condition to the trust policy:

```json
"Condition": {
  "StringEquals": {
    "{{< instance-url >}}:aud": "https://{{< instance-url >}}",
    "{{< instance-url >}}:sub": ["user@example.com"]
  },
  "StringLike": {
    "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
  }
}
```

### Restrict by email domain

Allow any user from a specific domain:

```json
"Condition": {
  "StringEquals": {
    "{{< instance-url >}}:aud": "https://{{< instance-url >}}"
  },
  "StringLike": {
    "{{< instance-url >}}:sub": "*@example.com",
    "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
  }
}
```

### Revoke access for a specific user

There are two complementary techniques for cutting off a user. Use both together for full offboarding.

**1. Immediate revocation via an explicit `Deny` on the role's permissions policy.** Permissions policies are evaluated on every AWS API call, and explicit `Deny` always wins. Because Vouch sets the `vouch:Email` session tag on every assumed-role session (see [Session tags](#session-tags)), you can block an offboarded user on the next API call -- including calls made with STS credentials that the Vouch agent has already cached:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOffboardedUsers",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/vouch:Email": [
            "former-employee@example.com"
          ]
        }
      }
    }
  ]
}
```

Attach this as an inline policy on each Vouch role, or apply it as a [Service Control Policy](/docs/aws-multi-account/) in the management account to enforce it organization-wide. Because Vouch marks session tags as transitive, the `vouch:Email` tag propagates through role chains, so the same condition works in both the entry role and any spoke roles.

**2. Block new role assumption via the trust policy.** Add a `StringNotEquals` clause to the trust policy listing the emails to deny. The next `AssumeRoleWithWebIdentity` call from that user will fail:

```json
"Condition": {
  "StringEquals": {
    "{{< instance-url >}}:aud": "https://{{< instance-url >}}"
  },
  "StringLike": {
    "{{< instance-url >}}:sub": "*@example.com",
    "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
  },
  "StringNotEquals": {
    "{{< instance-url >}}:sub": ["former-employee@example.com"]
  }
}
```

On its own, this only takes effect when the Vouch agent's cached STS credentials expire (up to 1 hour later), because the trust policy is evaluated at `AssumeRoleWithWebIdentity` time, not on every API call. Combined with the deny statement above, it provides a durable record of who is offboarded and prevents the role from being re-assumed even after the deny statement is later removed.

**For full offboarding**, also deactivate the user in Vouch so they cannot start new sessions and existing non-AWS credentials (like SSH certificates) are cut off:

- **With [SCIM](/docs/scim/) provisioning**, deactivating the user in your identity provider automatically revokes their active Vouch session and blocks future logins -- no manual step required.
- **Without SCIM**, an administrator can use the **Deactivate** and **Revoke credentials** actions in the [admin console](/docs/admin/) to do the same thing.

Note that revoking the Vouch session does not invalidate STS credentials already cached in the user's local Vouch agent -- the explicit `Deny` from step 1 is what blocks those on the AWS side. The Vouch-side action prevents new sessions and severs other credentials issued from the same session.

### Prevent session name spoofing

The `RoleSessionName` is a client-provided STS API parameter. Without a trust policy condition, someone with a valid Vouch JWT could set it to another user's email, making CloudTrail session ARNs misleading. The `sts:RoleSessionName` condition binds the session name to the authenticated `sub` claim from the validated JWT:

```json
"StringLike": {
  "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
}
```

All trust policy examples in this guide include this condition. AWS rejects any `AssumeRoleWithWebIdentity` call where the session name does not match the OIDC subject.

> **Note:** The immutable `SourceIdentity` claim (set to the user's email) provides a second attribution anchor in CloudTrail that cannot be spoofed regardless of this condition.

### Session tags

Vouch sets the following session tags when assuming a role, which you can use in IAM policies for [attribute-based access control (ABAC)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html):

| Tag Key | Value | Example |
|---------|-------|---------|
| `vouch:Email` | The user's verified email | `alice@example.com` |
| `vouch:Domain` | The user's organization domain (from the OIDC `hd` claim) | `example.com` |
| `vouch:AccessType` | Set to `ai` when an AI coding agent is detected | `ai` |
| `vouch:Agent` | The detected agent name (only present when an agent is detected) | `claude-code` |

You can reference these tags in IAM policy conditions using `aws:PrincipalTag`:

```json
{
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/vouch:Domain": "example.com"
    }
  }
}
```

Because Vouch marks all tags as [transitive](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html#id_session-tags_role-chaining), they automatically propagate through role chains. If a developer assumes a hub role via Vouch and then chains into a spoke role in another account, the spoke role's trust policy can still evaluate `aws:PrincipalTag/vouch:Email` and `aws:PrincipalTag/vouch:Domain` conditions.

---

## AI agent safety

When `vouch credential aws` runs inside an AI coding agent, Vouch automatically restricts the returned credentials to read-only access. No configuration is required -- the CLI detects the agent environment and applies the restriction transparently.

> **Note:** This read-only downscoping applies to the STS credential paths (`--role`, including role chaining). On the [IAM Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) path (`--account`/`--permission-set`), Vouch **refuses to issue credentials to a detected agent** rather than downscoping them -- permission-set credentials cannot be constrained with a session policy, so there is no way to enforce read-only access. Agent workflows that need AWS access should use the STS role-chaining model.

### How it works

The CLI checks for environment variables set by popular AI coding agents. When one is detected:

1. The [`ReadOnlyAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html) AWS managed policy is attached as a [session policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_session), which limits the effective permissions to the intersection of the role's policies and `ReadOnlyAccess` -- regardless of what the role itself allows.
2. The `vouch:AccessType=ai` and `vouch:Agent=<name>` session tags are added, where `<name>` is the verbatim value of the detected agent environment variable (for agents that set `AI_AGENT` or `AGENT`, the raw value is forwarded; for agents detected by a marker variable like `CLAUDE_CODE` or `CURSOR_TRACE_ID`, the agent name is used). These tags appear on every CloudTrail event for the session, so you can attribute API calls to the specific agent that made them.

### Supported agents

Vouch detects the following AI coding agents:

| Agent | Environment Variable |
|-------|---------------------|
| Claude Code | `AI_AGENT` or `CLAUDE_CODE` |
| Cursor | `CURSOR_TRACE_ID` |
| GitHub Copilot | `COPILOT_MODEL` |
| OpenAI Codex | `CODEX_SANDBOX` |
| Google Gemini | `GEMINI_CLI` |
| Augment | `AUGMENT_AGENT` |
| Cline | `CLINE_ACTIVE` |
| Amp | `AGENT=amp` |
| Goose | `AGENT=goose` |

If your agent is not listed, it will be detected if it sets the `AGENT` or `AI_AGENT` environment variable (an emerging convention).

### Role chaining with agents

When using [role chaining](/docs/aws-multi-account/#architecture) with an AI agent, Vouch applies an additional inline session policy to the management-account hop that restricts it to STS actions only (`sts:AssumeRole`, `sts:TagSession`, `sts:SetSourceIdentity`). The final role hop receives the `ReadOnlyAccess` session policy.

---

## How Vouch compares to `aws login` and `aws sso login`

AWS's built-in [`aws login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-local-development) and [`aws sso login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-sso) cover AWS only. `vouch login` covers AWS *and* SSH, GitHub, Docker, Cargo, CodeCommit, CodeArtifact, and databases -- from one phishing-resistant hardware-key session where every credential traces back to a physical key.

| | `aws login` | `aws sso login` | `vouch login` |
|---|---|---|---|
| **Authentication** | Browser + console credentials | Browser + Identity Center | YubiKey tap (FIDO2) |
| **Phishing-resistant** | Depends on IdP | Depends on IdP | Yes (hardware-bound) |
| **AWS credentials** | Yes (up to 12h) | Yes | Yes (up to 1h) |
| **SSH, GitHub, Docker, etc.** | No | No | Yes |
| **Identity in CloudTrail** | IAM user or role | SSO user | Hardware-verified user |
| **Requires AWS-managed service** | No | IAM Identity Center | No |

If you already use IAM Identity Center, `aws sso login` may cover your AWS needs. Vouch fits when you want one authentication event to cover AWS and everything else your team uses.

---

## Troubleshooting

### "Not authorized to perform sts:AssumeRoleWithWebIdentity"

- Verify the OIDC provider URL in the IAM trust policy matches `https://{{< instance-url >}}` exactly (no trailing slash).
- Confirm the `aud` condition matches the client ID registered with the OIDC provider.
- Ensure the trust policy `Action` includes all three required actions: `sts:AssumeRoleWithWebIdentity`, `sts:SetSourceIdentity`, and `sts:TagSession`.

### "Token is expired"

- Run `vouch login` again to refresh your session. OIDC tokens are short-lived by design.

### "Invalid identity token"

- Ensure the OIDC provider in AWS points to the correct Vouch server URL: `https://{{< instance-url >}}`.
- Verify that the Vouch server's JWKS endpoint (`https://{{< instance-url >}}/oauth/jwks`) is reachable from the internet (AWS must be able to fetch it).

### Credentials not appearing in the expected profile

- Run `vouch setup aws` again and verify the profile name.
- Check `~/.aws/config` for conflicting profile definitions.

### Permission errors after assuming the role

- The trust policy controls who can assume the role; the permissions policy controls what they can do. Verify the correct permissions policies are attached to the IAM role.

---

## Related guides

- [Multi-Account AWS Strategy](/docs/aws-multi-account/) -- Deploy Vouch across multiple AWS accounts with CloudFormation StackSets or Terraform modules.
- [Amazon EKS](/docs/eks/) -- Use your Vouch-backed AWS credentials to authenticate to Kubernetes clusters.
- [AWS Systems Manager](/docs/ssm/) -- Connect to EC2 instances through Session Manager using Vouch credentials.
- [Database Authentication](/docs/databases/) -- Connect to RDS, Aurora, and Redshift with IAM authentication.
- [AWS CodeArtifact](/docs/codeartifact/) -- Authenticate to package repositories using Vouch.
- [Infrastructure as Code](/docs/iac/) -- Use CDK, Terraform, SAM, and other IaC tools with Vouch credentials.
- [Claude & OpenAI APIs](/docs/ai-api-keys/) -- Replace long-lived AI provider API keys with short-lived tokens via Workload Identity Federation.
