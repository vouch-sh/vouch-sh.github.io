---
title: "Multi-Account AWS Strategy"
linkTitle: "AWS Multi-Account"
description: "Deploy Vouch OIDC federation across multiple AWS accounts with Organizations, StackSets, and SCPs."
weight: 3
subtitle: "Federate Vouch into dev, staging, and production AWS accounts"
params:
  docsGroup: infra
---

Multi-account AWS layouts use **role chaining**: the Vouch OIDC provider lives only in the management account, developers federate into a single management-account "hub" role, and the hub assumes "spoke" roles in member accounts using `sts:AssumeRole`. This page picks up where [AWS / Step 3 / Pattern C](/docs/aws/#pattern-c--stsassumerole-only) leaves off.

We don't recommend deploying separate OIDC providers in every account -- it multiplies maintenance, and AWS Organizations exists precisely so you don't have to. One hub plus per-account spokes covers the same use cases with less surface area.

---

## Architecture

```
Vouch (OIDC) ──▶ Management Account ──▶ Member Account
                  VouchAccess (hub)        VouchAccess (spoke)
                AssumeRoleWithWebIdentity     AssumeRole + SourceIdentity
```

- The Vouch OIDC provider lives in the **management account only**.
- The **hub role** uses [Pattern C](/docs/aws/#pattern-c--stsassumerole-only) -- its identity policy is `sts:AssumeRole` only.
- Each **spoke role** trusts the hub through a plain AWS-principal trust (no OIDC).
- The developer's verified email propagates through the chain as [`SourceIdentity`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html): set to `alice@example.com` at `AssumeRoleWithWebIdentity`, carried forward through each `AssumeRole`, recorded in CloudTrail in every member account.
- All session tags are [transitive](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html#id_session-tags_role-chaining), so conditions like `aws:PrincipalTag/vouch:Domain` and `aws:PrincipalTag/vouch:Email` work in spoke trust policies just as they do in the hub.

Every developer uses the same `vouch login` session. The AWS profile they select determines which spoke role -- and therefore which account -- they assume.

---

## Step 1 -- Deploy the hub role

<span class="role-label">Admin task</span>

Deploy the hub role in your management account using [Pattern C from the AWS guide](/docs/aws/#pattern-c--stsassumerole-only). Two specifics for the hub-and-spoke topology:

- The trust policy's `sub` condition uses your email domain (e.g. `*@example.com`).
- The identity policy's `Resource` must match the spoke role ARNs you'll deploy in Step 2: `arn:aws:iam::*:role/vouch/VouchAccess`.

Replace the `aws:PrincipalOrgId` placeholder with your AWS Organization ID, and record the hub role ARN (e.g. `arn:aws:iam::999999999999:role/vouch/VouchAccess`) -- every spoke role's trust policy references it.

---

## Step 2 -- Deploy spoke roles in member accounts

<span class="role-label">Admin task</span>

Each member account needs a `VouchAccess` role that trusts the hub role and grants the actual permissions developers need in that account. The spoke role's trust policy is a plain AWS-principal trust (no OIDC provider, no JWT condition) because the chained `AssumeRole` call comes from a regular IAM role, not from a federated identity. The `aws:SourceIdentity` condition ensures only requests originating from an authenticated Vouch user with a matching email domain can assume the role.

Pick one of the deployment options below. Both produce the same result.

### Option A -- CloudFormation StackSets

StackSets deploy a single template across every account in your AWS Organization (or a chosen OU).

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Vouch spoke role (deployed via StackSet)"

Parameters:
  ManagementAccountId:
    Type: String
    Description: "Account ID of the AWS Organization management account"
  EmailDomain:
    Type: String
    Description: Your Google Workspace domain (e.g. example.com)
  ManagedPolicyArn:
    Type: String
    Default: "arn:aws:iam::aws:policy/ReadOnlyAccess"
    Description: "Permissions policy to attach to the spoke role"

Resources:
  VouchSpokeRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: VouchAccess
      Path: /vouch/
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub "arn:${AWS::Partition}:iam::${ManagementAccountId}:role/vouch/VouchAccess"
            Action:
              - "sts:AssumeRole"
              - "sts:SetSourceIdentity"
              - "sts:TagSession"
            Condition:
              StringLike:
                "aws:SourceIdentity": !Sub "*@${EmailDomain}"
      ManagedPolicyArns:
        - !Ref ManagedPolicyArn

Outputs:
  RoleArn:
    Value: !GetAtt VouchSpokeRole.Arn
```

Deploy from the management account:

```bash
aws cloudformation create-stack-set \
  --stack-set-name vouch-spokes \
  --template-body file://vouch-spoke.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name vouch-spokes \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1 \
  --parameter-overrides \
    ParameterKey=ManagementAccountId,ParameterValue=999999999999 \
    ParameterKey=EmailDomain,ParameterValue=example.com
```

With `--auto-deployment Enabled`, new accounts added to the OU automatically receive the spoke role.

#### Per-account permissions

Override `ManagedPolicyArn` per account to scope permissions:

```bash
# Development: PowerUserAccess
aws cloudformation create-stack-instances \
  --stack-set-name vouch-spokes \
  --accounts 111111111111 \
  --regions us-east-1 \
  --parameter-overrides \
    ParameterKey=ManagementAccountId,ParameterValue=999999999999 \
    ParameterKey=EmailDomain,ParameterValue=example.com \
    ParameterKey=ManagedPolicyArn,ParameterValue=arn:aws:iam::aws:policy/PowerUserAccess

# Production: ReadOnlyAccess
aws cloudformation create-stack-instances \
  --stack-set-name vouch-spokes \
  --accounts 222222222222 \
  --regions us-east-1 \
  --parameter-overrides \
    ParameterKey=ManagementAccountId,ParameterValue=999999999999 \
    ParameterKey=EmailDomain,ParameterValue=example.com \
    ParameterKey=ManagedPolicyArn,ParameterValue=arn:aws:iam::aws:policy/ReadOnlyAccess
```

### Option B -- Terraform

Create a reusable module for the spoke role:

```hcl
# modules/vouch-spoke/main.tf

variable "management_account_id" {
  type        = string
  description = "AWS Organization management account ID"
}

variable "email_domain" {
  type        = string
  description = "Email domain to allow via SourceIdentity"
}

variable "policy_arns" {
  type    = list(string)
  default = ["arn:aws:iam::aws:policy/ReadOnlyAccess"]
}

data "aws_partition" "current" {}

resource "aws_iam_role" "vouch_spoke" {
  name = "VouchAccess"
  path = "/vouch/"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:${data.aws_partition.current.partition}:iam::${var.management_account_id}:role/vouch/VouchAccess"
        }
        Action = [
          "sts:AssumeRole",
          "sts:SetSourceIdentity",
          "sts:TagSession",
        ]
        Condition = {
          StringLike = {
            "aws:SourceIdentity" = "*@${var.email_domain}"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "vouch_spoke" {
  count      = length(var.policy_arns)
  role       = aws_iam_role.vouch_spoke.name
  policy_arn = var.policy_arns[count.index]
}

output "role_arn" {
  value = aws_iam_role.vouch_spoke.arn
}
```

Per-account usage:

```hcl
# environments/dev/main.tf
module "vouch" {
  source                = "../../modules/vouch-spoke"
  management_account_id = "999999999999"
  email_domain          = "example.com"
  policy_arns           = ["arn:aws:iam::aws:policy/PowerUserAccess"]
}

# environments/prod/main.tf
module "vouch" {
  source                = "../../modules/vouch-spoke"
  management_account_id = "999999999999"
  email_domain          = "example.com"
  policy_arns           = ["arn:aws:iam::aws:policy/ReadOnlyAccess"]
}
```

<div class="checkpoint">
<p><strong>You are done with role deployment when...</strong></p>
<ul>
  <li>The hub role exists in the management account with an <code>sts:AssumeRole</code>-only identity policy.</li>
  <li>Each member account has a <code>/vouch/VouchAccess</code> role whose trust policy lists the hub role as principal.</li>
  <li>From an authenticated Vouch session you can assume the hub and chain into a spoke; CloudTrail in the spoke account records the developer's email as <code>SourceIdentity</code>.</li>
</ul>
</div>

---

## Step 3 -- Configure developer profiles

<span class="role-label">Developer task</span>

Each developer configures a named AWS profile per account. The role ARN points to the spoke role (`/vouch/VouchAccess`) in the target account; the Vouch CLI handles the chain through the hub:

```bash
# Development account
vouch setup aws \
  --role arn:aws:iam::111111111111:role/vouch/VouchAccess \
  --profile vouch-dev

# Staging account
vouch setup aws \
  --role arn:aws:iam::333333333333:role/vouch/VouchAccess \
  --profile vouch-staging

# Production account
vouch setup aws \
  --role arn:aws:iam::222222222222:role/vouch/VouchAccess \
  --profile vouch-prod
```

This produces the following `~/.aws/config`:

```ini
[profile vouch-dev]
credential_process = vouch credential aws --role arn:aws:iam::111111111111:role/vouch/VouchAccess

[profile vouch-staging]
credential_process = vouch credential aws --role arn:aws:iam::333333333333:role/vouch/VouchAccess

[profile vouch-prod]
credential_process = vouch credential aws --role arn:aws:iam::222222222222:role/vouch/VouchAccess
```

Use profiles per command:

```bash
# Deploy to dev
cdk deploy --profile vouch-dev

# Check production
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

> **Skip role chaining entirely?** If your organization runs an IAM Identity Center **organization instance**, Vouch can issue credentials straight from your permission-set assignments -- no hub/spoke roles and no per-account OIDC provider. See [Reach accounts through IAM Identity Center](#reach-accounts-through-iam-identity-center) below.

---

## Reach accounts through IAM Identity Center

Role chaining (above) is one way to reach many accounts. If you already run **AWS IAM Identity Center** with an **organization instance**, there is a second option: Vouch can exchange a hardware-backed session for an Identity Center token and issue credentials directly for any **permission set** the developer is assigned -- no hub/spoke roles, no per-account OIDC provider, and no per-role IAM trust policy. Access is governed entirely by your existing Identity Center assignments.

The same `vouch credential aws` and `vouch setup aws` commands drive this path; it is selected by per-`[sso-session]` configuration (Step C). Under the hood, Vouch mints a short-lived **RS256** token, exchanges it via `sso-oidc:CreateTokenWithIAM` for an Identity Center token, and calls the SSO Portal `GetRoleCredentials` API.

> **Requires an organization instance of Identity Center.** The trusted token issuer used below is available only on Identity Center organization instances, not account instances.

### Step A -- Register the trusted token issuer and application

<span class="role-label">Admin task</span>

Register Vouch as a **trusted token issuer** and create a **customer-managed application** in your Identity Center management account. Terraform covers the trusted token issuer, the application, and its access scope; the jwt-bearer **grant** has no Terraform resource yet, so that one step uses the AWS CLI.

```hcl
data "aws_ssoadmin_instances" "this" {}

locals {
  identity_center_audience = "vouch-identity-center"
}

# Trust Vouch-issued RS256 tokens, mapping the token's email claim to the
# Identity Store user.
resource "aws_ssoadmin_trusted_token_issuer" "vouch" {
  name                      = "vouch"
  instance_arn              = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  trusted_token_issuer_type = "OIDC_JWT"

  trusted_token_issuer_configuration {
    oidc_jwt_configuration {
      issuer_url                    = "https://{{< instance-url >}}"
      claim_attribute_path          = "email"
      identity_store_attribute_path = "emails.value"
      jwks_retrieval_option         = "OPEN_ID_DISCOVERY"
    }
  }
}

# The customer-managed application developers exchange tokens against.
resource "aws_ssoadmin_application" "vouch" {
  name                     = "vouch"
  instance_arn             = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  application_provider_arn = "arn:aws:sso::aws:applicationProvider/custom"
}

# Allow the exchanged token to drive the SSO Portal (ListAccounts /
# ListAccountRoles / GetRoleCredentials).
resource "aws_ssoadmin_application_access_scope" "portal" {
  application_arn = aws_ssoadmin_application.vouch.application_arn
  scope           = "sso:account:access"
}
```

Then authorize the jwt-bearer grant, binding the application to the trusted token issuer and the audience Vouch requests. Replace the audience with the `local.identity_center_audience` value above:

```bash
aws sso-admin put-application-grant \
  --application-arn "$APPLICATION_ARN" \
  --grant-type "urn:ietf:params:oauth:grant-type:jwt-bearer" \
  --grant 'JwtBearer={AuthorizedTokenIssuers=[{TrustedTokenIssuerArn="'"$TTI_ARN"'",AuthorizedAudiences=["vouch-identity-center"]}]}'
```

> **CloudFormation note:** CloudFormation has no trusted-token-issuer resource type, and `AWS::SSO::Application` does not expose the jwt-bearer grant or access scope, so the steps above must use Terraform or the AWS CLI. CloudFormation *can* manage the IAM policy in Step B.

Record the **application ARN** and the **audience** value -- developers need both in Step C.

### Step B -- Allow the management role to exchange tokens

<span class="role-label">Admin task</span>

The Vouch **management role** (the role Vouch federates into via `AssumeRoleWithWebIdentity`, from [Pattern C](/docs/aws/#pattern-c--stsassumerole-only)) performs the SigV4 `CreateTokenWithIAM` call. Grant it that action on the application. This policy is the one piece of the setup that works in both CloudFormation and Terraform.

#### CloudFormation

```yaml
Resources:
  VouchCreateTokenPolicy:
    Type: AWS::IAM::RolePolicy
    Properties:
      RoleName: VouchManagement
      PolicyName: VouchCreateTokenWithIAM
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action: "sso-oauth:CreateTokenWithIAM"
            Resource: "arn:aws:sso::111111111111:application/ssoins-xxxx/apl-xxxx"
```

#### Terraform

```hcl
resource "aws_iam_role_policy" "vouch_create_token" {
  name = "VouchCreateTokenWithIAM"
  role = "VouchManagement"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "sso-oauth:CreateTokenWithIAM"
        Resource = aws_ssoadmin_application.vouch.application_arn
      }
    ]
  })
}
```

<div class="checkpoint">
<p><strong>You are done with the AWS-side setup when...</strong></p>
<ul>
  <li>Vouch is a trusted token issuer pointing at <code>https://{{< instance-url >}}</code>.</li>
  <li>A customer-managed application exists with the <code>sso:account:access</code> scope and a jwt-bearer grant bound to that issuer and your audience.</li>
  <li>The management role may call <code>sso-oauth:CreateTokenWithIAM</code> on the application.</li>
  <li>You have written down the application ARN and the audience value.</li>
</ul>
</div>

### Step C -- Configure the SSO session

<span class="role-label">Developer task</span>

Identity Center connection details (start URL, region, scopes) come from the `[sso-session]` block in `~/.aws/config`, exactly as for `vouch aws login`. Vouch stores the two extra values in `$XDG_CONFIG_HOME/vouch/config.json` (`~/.config/vouch/config.json`), keyed by SSO session name:

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

Both `identity_center_application_arn` and `identity_center_audience` must be set together. When they are, a single `vouch login` is enough -- Vouch performs the exchange for you. If they are omitted, the Identity Center path still works, but you must run `vouch aws login` first so Vouch can use the cached device token instead.

### Step D -- Use it

<span class="role-label">Developer task</span>

`vouch setup aws` writes `~/.aws/config` profiles whose `credential_process` calls `vouch credential aws`. With Identity Center configured, it has three modes:

```bash
# Interactive: pick one account and permission set
vouch setup aws

# Discover: write a profile for every assigned account + permission set
vouch setup aws --discover --region us-east-1
```

Both write the same shape of profile, so you can also add one by hand. In Identity Center mode the `credential_process` passes `--account` and a permission-set **name** (not a role ARN) as `--role`:

```ini
[profile prod-admin]
credential_process = vouch credential aws --sso-session "my-sso" --account 111111111111 --role "AdministratorAccess"
region = us-east-1
```

Test it:

```bash
vouch login   # or `vouch aws login` if you did not configure the token issuer in Step C
aws sts get-caller-identity --profile prod-admin
```

The issued session carries the same [session tags](/docs/aws/#session-tags) (`vouch:Email`, `vouch:Domain`, and the AI-agent tags) as the STS path, so your ABAC conditions and CloudTrail attribution work identically.

---

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

## Protecting Vouch roles from accidental deletion

A common convention is to prefix critical roles with `DO-NOT-DELETE-*`, but that is a social signal, not a control. Tired engineers ignore it; automation never reads it. Use technical guardrails as the real protection, and use names, paths, and tags only as the addressing scheme those guardrails attach to.

### Use an IAM path, not a name prefix

Place Vouch roles under a dedicated IAM [path](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html#identifiers-friendly-names) such as `/vouch/`. Paths are first-class in the role ARN, can be wildcarded in policy `Resource` fields, and don't pollute the role's display name:

```
arn:aws:iam::123456789012:role/vouch/VouchAccess
```

In CloudFormation, add a single line to the role:

```yaml
VouchRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: VouchAccess
    Path: /vouch/
```

In Terraform:

```hcl
resource "aws_iam_role" "vouch" {
  name = var.role_name
  path = "/vouch/"
  # ...
}
```

### Tag for ABAC and inventory

Tag every Vouch-managed resource so an SCP or audit query can find it even if someone forgets the path:

```yaml
Tags:
  - Key: ManagedBy
    Value: Vouch
  - Key: Purpose
    Value: OIDCFederation
```

### Deny deletion with an SCP

This is the actual control. Deny destructive IAM actions against anything in the `/vouch/` path and against the Vouch OIDC provider, with an exception for your deployment principal so legitimate updates can still happen:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectVouchRoles",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRole",
        "iam:DeleteRolePolicy",
        "iam:DetachRolePolicy",
        "iam:UpdateAssumeRolePolicy",
        "iam:DeleteOpenIDConnectProvider"
      ],
      "Resource": [
        "arn:aws:iam::*:role/vouch/*",
        "arn:aws:iam::*:oidc-provider/{{< instance-url >}}"
      ],
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/VouchDeploymentRole"
        }
      }
    }
  ]
}
```

Replace `VouchDeploymentRole` with whatever principal your CloudFormation StackSet, Terraform pipeline, or platform team uses.

### IaC-level protections

Belt-and-suspenders settings in your stack definitions catch the cases where someone bypasses the SCP exception and runs `terraform destroy` or deletes a CloudFormation stack:

- **CloudFormation:** set `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` on the role and OIDC provider resources.
- **Terraform:** add `lifecycle { prevent_destroy = true }` to each resource.
- **StackSets:** enable termination protection on the stack set itself.

### Detective backstop

Even with all of the above, alert on the action so you find out fast if something slips through. An EventBridge rule on `DeleteRole` and `DeleteOpenIDConnectProvider` events filtered to the `/vouch/` path, targeting an SNS topic, gives you a notification within seconds.

---

## Role design patterns

### By environment

The default spoke role is `/vouch/VouchAccess`, scoped per-account by the policy you attach. For higher-privilege production access, deploy a second spoke role with a tighter `aws:SourceIdentity` allowlist:

| Account | Spoke role | Policy | Who can chain in |
|---|---|---|---|
| Development | `vouch/VouchAccess` | `PowerUserAccess` | Anyone in `*@example.com` |
| Staging | `vouch/VouchAccess` | `PowerUserAccess` | Anyone in `*@example.com` |
| Production | `vouch/VouchAccess` | `ReadOnlyAccess` | Anyone in `*@example.com` |
| Production | `vouch/VouchDeploy` | Custom deploy policy | Specific emails only |

### Restricting production access to specific people

Tighten the `aws:SourceIdentity` condition on the production deployer's spoke role:

```json
"Condition": {
  "StringEquals": {
    "aws:SourceIdentity": [
      "alice@example.com",
      "bob@example.com"
    ]
  }
}
```

Only Alice and Bob can chain into `vouch/VouchDeploy`. Everyone else still gets the read-only `vouch/VouchAccess` spoke.

---

## Troubleshooting

### Credentials for the wrong account

If `aws sts get-caller-identity` shows an unexpected account, verify you are using the correct profile:

```bash
aws sts get-caller-identity --profile vouch-dev
```

Check `~/.aws/config` for the correct role ARN in each profile.
