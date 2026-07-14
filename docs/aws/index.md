# Replace AWS Access Keys with Short-Lived Credentials

> Stop distributing long-lived AWS access keys. Use OIDC federation to get temporary STS credentials backed by a YubiKey.

Source: https://vouch.sh/docs/aws/
Last updated: 2026-07-13

---
Vouch eliminates static AWS access keys. You configure AWS to trust Vouch as an OIDC identity provider, and developers get temporary STS credentials -- valid for up to 1 hour -- after authenticating with their YubiKey. Every API call is tied to a verified human identity in CloudTrail.

{{&lt; tldr &gt;}}
- **Prerequisite:** [Getting Started](/docs/getting-started/) (CLI installed, YubiKey enrolled).
- **Admin, once:** [register the OIDC provider](#step-1----register-the-vouch-oidc-provider) and [deploy a role](#step-2----deploy-a-role); share the role ARN.
- **Each developer:** `vouch setup aws --role &lt;ROLE_ARN&gt;`, then verify with `aws sts get-caller-identity --profile vouch`.
- Rolling this out to a whole team? Follow the [Team Rollout playbook](/docs/rollout/).
{{&lt; /tldr &gt;}}

## Step 1 -- Register the Vouch OIDC provider

{{&lt; role admin &gt;}}

Vouch uses **exactly one OIDC provider per organization**. Register it once -- in your single AWS account, or in the **management account** if you use AWS Organizations. An administrator does this before any user can assume a role.

&gt; For background on OIDC identity providers in AWS, see [Creating OpenID Connect (OIDC) identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) in the AWS documentation.


#### AWS CLI

```bash
aws iam create-open-id-connect-provider \
  --url &#34;https://us.vouch.sh&#34; \
  --client-id-list &#34;https://us.vouch.sh&#34;
```

#### CloudFormation

```yaml
AWSTemplateFormatVersion: &#34;2010-09-09&#34;
Description: Vouch OIDC Identity Provider

Resources:
  VouchOIDCProvider:
    Type: &#34;AWS::IAM::OIDCProvider&#34;
    Properties:
      Url: &#34;https://us.vouch.sh&#34;
      ClientIdList:
        - &#34;https://us.vouch.sh&#34;
```

#### Terraform

```hcl
resource &#34;aws_iam_openid_connect_provider&#34; &#34;vouch&#34; {
  url            = &#34;https://us.vouch.sh&#34;
  client_id_list = [&#34;https://us.vouch.sh&#34;]
}
```



&gt; **Note:** AWS fetches the JWKS from `https://us.vouch.sh/oauth/jwks` at runtime to verify token signatures. A `ThumbprintList` is no longer required -- AWS obtains the root CA thumbprint automatically.

---

## Choose your setup

{{&lt; role admin &gt;}}

How your team accesses AWS decides what you deploy next. This is the same question the `vouch setup aws` wizard asks each developer -- **&#34;How do you access AWS?&#34;** -- so admins and developers stay aligned:

| Your AWS layout | What to deploy | Guide |
|---|---|---|
| **Single account — one IAM role** | One role in this account | Continue to Step 2 below |
| **Multiple accounts — a management role that assumes into member roles** | A hub role here, plus a spoke role per account | [Role chaining](/docs/aws-multi-account/) |
| **Identity Center (SSO) — permission sets** | Register Vouch as a trusted token issuer | [Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) |

If you have an Organization, everything anchors in the management account. Single-account setup continues below.

---

## Step 2 -- Deploy a role

{{&lt; role admin &gt;}}

This is the role developers federate into with `vouch login`. Its **trust policy** is the same no matter what the role can do; only the **permissions policy** changes -- and *what the role is allowed to do is your team&#39;s decision.* Deploy the shared trust policy below, then attach a permissions policy one of two ways.

The `*@example.com` condition limits role assumption to anyone with a verified email in your domain (see [Tips for restricting access](#tips-for-restricting-access) for narrower patterns).

### Shared trust policy

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Principal&#34;: {
        &#34;Federated&#34;: &#34;arn:aws:iam::ACCOUNT_ID:oidc-provider/us.vouch.sh&#34;
      },
      &#34;Action&#34;: [
        &#34;sts:AssumeRoleWithWebIdentity&#34;,
        &#34;sts:SetSourceIdentity&#34;,
        &#34;sts:TagSession&#34;
      ],
      &#34;Condition&#34;: {
        &#34;StringEquals&#34;: {
          &#34;us.vouch.sh:aud&#34;: &#34;https://us.vouch.sh&#34;
        },
        &#34;StringLike&#34;: {
          &#34;us.vouch.sh:sub&#34;: &#34;*@example.com&#34;,
          &#34;sts:RoleSessionName&#34;: &#34;${us.vouch.sh:sub}&#34;
        },
        &#34;Bool&#34;: {
          &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
        }
      }
    }
  ]
}
```

The `sub` and `sts:RoleSessionName` conditions bind the session to the authenticated user, and `sts:RoleAuthorizedByIdp` requires the token to be pinned to this role; see [Tips for restricting access](#tips-for-restricting-access) for what each does.

### Managed policy

Attach an AWS-managed policy. Start with `ReadOnlyAccess` and broaden to exactly what your team needs -- don&#39;t reach for `PowerUserAccess` by default. See AWS&#39;s [job-function managed policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html) for options.


#### CloudFormation

```yaml
Resources:
  VouchDeveloperRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: VouchDeveloper
      AssumeRolePolicyDocument:
        Version: &#34;2012-10-17&#34;
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub &#34;arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/us.vouch.sh&#34;
            Action:
              - &#34;sts:AssumeRoleWithWebIdentity&#34;
              - &#34;sts:SetSourceIdentity&#34;
              - &#34;sts:TagSession&#34;
            Condition:
              StringEquals:
                &#34;us.vouch.sh:aud&#34;: &#34;https://us.vouch.sh&#34;
              StringLike:
                &#34;us.vouch.sh:sub&#34;: &#34;*@example.com&#34;
                &#34;sts:RoleSessionName&#34;: &#34;${us.vouch.sh:sub}&#34;
              Bool:
                &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
      ManagedPolicyArns:
        # Start safe. Attach the permissions your team needs.
        - !Sub &#34;arn:${AWS::Partition}:iam::aws:policy/ReadOnlyAccess&#34;
