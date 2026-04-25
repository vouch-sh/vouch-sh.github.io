# Replace AWS Access Keys with Short-Lived Credentials

> Stop distributing long-lived AWS access keys. Use OIDC federation to get temporary STS credentials backed by a YubiKey.

Source: https://vouch.sh/docs/aws/
Last updated: 2026-04-25

---


Vouch eliminates static AWS access keys. You configure AWS to trust Vouch as an OIDC identity provider, and developers get temporary STS credentials -- valid for up to 1 hour -- after authenticating with their YubiKey. Every API call is tied to a verified human identity in CloudTrail.

## How Vouch compares to `aws login` and `aws sso login`

AWS provides two built-in CLI authentication commands. Here is how they compare to Vouch:

- **[`aws login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-local-development)** -- New in AWS CLI v2.32+. Opens a browser to authenticate with your AWS Console credentials (IAM user, root, or federated identity) and issues temporary credentials for up to 12 hours. It requires the `SignInLocalDevelopmentAccess` managed policy and only covers AWS -- it does not provide credentials for SSH, GitHub, Docker registries, or other services.

- **[`aws sso login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-sso)** -- Authenticates through AWS IAM Identity Center (formerly AWS SSO). It requires an Identity Center instance and an SSO-configured profile (`aws configure sso`). Like `aws login`, it only covers AWS services.

- **`vouch login`** -- Authenticates with a FIDO2 hardware key (YubiKey) and provides credentials for AWS, SSH, GitHub, Docker registries, Cargo registries, AWS CodeCommit, AWS CodeArtifact, databases, and any OIDC-compatible application -- all from a single session. Every credential is tied to a hardware-verified human identity, and authentication is phishing-resistant by design.

| | `aws login` | `aws sso login` | `vouch login` |
|---|---|---|---|
| **Authentication** | Browser + console credentials | Browser + Identity Center | YubiKey tap (FIDO2) |
| **Phishing-resistant** | Depends on IdP | Depends on IdP | Yes (hardware-bound) |
| **AWS credentials** | Yes (up to 12h) | Yes | Yes (up to 1h) |
| **SSH, GitHub, Docker, etc.** | No | No | Yes |
| **Identity in CloudTrail** | IAM user or role | SSO user | Hardware-verified user |
| **Requires AWS-managed service** | No | IAM Identity Center | No |

If you already use IAM Identity Center, `aws sso login` may cover your AWS needs. Vouch is a better fit when you want a single authentication event to cover AWS and everything else your team uses, with the guarantee that every credential traces back to a physical hardware key.

## How it works

1. The developer runs `vouch login` and authenticates with their YubiKey.
2. The Vouch server issues a short-lived **OIDC ID token** signed with **ES256** (ECDSA over P-256). The token contains claims such as `sub` (user ID) and `email`.
3. When the developer runs an AWS command (or `vouch credential aws`), the CLI calls **AWS STS AssumeRoleWithWebIdentity**, presenting the ID token.
4. AWS validates the token signature against the Vouch server's JWKS endpoint, checks the audience and issuer, and returns temporary credentials (access key, secret key, session token) valid for up to 1 hour.
5. The developer's AWS CLI, SDK, or Terraform session uses these credentials transparently.

Because the ID token is scoped to the authenticated user and is short-lived, credentials cannot be shared or reused after expiry.

## Who does what

| Role | Steps | Outcome |
|---|---|---|
| **AWS administrator** | Step 1 and Step 2 | AWS trusts Vouch and exposes an IAM role developers can assume. |
| **Developer** | Step 3 and Step 4 | The local AWS profile uses Vouch credentials and confirms the assumed identity. |

---

## Step 1 -- Create the OIDC Provider in AWS (admin)

<span class="role-label">Admin task</span>

Before any user can assume a role, an administrator must register the Vouch server as an OIDC identity provider in the target AWS account.

### AWS CLI

> For background on OIDC identity providers in AWS, see [Creating OpenID Connect (OIDC) identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) in the AWS documentation.

