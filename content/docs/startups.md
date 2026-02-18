---
title: "Vouch for Startups"
linkTitle: "Startups"
description: "Skip IAM users, access keys, and IAM Identity Center. Go from Google Workspace to AWS in minutes with OIDC federation."
weight: 0
subtitle: "The fastest path from a new AWS account to a secure team setup"
params:
  docsGroup: featured
---

You just created an AWS account. Every tutorial says "create an IAM user." Don't.

IAM users come with long-lived access keys that never expire, get committed to Git, leaked in logs, and compromised by malware. Rotating them is a manual chore. When someone leaves, you have to hunt down every key they ever created. And none of this is necessary -- AWS supports OIDC federation, which means you can authenticate with the identity system your team already uses.

If your team uses **Google Workspace**, Vouch bridges it directly into AWS. One `vouch login` gives every developer short-lived credentials for AWS, SSH, GitHub, Docker registries, and more -- all tied to their Google Workspace identity, all backed by a hardware key.

---

## What you get

After following this guide, your team will have:

- **No IAM users** -- Every developer authenticates with their Google Workspace account + YubiKey.
- **No access keys** -- AWS credentials are temporary (1 hour) and never written to disk.
- **No credential files** -- No `~/.aws/credentials`, no SSH keys to distribute, no GitHub PATs.
- **Instant offboarding** -- When someone's Google Workspace account is deactivated, their AWS access ends immediately.
- **Full audit trail** -- Every AWS API call in CloudTrail shows which developer made it.

---

## Why not IAM Identity Center?

[AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) (formerly AWS SSO) is AWS's own solution for federated access. It's a good product, but it's designed for enterprises with dozens of accounts and hundreds of users. For a startup:

| Consideration | IAM Identity Center | Vouch |
|---|---|---|
| **Setup complexity** | Requires an AWS Organizations management account, an Identity Center instance, permission sets, and user/group sync | Deploy one CloudFormation template and run `vouch setup aws` |
| **Scope** | AWS only | AWS + SSH + GitHub + Docker + CodeCommit + CodeArtifact + databases + more |
| **Authentication** | Browser-based SSO (MFA depends on IdP config) | FIDO2 hardware key (phishing-resistant by design) |
| **Team size sweet spot** | 20+ people across multiple accounts | 2--50 people |
| **Credential type** | Session credentials via `aws sso login` | Session credentials via `vouch login` |

If you have a large organization with complex permission requirements across many AWS accounts, IAM Identity Center is the right choice. If you are a startup that wants secure AWS access without the overhead, Vouch gets you there faster.

## Why not AWS Builder ID?