```

#### Terraform

```hcl
data &#34;aws_caller_identity&#34; &#34;current&#34; {}
data &#34;aws_partition&#34; &#34;current&#34; {}

locals {
  aws_partition  = data.aws_partition.current.partition
  aws_account_id = data.aws_caller_identity.current.account_id
}

resource &#34;aws_iam_role&#34; &#34;vouch_developer&#34; {
  name = &#34;VouchDeveloper&#34;

  assume_role_policy = jsonencode({
    Version = &#34;2012-10-17&#34;
    Statement = [
      {
        Effect = &#34;Allow&#34;
        Principal = {
          Federated = &#34;arn:${local.aws_partition}:iam::${local.aws_account_id}:oidc-provider/us.vouch.sh&#34;
        }
        Action = [
          &#34;sts:AssumeRoleWithWebIdentity&#34;,
          &#34;sts:SetSourceIdentity&#34;,
          &#34;sts:TagSession&#34;,
        ]
        Condition = {
          StringEquals = {
            &#34;us.vouch.sh:aud&#34; = &#34;https://us.vouch.sh&#34;
          }
          StringLike = {
            &#34;us.vouch.sh:sub&#34; = &#34;*@example.com&#34;
            &#34;sts:RoleSessionName&#34;      = &#34;${us.vouch.sh:sub}&#34;
          }
          Bool = {
            &#34;sts:RoleAuthorizedByIdp&#34; = &#34;true&#34;
          }
        }
      }
    ]
  })

  # Start safe. Attach the permissions your team needs.
  managed_policy_arns = [&#34;arn:${local.aws_partition}:iam::aws:policy/ReadOnlyAccess&#34;]
}
```



### Explicit actions

Attach a custom inline policy listing only the actions your team needs. Use the same trust policy and role definition as the managed-policy example; replace the `ManagedPolicyArns` block with an inline policy. Example -- read/write a specific S3 bucket and read CloudWatch logs:

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: [
        &#34;s3:GetObject&#34;,
        &#34;s3:PutObject&#34;,
        &#34;s3:ListBucket&#34;
      ],
      &#34;Resource&#34;: [
        &#34;arn:aws:s3:::my-app-data&#34;,
        &#34;arn:aws:s3:::my-app-data/*&#34;
      ]
    },
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: [
        &#34;logs:GetLogEvents&#34;,
        &#34;logs:FilterLogEvents&#34;,
        &#34;logs:DescribeLogGroups&#34;,
        &#34;logs:DescribeLogStreams&#34;
      ],
      &#34;Resource&#34;: &#34;*&#34;
    }
  ]
}
```

