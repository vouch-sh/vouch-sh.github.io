---
title: "Multi-Account AWS Strategy"
linkTitle: "AWS Multi-Account"
description: "Deploy Vouch OIDC federation across multiple AWS accounts with Organizations, StackSets, and SCPs."
weight: 2
subtitle: "Federate Vouch into dev, staging, and production AWS accounts"
params:
  docsGroup: aws
---

Multi-account AWS layouts have two models, both anchored in the **management account** where the Vouch OIDC provider lives. **Most teams start with role chaining**; choose Identity Center if you already run it.

- **Role chaining (STS)** -- developers federate into a single management-account "hub" role, and the hub assumes "spoke" roles in member accounts using `sts:AssumeRole`. Covered in Steps 1--3 below.
- **[IAM Identity Center](#aws-iam-identity-center)** -- Vouch is registered as a trusted-token-issuer application in Identity Center, and developers get credentials for the accounts and permission sets they are assigned. Covered in the Identity Center section below.

We don't recommend deploying separate OIDC providers in every account -- it multiplies maintenance, and AWS Organizations exists precisely so you don't have to. One management account plus per-account access covers the same use cases with less surface area.

{{< tldr >}}
- **Prerequisite:** the [OIDC provider is registered](/docs/aws/#step-1----register-the-vouch-oidc-provider) in your management account.
- **Admin, once:** deploy the [hub role](#step-1----deploy-the-hub-role) there, then a [spoke role per member account](#step-2----deploy-spoke-roles-in-member-accounts) via StackSets or Terraform.
- **Each developer:** `vouch setup aws --management-role <HUB_ARN> --role <SPOKE_ARN> --profile vouch-<env>` -- one profile per account.
- On Identity Center? Skip the spokes and [register Vouch as a trusted token issuer](#aws-iam-identity-center) instead.
{{< /tldr >}}

---

## Architecture

```
Vouch (OIDC) ──▶ Management account ──▶ Member account
                  hub role                spoke role
                  (federate in)           (chain into the account)
```

You don't need to follow the STS calls to deploy this: put the OIDC provider and one hub role in the management account, and a spoke role in each member account.

- The Vouch OIDC provider lives in the **management account only**.
- The **hub role** is defined in [Step 1](#step-1--deploy-the-hub-role) -- its identity policy is `sts:AssumeRole` only.
- Each **spoke role** trusts the hub through a plain AWS-principal trust (no OIDC).
- The developer's verified email propagates through the chain as [`SourceIdentity`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html): set to `alice@example.com` at `AssumeRoleWithWebIdentity`, carried forward through each `AssumeRole`, recorded in CloudTrail in every member account.
- All session tags are [transitive](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html#id_session-tags_role-chaining), so conditions like `aws:PrincipalTag/vouch:Domain` and `aws:PrincipalTag/vouch:Email` work in spoke trust policies just as they do in the hub.

Every developer uses the same `vouch login` session. The AWS profile they select determines which spoke role -- and therefore which account -- they assume.

---

## Step 1 -- Deploy the hub role

{{< role admin >}}

Deploy the hub role in your management account. Developers federate into it with `vouch login`, and it can do nothing in this account except assume spoke roles in member accounts.

Its **trust policy** is the shared Vouch OIDC trust -- the `sub` condition uses your email domain:

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
        },
        "Bool": {
          "sts:RoleAuthorizedByIdp": "true"
        }
      }
    }
  ]
}
```

> **Note:** The `sts:RoleAuthorizedByIdp` condition requires the token to be [pinned to the hub role](/docs/aws/#require-role-pinning), which the Vouch CLI does automatically. Do **not** add that condition to spoke roles -- their second hop is a plain SigV4 `sts:AssumeRole` with no OIDC token, so the condition would never match.

Its **identity policy** grants `sts:AssumeRole` only, scoped to the spoke role ARNs you'll deploy in Step 2. The `aws:ResourceOrgID` condition restricts the hub to assuming roles only inside your AWS Organization -- it matches the org of the role being assumed against your own:

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
      "Resource": "arn:aws:iam::*:role/vouch/VouchAccess",
      "Condition": {
        "StringEquals": {
          "aws:ResourceOrgID": "${aws:PrincipalOrgID}"
        }
      }
    }
  ]
}
```

`${aws:PrincipalOrgID}` resolves to your organization automatically, so there's nothing to hand-edit -- and no account IDs to maintain as the org grows. Record the hub role ARN (e.g. `arn:aws:iam::999999999999:role/vouch/VouchAccess`) -- every spoke role's trust policy references it.

---

## Step 2 -- Deploy spoke roles in member accounts

{{< role admin >}}

Each member account needs a `VouchAccess` role that trusts the hub role and grants the actual permissions developers need in that account. The spoke role's trust policy is a plain AWS-principal trust (no OIDC provider, no JWT condition) because the chained `AssumeRole` call comes from a regular IAM role, not from a federated identity. The `aws:SourceIdentity` condition ensures only requests originating from an authenticated Vouch user with a matching email domain can assume the role.

Pick one of the deployment options below. Both produce the same result.

{{< tabs >}}
{{< tab "CloudFormation StackSets" >}}
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
{{< /tab >}}
{{< tab "Terraform" >}}
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
{{< /tab >}}
{{< /tabs >}}

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

{{< role developer >}}

Each developer configures a named AWS profile per account. Point `--role` at the spoke role (`/vouch/VouchAccess`) in the target account and `--management-role` at the hub role; the Vouch CLI stores the hub as an organization anchor and handles the chain through it:

```bash
# Development account
vouch setup aws \
  --management-role arn:aws:iam::999999999999:role/vouch/VouchAccess \
  --role arn:aws:iam::111111111111:role/vouch/VouchAccess \
  --profile vouch-dev

# Staging account
vouch setup aws \
  --management-role arn:aws:iam::999999999999:role/vouch/VouchAccess \
  --role arn:aws:iam::333333333333:role/vouch/VouchAccess \
  --profile vouch-staging

# Production account
vouch setup aws \
  --management-role arn:aws:iam::999999999999:role/vouch/VouchAccess \
  --role arn:aws:iam::222222222222:role/vouch/VouchAccess \
  --profile vouch-prod
```

Running `vouch setup aws` with no flags launches an interactive wizard that captures the management role and target roles for you.

This produces the following `~/.aws/config`. Each profile chains through the hub via `--via`:

```ini
[profile vouch-dev]
credential_process = vouch credential aws --role arn:aws:iam::111111111111:role/vouch/VouchAccess --via arn:aws:iam::999999999999:role/vouch/VouchAccess

[profile vouch-staging]
credential_process = vouch credential aws --role arn:aws:iam::333333333333:role/vouch/VouchAccess --via arn:aws:iam::999999999999:role/vouch/VouchAccess

[profile vouch-prod]
credential_process = vouch credential aws --role arn:aws:iam::222222222222:role/vouch/VouchAccess --via arn:aws:iam::999999999999:role/vouch/VouchAccess
```

> **Note:** Passing `--management-role` stores the hub as an organization anchor and writes `--via` into each profile, so chaining is explicit and unambiguous. Once the anchor is stored, later profiles can drop `--management-role` -- `vouch credential aws --role <spoke-arn>` resolves the management role from your stored organization automatically (pass `--via <management-role-arn>` to pick one when several organizations are configured).

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

---

## AWS IAM Identity Center

Instead of role chaining, you can register Vouch as a **trusted token issuer** in AWS IAM Identity Center. Developers then get credentials for exactly the accounts and permission sets they are assigned in Identity Center -- no spoke roles to deploy. Vouch signs a short-lived RS256 token, exchanges it for an Identity Center access token via `CreateTokenWithIAM`, and calls the SSO portal (`ListAccounts`, `ListAccountRoles`, `GetRoleCredentials`) on the developer's behalf.

This model requires an [organization instance](https://docs.aws.amazon.com/singlesignon/latest/userguide/organization-instances-identity-center.html) of IAM Identity Center and users provisioned so their email matches the Vouch identity (the token `sub`). If you provision Identity Center from the same identity provider Vouch uses, [SCIM](/docs/scim/) keeps them in sync.

> **AI agents cannot use this path.** Permission-set credentials cannot be constrained with a `ReadOnlyAccess` session policy, so Vouch **refuses to issue them to a detected AI coding agent** rather than downscoping. If your workflows include AI agents that need AWS access, use the [role-chaining](#step-1--deploy-the-hub-role) model above, where Vouch enforces read-only automatically.

### IdC Step 1 -- Deploy the management role

{{< role admin >}}

Deploy a management role in the management account using the same [shared Vouch OIDC trust policy](/docs/aws/#shared-trust-policy) as the rest of this guide (`AssumeRoleWithWebIdentity`, with a `*@example.com` `sub` condition). Vouch assumes this role via web identity and uses it to sign the `CreateTokenWithIAM` call. The token for this hop is [pinned to the management role](/docs/aws/#require-role-pinning), so this trust policy can also require `"Bool": {"sts:RoleAuthorizedByIdp": "true"}` like any other web-identity role.

The role needs **no identity policy** for this. Permission to call `CreateTokenWithIAM` is not granted through an identity policy on the role -- instead you attach a **resource policy to the customer managed application** (the *application credentials*) that names this role as the principal allowed to call the action. You apply it in [IdC Step 2](#idc-step-2--register-the-trusted-token-issuer-and-application); it looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::999999999999:role/vouch/VouchAccess"
      },
      "Action": "sso-oauth:CreateTokenWithIAM",
      "Resource": "*"
    }
  ]
}
```

Record the management role ARN -- IdC Step 2 references it as the principal in this resource policy.

### IdC Step 2 -- Register the trusted token issuer and application

{{< role admin >}}

Register Vouch as a **trusted token issuer** and add an OAuth 2.0 [customer managed application](https://docs.aws.amazon.com/singlesignon/latest/userguide/customermanagedapps.html) so it can exchange tokens and read your account assignments. The trusted token issuer, the application, and its account-access scope are managed in Terraform. The JWT-bearer grant (which binds the issuer and sets the `aud` claim) and the application credentials (the resource policy that lets the management role call `CreateTokenWithIAM`) have no Terraform resource yet, so apply those two with the AWS CLI.

#### Terraform

```hcl
data "aws_ssoadmin_instances" "this" {}

locals {
  instance_arn = tolist(data.aws_ssoadmin_instances.this.arns)[0]
}

# Trust Vouch's RS256 tokens. Vouch carries the user's email in `sub`;
# match it to the Identity Center user's email.
resource "aws_ssoadmin_trusted_token_issuer" "vouch" {
  name                      = "Vouch"
  instance_arn              = local.instance_arn
  trusted_token_issuer_type = "OIDC_JWT"

  trusted_token_issuer_configuration {
    oidc_jwt_configuration {
      issuer_url                    = "https://{{< instance-url >}}"
      claim_attribute_path          = "sub"
      identity_store_attribute_path = "emails.value"
      jwks_retrieval_option         = "OPEN_ID_DISCOVERY"
    }
  }
}

# Customer managed OAuth 2.0 application.
resource "aws_ssoadmin_application" "vouch" {
  name                     = "Vouch"
  instance_arn             = local.instance_arn
  application_provider_arn = "arn:aws:sso::aws:applicationProvider/custom"
}

# Let the application list accounts/roles and fetch credentials for the
# authenticated user's own assignments.
resource "aws_ssoadmin_application_access_scope" "vouch" {
  application_arn = aws_ssoadmin_application.vouch.arn
  scope           = "sso:account:access"
}

output "vouch_application_arn" {
  value = aws_ssoadmin_application.vouch.arn
}

output "vouch_trusted_token_issuer_arn" {
  value = aws_ssoadmin_trusted_token_issuer.vouch.arn
}
```

#### Grant and credentials (AWS CLI)

The AWS Terraform provider does not yet expose the JWT-bearer grant or the application authentication method, so set them with `aws sso-admin` after `terraform apply`:

```bash
APP_ARN=$(terraform output -raw vouch_application_arn)
TTI_ARN=$(terraform output -raw vouch_trusted_token_issuer_arn)
MGMT_ROLE_ARN=arn:aws:iam::999999999999:role/vouch/VouchAccess

# Bind the trusted token issuer and require aud = the Vouch issuer.
aws sso-admin put-application-grant \
  --application-arn "$APP_ARN" \
  --grant-type urn:ietf:params:oauth:grant-type:jwt-bearer \
  --grant "{\"JwtBearer\":{\"AuthorizedTokenIssuers\":[{\"TrustedTokenIssuerArn\":\"$TTI_ARN\",\"AuthorizedAudiences\":[\"https://{{< instance-url >}}\"]}]}}"

# Let the management role call CreateTokenWithIAM.
aws sso-admin put-application-authentication-method \
  --application-arn "$APP_ARN" \
  --authentication-method-type IAM \
  --authentication-method "{\"Iam\":{\"ActorPolicy\":{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"$MGMT_ROLE_ARN\"},\"Action\":\"sso-oauth:CreateTokenWithIAM\",\"Resource\":\"*\"}]}}}"
```

Both calls are idempotent -- re-running updates the grant or credentials in place. Record the application ARN (`terraform output -raw vouch_application_arn`); developers pass it to `vouch setup aws`.

#### Console alternative

Prefer the console? In the [IAM Identity Center console](https://console.aws.amazon.com/singlesignon):

1. Under **Settings**, add a trusted token issuer with **Issuer URL** `https://{{< instance-url >}}`, mapping the token's identity claim to the Identity Center user's email.
2. Under **Applications** -> **Customer managed** -> **Add application**, choose **I have an application I want to set up**, then **OAuth 2.0**.
3. On **Specify authentication settings**, select the trusted token issuer and set the **Aud claim** to `https://{{< instance-url >}}` (Vouch sets the audience equal to its issuer).
4. On **Specify application credentials**, name the management role from Step 1 as the principal allowed to call `sso-oauth:CreateTokenWithIAM`.
5. Open the application and turn on **Enable AWS account access** (the `sso:account:access` scope). This must be done from the management or a delegated administrator account. See [Enable AWS account access for customer managed applications](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-account-access-customer-managed-apps.html).

> **Note:** The `sso:account:access` scope grants the application access to every account and permission set assigned to the authenticated user; you cannot scope it to a subset. Access is still bounded by each user's own Identity Center assignments.

### IdC Step 3 -- Discover accounts and permission sets

{{< role developer >}}

With the application registered, developers run a single command to enumerate every account and permission set they are assigned and write one profile per assignment:

```bash
vouch setup aws \
  --management-role arn:aws:iam::999999999999:role/vouch/VouchAccess \
  --identity-center-application arn:aws:sso::999999999999:application/ssoins-1111/apl-2222 \
  --region us-east-1 \
  --discover
```

No `aws sso login` is required -- `vouch login` is the only authentication, because Vouch is the trusted token issuer. The `--discover` run writes profiles named `vouch-<account>-<permission-set>`:

```ini
[profile vouch-production-administratoraccess]
credential_process = vouch credential aws --idc-application arn:aws:sso::999999999999:application/ssoins-1111/apl-2222 --account 222222222222 --permission-set "AdministratorAccess"
output = json
```

Use them like any other profile:

```bash
aws s3 ls --profile vouch-production-administratoraccess
```

Re-run `vouch setup aws --discover` at any time to pick up newly assigned accounts and permission sets; existing profiles are left untouched.

<div class="checkpoint">
<p><strong>You are done with Identity Center setup when...</strong></p>
<ul>
  <li>The management role is named as the principal in the customer managed application's resource policy (<em>application credentials</em>), allowing it to call <code>sso-oauth:CreateTokenWithIAM</code>.</li>
  <li>The trusted token issuer's <strong>Issuer URL</strong> and the application's <strong>Aud claim</strong> are both <code>https://{{< instance-url >}}</code>.</li>
  <li><code>vouch setup aws --discover</code> writes a profile per assignment, and <code>aws sts get-caller-identity</code> against one returns the expected account.</li>
</ul>
</div>

---

## Restricting federation with SCPs

Use [Service Control Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) to lock down who can register or change an OIDC provider, so a developer can't add a rogue identity provider that federates into your accounts. This denies every OIDC-provider change except from your deployment principal:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnauthorizedOIDCProviders",
      "Effect": "Deny",
      "Action": [
        "iam:CreateOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider",
        "iam:UpdateOpenIDConnectProviderThumbprint",
        "iam:AddClientIDToOpenIDConnectProvider",
        "iam:RemoveClientIDFromOpenIDConnectProvider"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/VouchDeploymentRole"
        }
      }
    }
  ]
}
```

> **Note:** Replace `VouchDeploymentRole` with the principal your CloudFormation StackSet, Terraform pipeline, or platform team uses to manage the Vouch OIDC provider -- the same one referenced in [Deny deletion with an SCP](#deny-deletion-with-an-scp). Every other principal, including developers, is then blocked from creating or modifying OIDC providers.

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