[AWS Builder ID](https://docs.aws.amazon.com/signin/latest/userguide/sign-in-aws_builder_id.html) provides individual developer identity for AWS services. The key difference: Builder ID is **individual** identity, not **organizational** identity. It does not know about your Google Workspace domain, your team structure, or your offboarding process. You cannot restrict AWS access to "people who work at my company" using Builder ID alone.

Vouch federates your organization's identity (Google Workspace domain) into AWS, so access is tied to employment by design.

---

## Step 1 -- Get YubiKeys for the team

Order [YubiKey 5 series](https://www.yubico.com/products/yubikey-5-overview/) keys for each team member. Any FIDO2-compatible security key works, but YubiKey 5 series is recommended.

---

## Step 2 -- Enroll the team

The first person to log into Vouch from your Google Workspace domain becomes the organization owner.

**Owner enrollment:**

```bash
# Install the CLI
brew install vouch-sh/tap/vouch
brew services start vouch

# Enroll (first person becomes the org owner)
vouch enroll --server https://{{< instance-url >}}
```

**Team member enrollment:**

Each team member installs the CLI and enrolls with the same server:

```bash
brew install vouch-sh/tap/vouch
brew services start vouch
vouch enroll --server https://{{< instance-url >}}
```

As long as they authenticate with the same Google Workspace domain, they join the same organization. No invite codes or admin approval needed for initial enrollment.

---

## Step 3 -- Deploy the AWS OIDC provider

Deploy a single CloudFormation template to register Vouch as an OIDC identity provider in your AWS account and create an IAM role for your team:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Vouch OIDC Federation — one-click setup for startups"

Parameters:
  VouchServerUrl:
    Type: String
    Default: "{{< instance-url >}}"
    Description: "Vouch server hostname (without https://)"

Resources:
  VouchOIDCProvider:
    Type: AWS::IAM::OIDCProvider
    Properties:
      Url: !Sub "https://${VouchServerUrl}"
      ClientIdList:
        - !Sub "https://${VouchServerUrl}"

  VouchDeveloperRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: VouchDeveloper
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
        - arn:aws:iam::aws:policy/PowerUserAccess

Outputs:
  RoleArn:
    Value: !GetAtt VouchDeveloperRole.Arn
    Description: "Use this ARN with 'vouch setup aws --role <ARN>'"
```

Deploy it:

```bash
aws cloudformation deploy \
  --template-file vouch-startup.yaml \
  --stack-name vouch-federation \
  --capabilities CAPABILITY_NAMED_IAM
```

> **Note:** This template uses `PowerUserAccess` as a starting point. Adjust the managed policy to match your team's needs. See the [AWS documentation](/docs/aws/#tips-for-restricting-access) for examples of restricting access by email address or domain.

---

## Step 4 -- Configure each developer's CLI

Each developer runs one command to configure their AWS profile:

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

Replace the account ID with your own (printed in the CloudFormation stack outputs).

---

## Step 5 -- Log in and verify

```bash
vouch login
aws sts get-caller-identity --profile vouch
```

You should see your email address in the assumed role ARN:

```json
{
  "UserId": "AROA...:alice@yourcompany.com",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@yourcompany.com"
}
```

From here, `aws s3 ls --profile vouch`, `cdk deploy`, `terraform apply`, and any other AWS tool works with no additional setup.

---

## Step 6 -- Add more integrations

With the same `vouch login` session, configure the rest of your toolchain:

| Integration | Setup | Docs |
|---|---|---|
| SSH certificates | Automatic after login | [SSH](/docs/ssh/) |
| GitHub | `vouch setup github` | [GitHub](/docs/github/) |
| Docker (ECR) | `vouch setup docker` | [Docker](/docs/docker/) |
| CodeCommit | `vouch setup codecommit` | [CodeCommit](/docs/codecommit/) |
| CodeArtifact | `vouch setup codeartifact` | [CodeArtifact](/docs/codeartifact/) |
| EKS | `vouch setup eks` | [EKS](/docs/eks/) |

Each integration takes one command. After setup, every tool uses the same session -- one YubiKey tap covers the entire developer toolchain.

---

## What happens when someone leaves

This is where the investment pays off:

1. **Deactivate their Google Workspace account** (which you were going to do anyway).
2. **If SCIM is configured:** Vouch automatically revokes their active sessions. No manual steps needed. See [SCIM Provisioning](/docs/scim/) for setup.
3. **If SCIM is not configured:** Remove the user manually from the Vouch server. Their active session is revoked immediately.
4. **Outstanding credentials expire on their own** -- AWS STS credentials within 1 hour, SSH certificates within 8 hours.

There are no access keys to hunt down, no SSH keys to remove from servers, no GitHub PATs to revoke. The identity system you already manage (Google Workspace) is the single source of truth.

---

## Scaling up

As your team grows:

- **5--15 people:** Manual user management works fine. SCIM is optional.
- **15--50 people:** Set up [SCIM provisioning](/docs/scim/) to automate onboarding and offboarding with Google Workspace.
- **Multiple AWS accounts:** See [Multi-Account AWS Strategy](/docs/aws-multi-account/) for deploying the OIDC provider across dev/staging/prod accounts.
- **Compliance requirements:** See [Security](/docs/security/) for the full threat model and AWS Well-Architected alignment.
