---
title: "About"
description: "Vouch is a hardware-backed authentication system that issues short-lived developer credentials after FIDO2 verification. No credential is ever issued without proof of human presence."
layout: "single"
---

## What is Vouch

Vouch is a hardware-backed authentication system for developer infrastructure. It replaces long-lived secrets -- SSH keys, AWS access keys, GitHub tokens, Docker credentials -- with short-lived, cryptographically attested credentials. No credential is ever issued without proof of human presence via a [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) security key.

If your team spends time rotating AWS keys, copying SSH public keys to servers, managing GitHub PATs, or running `aws ecr get-login-password` cron jobs, Vouch eliminates all of it with a single YubiKey tap each morning.

## How it works

1. **Sign in** through your organization's identity provider (SSO).
2. **Register a security key** (one-time enrollment of a YubiKey or compatible FIDO2 key).
3. **Tap your key** each workday to get 8 hours of credentials for every integrated service.

After a single `vouch login`, credential helpers for SSH, AWS, GitHub, EKS, Docker, Cargo, AWS CodeArtifact, and AWS CodeCommit provide tokens on demand -- transparently and without any long-lived secrets on disk.

## Security model

- **Physical hardware key + PIN** -- Every credential issuance requires a FIDO2 assertion: physical touch of the security key and knowledge of the PIN. Software alone cannot obtain credentials.
- **Phishing-resistant** -- FIDO2 credentials are cryptographically bound to the Vouch server origin. They cannot be replayed against a different site.
- **Short-lived credentials** -- All issued credentials (SSH certificates, OIDC tokens, AWS STS sessions) expire after a maximum of 8 hours. There is nothing to revoke and nothing to rotate.
- **Full audit trail** -- Every credential issuance is logged with the authenticated identity, hardware key attestation, and timestamp.

## Integrations

Vouch provides native credential helpers for:

- **[SSH](/docs/ssh/)** -- Short-lived certificates signed by your organization's CA
- **[AWS](/docs/aws/)** -- STS credentials via OIDC federation
- **[GitHub](/docs/github/)** -- Short-lived repository access tokens
- **[Amazon EKS](/docs/eks/)** -- Kubernetes authentication via IAM
- **[Docker](/docs/docker/)** -- Container registry authentication (ECR, GHCR)
- **[Cargo](/docs/cargo/)** -- Private Cargo registry authentication
- **[AWS CodeArtifact](/docs/codeartifact/)** -- Package repository authentication
- **[AWS CodeCommit](/docs/codecommit/)** -- Git repository authentication
- **[AWS Systems Manager Session Manager](/docs/ssm/)** -- Secure shell access through AWS Systems Manager
- **[Database Authentication](/docs/databases/)** -- IAM authentication for RDS, Aurora, and Redshift
- **[Infrastructure as Code](/docs/iac/)** -- CDK, Terraform, SAM, and other IaC tools
- **[CI/CD Integration](/docs/cicd/)** -- Human authorization gates for deployment pipelines
- **[AI Model Access](/docs/bedrock/)** -- Hardware-verified access to Amazon Bedrock
- **[OIDC Applications](/docs/applications/)** -- "Sign in with Vouch" for your own applications

## Open source

The Vouch CLI and agent are open source under the **Apache-2.0 / MIT** dual license. The server source is available under the **BSL 1.1** license, which converts to Apache-2.0 after 2 years. Security tools should be auditable.

- [GitHub Repository](https://github.com/vouch-sh/vouch)

## Company

Vouch is built by **Smoke Turner, LLC**.