In CloudFormation, use [`Policies`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-iam-role.html#cfn-iam-role-policies) on the role; in Terraform, use [`aws_iam_role_policy`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy).

&lt;div class=&#34;checkpoint&#34;&gt;
&lt;p&gt;&lt;strong&gt;You are done with role deployment when...&lt;/strong&gt;&lt;/p&gt;
&lt;ul&gt;
  &lt;li&gt;The role uses the shared trust policy with your email domain in the &lt;code&gt;sub&lt;/code&gt; condition.&lt;/li&gt;
  &lt;li&gt;You attached a permissions policy your team is comfortable with (start with &lt;code&gt;ReadOnlyAccess&lt;/code&gt;).&lt;/li&gt;
  &lt;li&gt;You have the role ARN written down -- developers need it in Step 3.&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;

---

## Step 3 -- Configure the Vouch CLI

{{&lt; role developer &gt;}}

Each developer runs the wizard -- it asks the same **&#34;How do you access AWS?&#34;** question and writes the AWS profile for you:

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

&gt; **Note:** If you need a specific region for this profile, add a `region` line manually (e.g., `region = us-east-1`).

---

## Step 4 -- Test

{{&lt; role developer &gt;}}

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
  &#34;UserId&#34;: &#34;AROA...:alice@example.com&#34;,
  &#34;Account&#34;: &#34;123456789012&#34;,
  &#34;Arn&#34;: &#34;arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@example.com&#34;
}
```

Try running a command against a real AWS service:

```bash
aws s3 ls --profile vouch
```

&lt;div class=&#34;checkpoint&#34;&gt;
&lt;p&gt;&lt;strong&gt;You are done when...&lt;/strong&gt;&lt;/p&gt;
&lt;ul&gt;
  &lt;li&gt;&lt;code&gt;aws sts get-caller-identity --profile vouch&lt;/code&gt; returns an assumed-role ARN for the expected AWS account.&lt;/li&gt;
  &lt;li&gt;The role session name matches the authenticated Vouch user.&lt;/li&gt;
  &lt;li&gt;A real AWS command succeeds without static credentials in &lt;code&gt;~/.aws/credentials&lt;/code&gt;.&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;

---

## Console access

Open the AWS Management Console directly from the CLI without entering credentials in a browser:

```bash
vouch aws console
```

This uses your active Vouch session to obtain temporary STS credentials, exchanges them for a federation sign-in token, and opens the console in your default browser. Pass `--role` to specify a role, or omit it to use the role from your configured AWS profile. For [IAM Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) access, pass `--account &lt;id&gt; --permission-set &lt;name&gt;` (add `--idc-application &lt;arn&gt;` when more than one instance is configured); use `--via &lt;management-role-arn&gt;` to select a management role when multiple organizations are configured.

---

## Tips for restricting access

### Restrict by email address

Limit role assumption to specific users by adding an email condition to the trust policy:

```json
&#34;Condition&#34;: {
  &#34;StringEquals&#34;: {
    &#34;us.vouch.sh:aud&#34;: &#34;https://us.vouch.sh&#34;,
    &#34;us.vouch.sh:sub&#34;: [&#34;user@example.com&#34;]
  },
  &#34;StringLike&#34;: {
    &#34;sts:RoleSessionName&#34;: &#34;${us.vouch.sh:sub}&#34;
  },
  &#34;Bool&#34;: {
    &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
  }
}
```

### Restrict by email domain

Allow any user from a specific domain:

```json
&#34;Condition&#34;: {
  &#34;StringEquals&#34;: {
    &#34;us.vouch.sh:aud&#34;: &#34;https://us.vouch.sh&#34;
  },
  &#34;StringLike&#34;: {
    &#34;us.vouch.sh:sub&#34;: &#34;*@example.com&#34;,
    &#34;sts:RoleSessionName&#34;: &#34;${us.vouch.sh:sub}&#34;
  },
  &#34;Bool&#34;: {
    &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
  }
}
```

### Revoke access for a specific user

There are two complementary techniques for cutting off a user. Use both together for full offboarding.

**1. Immediate revocation via an explicit `Deny` on the role&#39;s permissions policy.** Permissions policies are evaluated on every AWS API call, and explicit `Deny` always wins. Because Vouch sets the `vouch:Email` session tag on every assumed-role session (see [Session tags](#session-tags)), you can block an offboarded user on the next API call -- including calls made with STS credentials that the Vouch agent has already cached:

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Sid&#34;: &#34;DenyOffboardedUsers&#34;,
      &#34;Effect&#34;: &#34;Deny&#34;,
      &#34;Action&#34;: &#34;*&#34;,
      &#34;Resource&#34;: &#34;*&#34;,
      &#34;Condition&#34;: {
        &#34;StringEquals&#34;: {
          &#34;aws:PrincipalTag/vouch:Email&#34;: [
            &#34;former-employee@example.com&#34;
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
&#34;Condition&#34;: {
  &#34;StringEquals&#34;: {
    &#34;us.vouch.sh:aud&#34;: &#34;https://us.vouch.sh&#34;
  },
  &#34;StringLike&#34;: {
    &#34;us.vouch.sh:sub&#34;: &#34;*@example.com&#34;,
    &#34;sts:RoleSessionName&#34;: &#34;${us.vouch.sh:sub}&#34;
  },
  &#34;StringNotEquals&#34;: {
    &#34;us.vouch.sh:sub&#34;: [&#34;former-employee@example.com&#34;]
  },
  &#34;Bool&#34;: {
    &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
  }
}
```

