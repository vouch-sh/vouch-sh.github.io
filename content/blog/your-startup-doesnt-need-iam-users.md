---
title: "Your Startup Doesn't Need IAM Users"
description: "You just created an AWS account. Every tutorial says 'create an IAM user.' There's a better way — OIDC federation from Google Workspace."
date: 2026-02-17
---

You just created an AWS account. You followed the tutorial. You created an IAM user, generated access keys, pasted them into `~/.aws/credentials`, and ran `aws s3 ls`. It worked. You moved on to building your product.

Six months later, you have five engineers, fifteen IAM users (some of which you forgot you created), access keys that have never been rotated, and credentials scattered across laptops, CI/CD pipelines, and Slack messages. You know this is bad, but fixing it feels like a yak shave.

Here's the thing: **you never needed IAM users in the first place.**

## The IAM user trap

IAM users are the AWS equivalent of shared passwords. They generate long-lived access keys that:

- **Never expire** unless you manually rotate them.
- **Live on disk** in `~/.aws/credentials` where any process can read them.
- **Get shared** via Slack, email, or committed to Git (accidentally or intentionally).
- **Leave no trace** of who actually used them -- CloudTrail shows the IAM user, not the human.
- **Survive offboarding** unless someone remembers to deactivate every key.

AWS themselves say [don't create IAM users for humans](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#bp-users-federation-idp). But every "Getting Started with AWS" tutorial still starts with `aws iam create-user`.

## The alternative: OIDC federation

AWS supports [OpenID Connect (OIDC) federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html). Instead of creating IAM users, you register an OIDC identity provider in your AWS account, create an IAM role that trusts it, and developers authenticate through the identity provider to get temporary credentials.

The credentials are:

- **Short-lived** -- They expire in 1 hour. No rotation needed.
- **In-memory** -- They are never written to a credentials file.
- **Attributable** -- CloudTrail shows `assumed-role/VouchDeveloper/alice@example.com`, not `IAMUser/dev-user-3`.
- **Automatic** -- Developers don't manage keys. They authenticate once and tools pick up credentials transparently.

## Google Workspace is your identity provider

If your startup uses Google Workspace (and most do), you already have an organizational identity system. Everyone has an `@yourcompany.com` email. When someone leaves, their account gets deactivated.

[Vouch](https://vouch.sh) federates Google Workspace into AWS via OIDC. After a one-time setup, developers authenticate with their Google Workspace account + a YubiKey, and get temporary AWS credentials. No IAM users, no access keys, no credential files.

## The setup

### 1. Register the OIDC provider (one-time, admin)

```yaml
# vouch-federation.yaml
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  VouchOIDCProvider:
    Type: AWS::IAM::OIDCProvider
    Properties:
      Url: "https://us.vouch.sh"
      ClientIdList:
        - "https://us.vouch.sh"

  VouchDeveloperRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: VouchDeveloper
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub "arn:aws:iam::${AWS::AccountId}:oidc-provider/us.vouch.sh"
            Action: "sts:AssumeRoleWithWebIdentity"
            Condition:
              StringEquals:
                "us.vouch.sh:aud": "https://us.vouch.sh"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/PowerUserAccess
```

Deploy it:

```bash
aws cloudformation deploy \
  --template-file vouch-federation.yaml \
  --stack-name vouch \
  --capabilities CAPABILITY_NAMED_IAM
```

### 2. Each developer installs Vouch and enrolls

```bash
brew install vouch-sh/tap/vouch
brew services start vouch
vouch enroll --server https://us.vouch.sh
```

### 3. Configure the AWS profile

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

### 4. Log in and use AWS

```bash
vouch login
aws sts get-caller-identity --profile vouch
```

```json
{
  "UserId": "AROA...:alice@yourcompany.com",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@yourcompany.com"
}
```

That's it. No IAM user. No access key. No credentials file.

## What you get for free

The same `vouch login` that gives you AWS credentials also gives you:

- **SSH certificates** -- No more copying public keys to servers.
- **GitHub tokens** -- No more personal access tokens.
- **Docker registry auth** -- ECR and GHCR, no `docker login`.
- **CodeCommit access** -- Git clone without HTTPS credentials.
- **CodeArtifact tokens** -- pip, npm, Cargo without token rotation scripts.

One YubiKey tap, one 8-hour session, every tool works.

## What about IAM Identity Center?

[IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) is AWS's own solution for federated access. It's well-built and appropriate for large organizations with dozens of accounts and complex permission sets. But for a 5-person startup, it requires an AWS Organizations management account, an Identity Center instance, permission sets, and user/group synchronization. It only covers AWS -- you still need separate solutions for SSH, GitHub, and Docker.

If your team is small and you want one tool that covers everything, Vouch is the faster path.

## Offboarding

When someone leaves your company:

1. Deactivate their Google Workspace account (which you were going to do anyway).
2. Their Vouch sessions are revoked (immediately if SCIM is configured).
3. Outstanding AWS credentials expire within 1 hour.
4. SSH certificates expire within 8 hours.

There are no access keys to hunt down, no SSH keys to remove from servers, no GitHub PATs to revoke. The identity system you already manage is the single source of truth.

## Getting started

See the [Vouch for Startups](https://vouch.sh/docs/startups/) guide for the complete walkthrough, including multi-account setup and team onboarding.