```bash
aws iam create-open-id-connect-provider \
  --url "https://us.vouch.sh" \
  --client-id-list "https://us.vouch.sh"
```

> **Note:** AWS fetches the JWKS from `https://us.vouch.sh/.well-known/jwks.json` at runtime to verify token signatures. A `ThumbprintList` is no longer required -- AWS obtains the root CA thumbprint automatically.

### CloudFormation

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Vouch OIDC Identity Provider

Resources:
  VouchOIDCProvider:
    Type: "AWS::IAM::OIDCProvider"
    Properties:
      Url: "https://us.vouch.sh"
      ClientIdList:
        - "https://us.vouch.sh"
```

### Terraform

```hcl
resource "aws_iam_openid_connect_provider" "vouch" {
  url            = "https://us.vouch.sh"
  client_id_list = ["https://us.vouch.sh"]
}
```

---

## Step 2 -- Create an IAM Role (admin)

<span class="role-label">Admin task</span>

Create an IAM role that developers will assume. The trust policy must allow [`AssumeRoleWithWebIdentity`](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) from the Vouch OIDC provider.

### AWS CLI

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/vouch-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/us.vouch.sh"
      },
      "Action": [
        "sts:AssumeRoleWithWebIdentity",
        "sts:SetSourceIdentity",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "us.vouch.sh:aud": "https://us.vouch.sh"
        },
        "StringLike": {
          "us.vouch.sh:sub": "*@example.com",
          "sts:RoleSessionName": "${us.vouch.sh:sub}"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name VouchDeveloper \
  --assume-role-policy-document file:///tmp/vouch-trust-policy.json

# Attach a permissions policy (example: read-only access)
aws iam attach-role-policy \
  --role-name VouchDeveloper \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### CloudFormation

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
              Federated: !Sub "arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/us.vouch.sh"
            Action:
              - "sts:AssumeRoleWithWebIdentity"
              - "sts:SetSourceIdentity"
              - "sts:TagSession"
            Condition:
              StringEquals:
                "us.vouch.sh:aud": "https://us.vouch.sh"
              StringLike:
                "us.vouch.sh:sub": "*@example.com"
                "sts:RoleSessionName": "${us.vouch.sh:sub}"
      ManagedPolicyArns:
        - !Sub "arn:${AWS::Partition}:iam::aws:policy/ReadOnlyAccess"
```

### Terraform

```hcl
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}
data "aws_iam_policy" "readonly" {
  name = "ReadOnlyAccess"
}

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
          Federated = "arn:${local.aws_partition}:iam::${local.aws_account_id}:oidc-provider/us.vouch.sh"
        }
        Action = [
          "sts:AssumeRoleWithWebIdentity",
          "sts:SetSourceIdentity",
          "sts:TagSession",
        ]
        Condition = {
          StringEquals = {
            "us.vouch.sh:aud" = "https://us.vouch.sh"
          }
          StringLike = {
            "us.vouch.sh:sub" = "*@example.com"
            "sts:RoleSessionName"      = "${us.vouch.sh:sub}"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "vouch_developer_readonly" {
  role       = aws_iam_role.vouch_developer.name
  policy_arn = data.aws_iam_policy.readonly.arn
}
```

---

## Tips for restricting access

### Restrict by email address

Limit role assumption to specific users by adding an email condition to the trust policy:

```json
"Condition": {
  "StringEquals": {
    "us.vouch.sh:aud": "https://us.vouch.sh",
    "us.vouch.sh:sub": ["user@example.com"]
  },
  "StringLike": {
    "sts:RoleSessionName": "${us.vouch.sh:sub}"
  }
}
```

### Restrict by email domain

Allow any user from a specific domain:

```json
"Condition": {
  "StringEquals": {
    "us.vouch.sh:aud": "https://us.vouch.sh"
  },
  "StringLike": {
    "us.vouch.sh:sub": "*@example.com",
    "sts:RoleSessionName": "${us.vouch.sh:sub}"
  }
}
```

### Prevent session name spoofing