On its own, this only takes effect when the Vouch agent&#39;s cached STS credentials expire (up to 1 hour later), because the trust policy is evaluated at `AssumeRoleWithWebIdentity` time, not on every API call. Combined with the deny statement above, it provides a durable record of who is offboarded and prevents the role from being re-assumed even after the deny statement is later removed.

**For full offboarding**, also deactivate the user in Vouch so they cannot start new sessions and existing non-AWS credentials (like SSH certificates) are cut off:

- **With [SCIM](/docs/scim/) provisioning**, deactivating the user in your identity provider automatically revokes their active Vouch session and blocks future logins -- no manual step required.
- **Without SCIM**, an administrator can use the **Deactivate** and **Revoke credentials** actions in the [admin console](/docs/admin/) to do the same thing.

Note that revoking the Vouch session does not invalidate STS credentials already cached in the user&#39;s local Vouch agent -- the explicit `Deny` from step 1 is what blocks those on the AWS side. The Vouch-side action prevents new sessions and severs other credentials issued from the same session.

### Prevent session name spoofing

The `RoleSessionName` is a client-provided STS API parameter. Without a trust policy condition, someone with a valid Vouch JWT could set it to another user&#39;s email, making CloudTrail session ARNs misleading. The `sts:RoleSessionName` condition binds the session name to the authenticated `sub` claim from the validated JWT:

