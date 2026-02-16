---
title: "Replace AWS Access Keys with Short-Lived Credentials"
linkTitle: "AWS"
description: "Stop distributing long-lived AWS access keys. Use OIDC federation to get temporary STS credentials backed by a YubiKey."
weight: 2
subtitle: "Configure AWS IAM to trust Vouch as an OIDC identity provider"
params:
  docsGroup: infra
---

If your team distributes AWS access keys through `~/.aws/credentials`, you have credentials that never expire, are trivially exfiltrated by malware, and leave no trace of who used them. Rotating keys is a manual chore that rarely happens on schedule, and a single compromised laptop exposes keys that work indefinitely.

Vouch eliminates static access keys entirely. You configure AWS to trust Vouch as an OIDC identity provider, and developers get temporary STS credentials -- valid for up to 1 hour -- after authenticating with their YubiKey. Every API call is tied to a verified human identity in CloudTrail. There are no keys to rotate, no credentials on disk, and no shared secrets.

## How Vouch compares to `aws login` and `aws sso login`

AWS provides two built-in CLI authentication commands. Here is how they compare to Vouch:

- **[`aws login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-local-development)** -- New in AWS CLI v2.32+. Opens a browser to authenticate with your AWS Console credentials (IAM user, root, or federated identity) and issues temporary credentials for up to 12 hours. It requires the `SignInLocalDevelopmentAccess` managed policy and only covers AWS -- it does not provide credentials for SSH, GitHub, Docker registries, or other services.

- **[`aws sso login`](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html#command-line-sign-in-sso)** -- Authenticates through AWS IAM Identity Center (formerly AWS SSO). It requires an Identity Center instance and an SSO-configured profile (`aws configure sso`). Like `aws login`, it only covers AWS services.

- **`vouch login`** -- Authenticates with a FIDO2 hardware key (YubiKey) and provides credentials for AWS, SSH, GitHub, Docker registries, Cargo registries, CodeCommit, CodeArtifact, databases, and any OIDC-compatible application -- all from a single session. Every credential is tied to a hardware-verified human identity, and authentication is phishing-resistant by design.

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

---

## Step 1 -- Create the OIDC Provider in AWS (admin)

Before any user can assume a role, an administrator must register the Vouch server as an OIDC identity provider in the target AWS account.

### AWS CLI

> For background on OIDC identity providers in AWS, see [Creating OpenID Connect (OIDC) identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) in the AWS documentation.

```bash
aws iam create-open-id-connect-provider \
  --url "https://{{< instance-url >}}" \
  --client-id-list "https://{{< instance-url >}}"
```

> **Note:** AWS fetches the JWKS from `https://{{< instance-url >}}/.well-known/jwks.json` at runtime to verify token signatures. A `ThumbprintList` is no longer required -- AWS obtains the root CA thumbprint automatically.

### CloudFormation

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

### Terraform

```hcl
resource "aws_iam_openid_connect_provider" "vouch" {
  url            = "https://{{< instance-url >}}"
  client_id_list = ["https://{{< instance-url >}}"]
}
```

---

## Step 2 -- Create an IAM Role (admin)

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
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/{{< instance-url >}}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "{{< instance-url >}}:aud": "{{< instance-url >}}"
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
              Federated: !Sub "arn:aws:iam::${AWS::AccountId}:oidc-provider/{{< instance-url >}}"
            Action: "sts:AssumeRoleWithWebIdentity"
            Condition:
              StringEquals:
                "{{< instance-url >}}:aud": "{{< instance-url >}}"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/ReadOnlyAccess
```

### Terraform

```hcl
data "aws_caller_identity" "current" {}

resource "aws_iam_role" "vouch_developer" {
  name = "VouchDeveloper"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/{{< instance-url >}}"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "{{< instance-url >}}:aud" = "{{< instance-url >}}"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "vouch_developer_readonly" {
  role       = aws_iam_role.vouch_developer.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

---

## Tips for restricting access

### Restrict by email address

Limit role assumption to specific users by adding an email condition to the trust policy:

```json
"Condition": {
  "StringEquals": {
    "{{< instance-url >}}:aud": "{{< instance-url >}}",
    "{{< instance-url >}}:sub": "user@example.com"
  }
}
```

### Restrict by email domain

Allow any user from a specific domain:

```json
"Condition": {
  "StringEquals": {
    "{{< instance-url >}}:aud": "{{< instance-url >}}"
  },
  "StringLike": {
    "{{< instance-url >}}:sub": "*@example.com"
  }
}
```

### Session tags

Vouch sets the following session tags when assuming a role, which you can use in IAM policies for [attribute-based access control (ABAC)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html):

| Tag Key | Value | Example |
|---------|-------|---------|
| `email` | The user's verified email | `alice@example.com` |
| `domain` | The user's organization domain (from the OIDC `hd` claim) | `example.com` |

You can reference these tags in IAM policy conditions using `aws:PrincipalTag`:

```json
{
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/domain": "example.com"
    }
  }
}
```

---

## Step 3 -- Configure the Vouch CLI

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

After setup, any tool that reads AWS profiles will transparently use Vouch credentials.

---

## Step 4 -- Test

Verify that everything is working:

```bash
# Make sure you are logged in
vouch login

# Check your identity
aws sts get-caller-identity --profile vouch
```

You should see output similar to:

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

---

## Troubleshooting

### "Not authorized to perform sts:AssumeRoleWithWebIdentity"

- Verify the OIDC provider URL in the IAM trust policy matches `https://{{< instance-url >}}` exactly (no trailing slash).
- Confirm the `aud` condition matches the client ID registered with the OIDC provider.

### "Token is expired"

- Run `vouch login` again to refresh your session. OIDC tokens are short-lived by design.

### "Invalid identity token"

- Ensure the OIDC provider in AWS points to the correct Vouch server URL: `https://{{< instance-url >}}`.
- Verify that the Vouch server's JWKS endpoint (`https://{{< instance-url >}}/.well-known/jwks.json`) is reachable from the internet (AWS must be able to fetch it).

### Credentials not appearing in the expected profile

- Run `vouch setup aws` again and verify the profile name.
- Check `~/.aws/config` for conflicting profile definitions.

### Permission errors after assuming the role

- The trust policy controls who can assume the role; the permissions policy controls what they can do. Verify the correct permissions policies are attached to the IAM role.