The `RoleSessionName` is a client-provided STS API parameter. Without a trust policy condition, someone with a valid Vouch JWT could set it to another user's email, making CloudTrail session ARNs misleading. The `sts:RoleSessionName` condition binds the session name to the authenticated `sub` claim from the validated JWT:

```json
"StringLike": {
  "sts:RoleSessionName": "${us.vouch.sh:sub}"
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

## Step 3 -- Configure the Vouch CLI

<span class="role-label">Developer task</span>

Each developer needs to tell the CLI which IAM role to assume and in which AWS profile to store the credentials. Run:

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

This command accepts the following flags:

- `--role` -- The ARN of the IAM role created in Step 2 (required).
- `--profile` -- The AWS profile name to write credentials to (default: `vouch`; additional profiles auto-name as `vouch-2`, `vouch-3`, etc.).

The command writes a `credential_process` entry into `~/.aws/config` so that the AWS CLI and SDKs automatically call `vouch credential aws` whenever credentials are needed:

```ini
[profile vouch]
credential_process = vouch credential aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

> **Note:** If you need a specific region for this profile, add a `region` line manually (e.g., `region = us-east-1`).

If your organization uses AWS IAM Identity Center, you can auto-discover all available accounts and roles with `vouch setup aws --discover`. See [SSO-based authentication](#sso-based-authentication) below and [Multi-Account AWS Strategy](/docs/aws-multi-account/) for details.

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

## SSO-based authentication

If your organization uses [AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html), the Vouch CLI can authenticate through your existing SSO session and automatically discover available accounts and roles:

```bash
# Authenticate via IAM Identity Center SSO
vouch aws login

# List available accounts
vouch aws accounts

# List roles in a specific account
vouch aws roles --account 123456789012
```

See [Multi-Account AWS Strategy](/docs/aws-multi-account/) for details on role chaining and auto-discovery across multiple accounts.

---

## Console access

Open the AWS Management Console directly from the CLI without entering credentials in a browser:

```bash
vouch aws console
```

This uses your active Vouch session to obtain temporary STS credentials, exchanges them for a federation sign-in token, and opens the console in your default browser. Pass `--role` to specify a role, or omit it to use the role from your configured AWS profile.

---

## AI agent safety

When `vouch credential aws` runs inside an AI coding agent, Vouch automatically restricts the returned credentials to read-only access. No configuration is required -- the CLI detects the agent environment and applies the restriction transparently.

### How it works

The CLI checks for environment variables set by popular AI coding agents. When one is detected:

1. The [`ReadOnlyAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html) AWS managed policy is attached as a [session policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_session), which limits the effective permissions to the intersection of the role's policies and `ReadOnlyAccess` -- regardless of what the role itself allows.
2. The `vouch:AccessType=ai` and `vouch:Agent=<name>` session tags are added, so you can identify agent-originated API calls in CloudTrail.

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

When using [IAM role chaining](/docs/aws-multi-account/#option-3----iam-role-chaining) with an AI agent, Vouch applies an additional inline session policy to the management-account hop that restricts it to STS actions only (`sts:AssumeRole`, `sts:TagSession`, `sts:SetSourceIdentity`). The final role hop receives the `ReadOnlyAccess` session policy.

---

## Troubleshooting

### "Not authorized to perform sts:AssumeRoleWithWebIdentity"

- Verify the OIDC provider URL in the IAM trust policy matches `https://us.vouch.sh` exactly (no trailing slash).
- Confirm the `aud` condition matches the client ID registered with the OIDC provider.
- Ensure the trust policy `Action` includes all three required actions: `sts:AssumeRoleWithWebIdentity`, `sts:SetSourceIdentity`, and `sts:TagSession`.

### "Token is expired"

- Run `vouch login` again to refresh your session. OIDC tokens are short-lived by design.

### "Invalid identity token"

- Ensure the OIDC provider in AWS points to the correct Vouch server URL: `https://us.vouch.sh`.
- Verify that the Vouch server's JWKS endpoint (`https://us.vouch.sh/.well-known/jwks.json`) is reachable from the internet (AWS must be able to fetch it).

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
