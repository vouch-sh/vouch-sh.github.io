---
title: "Multi-Account AWS Strategy"
linkTitle: "AWS Multi-Account"
description: "Deploy Vouch OIDC federation across multiple AWS accounts with Organizations, StackSets, and SCPs."
weight: 3
subtitle: "Federate Vouch into dev, staging, and production AWS accounts"
params:
  docsGroup: infra
---

Once your organization has more than one AWS account, you need a strategy for deploying Vouch federation across all of them. This guide covers three approaches -- per-account OIDC providers (via CloudFormation StackSets or Terraform), IAM role chaining through a single hub account, and SCPs to enforce that only Vouch can federate into your accounts.

---

## Architecture

```
AWS Organization (Management Account)
├── Shared Services (CI/CD, logging)
├── Development
├── Staging
└── Production

Each account gets:
  ├── OIDC Provider (trusting us.vouch.sh)
  └── IAM Roles (VouchDeveloper, VouchReadOnly, etc.)
```

Every developer uses the same `vouch login` session. The AWS profile they select determines which account and role they assume.

---

## Option 1 -- CloudFormation StackSets

StackSets deploy a CloudFormation template across all accounts in an AWS Organization (or a subset).

### Template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Vouch OIDC federation (deployed via StackSet)"

Parameters:
  EmailDomain:
    Type: String
    Description: Your Google Workspace domain (e.g. example.com)
  ManagedPolicyArn
    Type: String
    Default: "arn:aws:iam::aws:policy/ReadOnlyAccess"
    Description: "Permissions policy to attach to the role"

Resources:
  VouchOIDCProvider:
    Type: AWS::IAM::OIDCProvider
    Properties:
      Url: !Sub "https://${VouchServerUrl}"
      ClientIdList:
        - !Sub "https://${VouchServerUrl}"

  VouchRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: VouchDeveloper
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub "arn:${AWS::Partition}$:iam::${AWS::AccountId}:oidc-provider/{{< instance-url >}}"
            Action:
              - "sts:AssumeRoleWithWebIdentity"
              - "sts:SetSourceIdentity"
              - "sts:TagSession"
            Condition:
              StringEquals:
                "{{< instance-url >}}:aud": "https://{{< instance-url >}}"
              StringLike:
                "{{< instance-url >}}:sub": !Sub "*@${EmailDomain}"
                "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
      ManagedPolicyArns:
        - !Ref ManagedPolicyArn

Outputs:
  RoleArn:
    Value: !GetAtt VouchRole.Arn
```

### Deploy to all accounts

From the management account:

```bash
aws cloudformation create-stack-set \
  --stack-set-name vouch-federation \
  --template-body file://vouch-stackset.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name vouch-federation \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1
```

With `--auto-deployment Enabled`, new accounts added to the OU automatically receive the Vouch OIDC provider and IAM role.

### Per-account role customization

Use different parameter overrides per account to control permissions:

```bash
# Development: PowerUserAccess
aws cloudformation create-stack-instances \
  --stack-set-name vouch-federation \
  --accounts 111111111111 \
  --regions us-east-1 \
  --parameter-overrides ParameterKey=ManagedPolicyArn,ParameterValue=arn:aws:iam::aws:policy/PowerUserAccess ParameterKey=RoleName,ParameterValue=VouchDeveloper

# Production: ReadOnlyAccess
aws cloudformation create-stack-instances \
  --stack-set-name vouch-federation \
  --accounts 222222222222 \
  --regions us-east-1 \
  --parameter-overrides ParameterKey=ManagedPolicyArn,ParameterValue=arn:aws:iam::aws:policy/ReadOnlyAccess ParameterKey=RoleName,ParameterValue=VouchReadOnly
```

---

## Option 2 -- Terraform

### Module

Create a reusable Terraform module:

```hcl
# modules/vouch-federation/main.tf

variable "vouch_server_url" {
  type    = string
  default = "{{< instance-url >}}"
}

variable "role_name" {
  type    = string
  default = "VouchDeveloper"
}

variable "policy_arns" {
  type    = list(string)
  default = ["arn:aws:iam::aws:policy/ReadOnlyAccess"]
}

