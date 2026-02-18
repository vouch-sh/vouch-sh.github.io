---
title: "Google Workspace to AWS in 5 Minutes"
description: "Your team already has Google Workspace. Here's how to federate it into AWS without IAM Identity Center, IAM users, or access keys."
date: 2026-02-07
---

Your startup runs on Google Workspace. Email, calendar, docs, drive -- all tied to `@yourcompany.com` accounts. When someone joins, they get a Google Workspace account. When someone leaves, it gets deactivated.

Your startup also uses AWS. And here's where it gets messy.

AWS doesn't know about your Google Workspace domain. It has its own identity system (IAM) with its own users, its own credentials, and its own lifecycle. So now you're managing two identity systems: Google Workspace for productivity tools, and IAM for infrastructure. Onboarding means creating accounts in both. Offboarding means remembering to deactivate accounts in both. Credentials leak in the gap between them.

**There is a better way.** You can federate your Google Workspace identity directly into AWS, so that `@yourcompany.com` is the only identity your team needs for everything -- email, docs, and AWS.

## The options

### Option 1: IAM Users (don't)

Create an IAM user for each developer. Generate access keys. Paste them into `~/.aws/credentials`. Hope nobody commits them to Git.

This is what most tutorials teach. It's also what [AWS explicitly recommends against](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#bp-users-federation-idp) for human access.

### Option 2: IAM Identity Center (enterprise-grade)

Set up an AWS Organizations management account. Deploy an IAM Identity Center instance. Configure an external identity provider (Google Workspace, Okta, or Azure AD). Create permission sets. Assign users and groups.

This is the right solution for a 200-person company with 50 AWS accounts. For a 5-person startup with one account, it's overkill. It takes a full day to set up correctly, and it only covers AWS -- you still need separate solutions for SSH, GitHub, Docker, and package registries.

### Option 3: AWS Builder ID (individual identity)

AWS Builder ID provides individual developer identity for AWS services. But it's **individual** identity, not **organizational** identity. Builder ID doesn't know about your Google Workspace domain. It can't enforce "only people who work at my company can access my AWS account." It's designed for AWS community participation and personal development, not team access management.

### Option 4: Vouch (the bridge)

Vouch federates your Google Workspace domain into AWS using OIDC. Developers authenticate with their `@yourcompany.com` account + a YubiKey, and get temporary AWS credentials. One tool, one login, every service works.

## The 5-minute setup

### Minute 1: Install Vouch

```bash
brew install vouch-sh/tap/vouch
brew services start vouch
```

### Minute 2: Enroll your YubiKey

```bash
vouch enroll --server https://us.vouch.sh
```

This opens a browser where you sign in with your Google Workspace account and register your YubiKey. The first person from your domain becomes the organization owner.

### Minute 3: Deploy the CloudFormation template

```bash
aws cloudformation deploy \
  --template-file vouch-federation.yaml \
  --stack-name vouch \
  --capabilities CAPABILITY_NAMED_IAM
```

The template (two resources -- an OIDC provider and an IAM role) is available in the [Startups guide](https://vouch.sh/docs/startups/).

### Minute 4: Configure your AWS profile

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

### Minute 5: Log in and test

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

Done. Your Google Workspace identity is now your AWS identity.

## What else works

The same `vouch login` session provides credentials for every developer tool:

| Tool | What Vouch provides |
|---|---|
| `aws` CLI / SDKs | Temporary STS credentials (1-hour lifetime) |
| `ssh` | Short-lived SSH certificates (8-hour lifetime) |
| `git push` (GitHub) | GitHub installation tokens (1-hour lifetime) |
| `docker push` (ECR) | ECR authorization tokens |
| `git push` (CodeCommit) | SigV4-signed Git HTTP requests |
| `cargo build` / `pip install` | CodeArtifact authorization tokens |
| `kubectl` (EKS) | Kubernetes authentication tokens |

One YubiKey tap at the start of the day. Everything works for 8 hours.

## Team onboarding

When a new engineer joins:

1. They get a Google Workspace `@yourcompany.com` account (which you were doing anyway).
2. They install Vouch and enroll their YubiKey.
3. They run `vouch setup aws` with the role ARN.
4. They run `vouch login` and start working.

No IAM user to create. No access keys to distribute. No SSH keys to add to servers.

## Offboarding

When someone leaves:

1. Deactivate their Google Workspace account (which you were doing anyway).
2. If [SCIM](https://vouch.sh/docs/scim/) is configured, their Vouch sessions are revoked automatically.
3. Outstanding credentials expire on their own -- AWS within 1 hour, SSH within 8 hours.

There are no static credentials to hunt down and revoke.

## The key insight

AWS has a terrible onboarding experience for startups. Credit card upfront, 200+ services, IAM complexity. It's why so many startups choose Vercel, Netlify, or GCP for their first infrastructure.

But startups that choose AWS don't need IAM users or IAM Identity Center. They already have an organizational identity system: **Google Workspace**. The missing piece is a bridge between Google Workspace and AWS.

Vouch is that bridge.

## Getting started

See the complete [Vouch for Startups](https://vouch.sh/docs/startups/) guide for multi-account setup, team scaling, and integration details.