```json
&#34;StringLike&#34;: {
  &#34;sts:RoleSessionName&#34;: &#34;${us.vouch.sh:sub}&#34;
}
```

All trust policy examples in this guide include this condition. AWS rejects any `AssumeRoleWithWebIdentity` call where the session name does not match the OIDC subject.

&gt; **Note:** The immutable `SourceIdentity` claim (set to the user&#39;s email) provides a second attribution anchor in CloudTrail that cannot be spoofed regardless of this condition.

### Require role pinning

The Vouch CLI requests each OIDC token pinned to the role it is about to assume, and the Vouch server embeds that role ARN in the token&#39;s `https://aws.amazon.com/roles` claim. AWS STS enforces the claim: a pinned token can only be exchanged for the exact role it was minted for, so a leaked token cannot assume any other role that trusts the Vouch issuer. This happens automatically -- no configuration is required.

The trust policy makes pinning mandatory with the [`sts:RoleAuthorizedByIdp`](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-sts) condition key:

```json
&#34;Condition&#34;: {
  &#34;Bool&#34;: {
    &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
  }
}
```

With this in place, AWS rejects any `AssumeRoleWithWebIdentity` call whose token does not name this role in its `roles` claim. All trust policy examples in this guide include this condition. Two caveats:

- **Roll the CLI out first.** Older CLI versions do not request pinning, so their tokens carry no `roles` claim and fail the condition. If part of your team is on an older CLI, leave the condition out until everyone has upgraded.
- **Web-identity trust statements only.** The condition key is defined for `AssumeRoleWithWebIdentity`. A role assumed by a plain SigV4 `sts:AssumeRole` call -- such as a [spoke role](/docs/aws-multi-account/) in a management-role chain -- has no OIDC token in the request, so a `Bool`-`true` condition there can never match.

Matching is by exact role ARN; wildcards are not supported.

### Session tags

Vouch sets the following session tags when assuming a role, which you can use in IAM policies for [attribute-based access control (ABAC)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html):

| Tag Key | Value | Example |
|---------|-------|---------|
| `vouch:Email` | The user&#39;s verified email | `alice@example.com` |
| `vouch:Domain` | The user&#39;s organization domain (from the OIDC `hd` claim) | `example.com` |
| `vouch:AccessType` | Set to `ai` when an AI coding agent is detected | `ai` |
| `vouch:Agent` | The detected agent name (only present when an agent is detected) | `claude-code` |

You can reference these tags in IAM policy conditions using `aws:PrincipalTag`:

```json
{
  &#34;Condition&#34;: {
    &#34;StringEquals&#34;: {
      &#34;aws:PrincipalTag/vouch:Domain&#34;: &#34;example.com&#34;
    }
  }
}
```

Because Vouch marks all tags as [transitive](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html#id_session-tags_role-chaining), they automatically propagate through role chains. If a developer assumes a hub role via Vouch and then chains into a spoke role in another account, the spoke role&#39;s trust policy can still evaluate `aws:PrincipalTag/vouch:Email` and `aws:PrincipalTag/vouch:Domain` conditions.

---

## AI agent safety

When `vouch credential aws` runs inside an AI coding agent, Vouch automatically restricts the returned credentials to read-only access. No configuration is required -- the CLI detects the agent environment and applies the restriction transparently.

&gt; **Note:** This read-only downscoping applies to the STS credential paths (`--role`, including role chaining). On the [IAM Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) path (`--account`/`--permission-set`), Vouch **refuses to issue credentials to a detected agent** rather than downscoping them -- permission-set credentials cannot be constrained with a session policy, so there is no way to enforce read-only access. Agent workflows that need AWS access should use the STS role-chaining model.

### How it works

The CLI checks for environment variables set by popular AI coding agents. When one is detected:

1. The [`ReadOnlyAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html) AWS managed policy is attached as a [session policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_session), which limits the effective permissions to the intersection of the role&#39;s policies and `ReadOnlyAccess` -- regardless of what the role itself allows.
2. The `vouch:AccessType=ai` and `vouch:Agent=&lt;name&gt;` session tags are added, where `&lt;name&gt;` is the verbatim value of the detected agent environment variable (for agents that set `AI_AGENT` or `AGENT`, the raw value is forwarded; for agents detected by a marker variable like `CLAUDE_CODE` or `CURSOR_TRACE_ID`, the agent name is used). These tags appear on every CloudTrail event for the session, so you can attribute API calls to the specific agent that made them.

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

## How it works

`vouch login` authenticates the developer with their YubiKey, and the Vouch server issues a short-lived OIDC ID token carrying their email in the `sub` claim and the target role ARN in the `https://aws.amazon.com/roles` claim. When the developer runs an AWS command, the CLI calls **AWS STS AssumeRoleWithWebIdentity** with that token; AWS validates it against the Vouch server and returns temporary credentials valid for up to 1 hour. Because the token is scoped to the authenticated user, [pinned to the requested role](#require-role-pinning), and short-lived, credentials cannot be shared, redirected to another role, or reused after expiry.

---

## How Vouch compares to `aws login` and `aws sso login`

AWS&#39;s built-in [`aws login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-local-development) and [`aws sso login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-sso) cover AWS only. `vouch login` covers AWS *and* SSH, GitHub, Docker, Cargo, CodeCommit, CodeArtifact, and databases -- from one phishing-resistant hardware-key session where every credential traces back to a physical key.

| | `aws login` | `aws sso login` | `vouch login` |
|---|---|---|---|
| **Authentication** | Browser &#43; console credentials | Browser &#43; Identity Center | YubiKey tap (FIDO2) |
| **Phishing-resistant** | Depends on IdP | Depends on IdP | Yes (hardware-bound) |
| **AWS credentials** | Yes (up to 12h) | Yes | Yes (up to 1h) |
| **SSH, GitHub, Docker, etc.** | No | No | Yes |
| **Identity in CloudTrail** | IAM user or role | SSO user | Hardware-verified user |
| **Requires AWS-managed service** | No | IAM Identity Center | No |

If you already use IAM Identity Center, `aws sso login` may cover your AWS needs. Vouch fits when you want one authentication event to cover AWS and everything else your team uses.

---

## Troubleshooting

### &#34;Not authorized to perform sts:AssumeRoleWithWebIdentity&#34;

- Verify the OIDC provider URL in the IAM trust policy matches `https://us.vouch.sh` exactly (no trailing slash).
- Confirm the `aud` condition matches the client ID registered with the OIDC provider.
- Ensure the trust policy `Action` includes all three required actions: `sts:AssumeRoleWithWebIdentity`, `sts:SetSourceIdentity`, and `sts:TagSession`.
- If the trust policy requires [`sts:RoleAuthorizedByIdp`](#require-role-pinning), make sure the CLI is up to date -- older CLI versions issue tokens without the `roles` claim and fail the condition.

### &#34;Token is expired&#34;

- Run `vouch login` again to refresh your session. OIDC tokens are short-lived by design.

### &#34;Invalid identity token&#34;

- Ensure the OIDC provider in AWS points to the correct Vouch server URL: `https://us.vouch.sh`.
- Verify that the Vouch server&#39;s JWKS endpoint (`https://us.vouch.sh/oauth/jwks`) is reachable from the internet (AWS must be able to fetch it).

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
- [Claude &amp; OpenAI APIs](/docs/ai-api-keys/) -- Replace long-lived AI provider API keys with short-lived tokens via Workload Identity Federation.
