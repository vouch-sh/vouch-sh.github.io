---
title: "Multi-Account AWS Strategy"
linkTitle: "AWS Multi-Account"
description: "Deploy Vouch OIDC federation across multiple AWS accounts with Organizations, StackSets, and SCPs."
weight: 2
subtitle: "Federate Vouch into dev, staging, and production AWS accounts"
params:
  docsGroup: infra
---

Most startups outgrow a single AWS account quickly. By the time you have a dev and production environment, you have two accounts. Add staging, a shared services account for CI/CD, and a logging account, and you have five. Each account needs an OIDC provider and IAM roles that trust Vouch.

This guide covers deploying Vouch OIDC federation across an AWS Organization using CloudFormation StackSets, Terraform modules, and SCPs to enforce that only Vouch can federate into your accounts.

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
  VouchServerUrl:
    Type: String
    Default: "{{< instance-url >}}"

  RoleName:
    Type: String
    Default: "VouchDeveloper"

  ManagedPolicyArn:
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
      RoleName: !Ref RoleName
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub "arn:aws:iam::${AWS::AccountId}:oidc-provider/${VouchServerUrl}"
            Action: "sts:AssumeRoleWithWebIdentity"
            Condition:
              StringEquals:
                !Sub "${VouchServerUrl}:aud": !Sub "https://${VouchServerUrl}"
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

data "aws_caller_identity" "current" {}

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
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${var.vouch_server_url}"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.vouch_server_url}:aud" = "https://${var.vouch_server_url}"
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
  source      = "../../modules/vouch-federation"
  role_name   = "VouchDeveloper"
  policy_arns = ["arn:aws:iam::aws:policy/PowerUserAccess"]
}

# environments/prod/main.tf
module "vouch" {
  source      = "../../modules/vouch-federation"
  role_name   = "VouchReadOnly"
  policy_arns = ["arn:aws:iam::aws:policy/ReadOnlyAccess"]
}
```

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