variable "allowed_email_pattern" {
  type        = string
  default     = "*@example.com"
  description = "Email pattern to restrict who can assume this role"
}

data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  aws_account_id = data.aws_caller_identity.current.account_id
  aws_partition  = data.aws_partition.current.partition
}

resource "aws_iam_openid_connect_provider" "vouch" {
  url            = "https://${var.vouch_server_url}"
  client_id_list = ["https://${var.vouch_server_url}"]
}

resource "aws_iam_role" "vouch" {
  name = var.role_name

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:${local.aws_partition}:iam::${local.aws_account_id}:oidc-provider/${var.vouch_server_url}"
        }
        Action = [
          "sts:AssumeRoleWithWebIdentity",
          "sts:SetSourceIdentity",
          "sts:TagSession",
        ]
        Condition = {
          StringEquals = {
            "${var.vouch_server_url}:aud" = "https://${var.vouch_server_url}"
          }
          StringLike = {
            "${var.vouch_server_url}:sub" = var.allowed_email_pattern
            "sts:RoleSessionName"         = "$${${var.vouch_server_url}:sub}"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "vouch" {
  count      = length(var.policy_arns)
  role       = aws_iam_role.vouch.name
  policy_arn = var.policy_arns[count.index]
}

output "role_arn" {
  value = aws_iam_role.vouch.arn
}
```

### Usage per account

```hcl
# environments/dev/main.tf
module "vouch" {
  source                = "../../modules/vouch-federation"
  role_name             = "VouchDeveloper"
  policy_arns           = ["arn:aws:iam::aws:policy/PowerUserAccess"]
  allowed_email_pattern = "*@example.com"
}

# environments/prod/main.tf
module "vouch" {
  source                = "../../modules/vouch-federation"
  role_name             = "VouchReadOnly"
  policy_arns           = ["arn:aws:iam::aws:policy/ReadOnlyAccess"]
  allowed_email_pattern = "*@example.com"
}
```

---

## Option 3 -- IAM Role Chaining

In Options 1 and 2, every AWS account has its own OIDC provider and directly trusts Vouch. Role chaining is an alternative where only the **management account** has an OIDC provider. Developers federate into a single management account role, which then assumes roles in member accounts using [`sts:AssumeRole`](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html). The developer's verified email propagates through the chain via [`SourceIdentity`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html).

```
Vouch (OIDC) → Management Account Role → Member Account Role
                 (AssumeRoleWithWebIdentity)   (AssumeRole + SourceIdentity)
```

This pattern reduces the number of OIDC providers to one, simplifies member account IAM, and still provides per-user attribution in CloudTrail across all accounts.

### Management account -- Trust policy

Create a role in the management account that trusts Vouch as an OIDC provider. The three actions allow federation, source identity propagation, and session tagging:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::MANAGEMENT_ACCOUNT_ID:oidc-provider/{{< instance-url >}}"
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

### Management account -- Identity policy

Attach an identity policy to the management account role that allows it to assume roles in member accounts, propagate session tags, and propagate the source identity:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:SetSourceIdentity",
        "sts:TagSession"
      ],
      "Resource": "arn:aws:iam::*:role/VouchAccess",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgId": "o-xxxxxxxxx"
        }
      }
    }
  ]
}
```

> **Tip:** Replace the organization ID (`o-xxxxxxxxx`) in the condition key to only allow access to the AWS Accounts within the AWS Organization.

### Member account -- Trust policy

Each member account has a `VouchAccess` role that trusts the management account role (not Vouch directly). The `aws:SourceIdentity` condition ensures that only requests originating from a verified Vouch user with a matching email domain can assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT_ACCOUNT_ID:role/VouchAccess"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:SetSourceIdentity",
        "sts:TagSession"
      ],
      "Condition": {
        "StringLike": {
          "sts:SourceIdentity": "*@example.com"
        }
      }
    }
  ]
}
```

### How SourceIdentity works

When a developer federates into the management account role, Vouch sets the `SourceIdentity` to their verified email address. This identity propagates through the role chain:

1. `vouch login` -- authenticates with YubiKey.
2. `AssumeRoleWithWebIdentity` -- federates into the management account role; `SourceIdentity` is set to `alice@example.com`.
3. `AssumeRole` -- chains into a member account role; `SourceIdentity` is carried forward.
4. CloudTrail in the member account records `alice@example.com` as the source identity.

This gives you per-user attribution in every account without deploying OIDC providers everywhere.

---

## Restricting federation with SCPs

Use [Service Control Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) to ensure that only the Vouch OIDC provider can federate into your accounts. This prevents anyone from registering a rogue OIDC provider:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonVouchOIDCProviders",
      "Effect": "Deny",
      "Action": [
        "iam:CreateOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "${aws:PrincipalOrgID}"
        }
      }
    }
  ]
}
```

> **Note:** Adjust this SCP to allow your management account or deployment pipeline to manage OIDC providers. The goal is to prevent individual developers from creating unauthorized OIDC providers in member accounts.

---

## Developer setup

Each developer configures a named AWS profile for each account:

```bash
# Development account
vouch setup aws \
  --role arn:aws:iam::111111111111:role/VouchDeveloper \
  --profile vouch-dev

# Staging account
vouch setup aws \
  --role arn:aws:iam::333333333333:role/VouchDeveloper \
  --profile vouch-staging

# Production account (read-only)
vouch setup aws \
  --role arn:aws:iam::222222222222:role/VouchReadOnly \
  --profile vouch-prod
```

This produces the following `~/.aws/config`:

```ini
[profile vouch-dev]
credential_process = vouch credential aws --role arn:aws:iam::111111111111:role/VouchDeveloper

[profile vouch-staging]
credential_process = vouch credential aws --role arn:aws:iam::333333333333:role/VouchDeveloper

[profile vouch-prod]
credential_process = vouch credential aws --role arn:aws:iam::222222222222:role/VouchReadOnly
```

Use profiles per command:

```bash
# Deploy to dev
cdk deploy --profile vouch-dev

# Check production (read-only)
aws s3 ls --profile vouch-prod
```

Or set a default:

```bash
export AWS_PROFILE=vouch-dev
```

### Auto-discovery with SSO

If your organization uses AWS IAM Identity Center, developers can skip the manual per-account configuration entirely:

```bash
# Authenticate with IAM Identity Center
vouch aws login

# See which accounts are available
vouch aws accounts

# See which roles are available in a specific account
vouch aws roles --account 111111111111

# Auto-discover all accounts and roles, configure all profiles at once
vouch setup aws --discover --prefix vouch --region us-east-1
```

The `--discover` flag queries IAM Identity Center for every account and role the developer can access, and writes a `credential_process` profile for each one into `~/.aws/config`. This is especially useful in organizations with many accounts where maintaining per-account setup commands is impractical.

---

## Role design patterns

### By environment

| Account | Role | Policy | Who |
|---|---|---|---|
| Development | `VouchDeveloper` | `PowerUserAccess` | All developers |
| Staging | `VouchDeveloper` | `PowerUserAccess` | All developers |
| Production | `VouchReadOnly` | `ReadOnlyAccess` | All developers |
| Production | `VouchDeployer` | Custom deploy policy | Senior engineers only |

### Restricting production access by email

Add an email condition to the production deployer role's trust policy:

```json
"Condition": {
  "StringEquals": {
    "{{< instance-url >}}:aud": "https://{{< instance-url >}}",
    "{{< instance-url >}}:sub": [
      "alice@example.com",
      "bob@example.com"
    ]
  },
  "StringLike": {
    "sts:RoleSessionName": "${{{< instance-url >}}:sub}"
  }
}
```

Only Alice and Bob can assume the `VouchDeployer` role. Everyone else can still assume `VouchReadOnly`.

---

## Troubleshooting

### "OIDC provider already exists"

The OIDC provider URL must be unique per account. If you already created one manually, the StackSet deployment will fail for that account. Either import the existing resource or delete it before deploying.

### Credentials for the wrong account

If `aws sts get-caller-identity` shows an unexpected account, verify you are using the correct profile:

```bash
aws sts get-caller-identity --profile vouch-dev
```

Check `~/.aws/config` for the correct role ARN in each profile.
