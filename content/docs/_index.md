---
title: "Documentation"
description: "Set up and use Vouch for hardware-backed SSH, AWS, GitHub, Docker, Kubernetes, and Cargo credentials with your YubiKey."
---

Welcome to the Vouch documentation. These guides walk you through installing the CLI, enrolling your YubiKey, and integrating Vouch with the services your team relies on every day -- AWS, SSH, GitHub, Docker, Kubernetes, and more.

Vouch replaces long-lived secrets with short-lived, hardware-backed credentials. After a single `vouch login` authenticated by your YubiKey, you get up to 8 hours of seamless access to cloud infrastructure, remote servers, and Git forges without passwords, static keys, or shared secrets.

If you are spending time rotating AWS access keys, copying SSH public keys to servers, or managing GitHub PATs across your team, Vouch eliminates all of that. One YubiKey tap replaces a dozen credential workflows. Each guide below explains the relevant concepts (OIDC federation, SSH certificate authorities, SCIM provisioning) as it goes -- no prior knowledge required.

## Where to start

- **[Getting Started](/docs/getting-started/)** -- Install the CLI, enroll your YubiKey, and authenticate for the first time.
- **[AWS Integration](/docs/aws/)** -- Configure AWS IAM to trust Vouch as an OIDC identity provider for short-lived STS credentials.
- **[SSH Certificates](/docs/ssh/)** -- Configure SSH servers to trust Vouch certificates for passwordless authentication.
- **[Amazon EKS](/docs/eks/)** -- Authenticate to EKS clusters using AWS IAM and EKS Access Entries.
- **[GitHub Integration](/docs/github/)** -- Access private GitHub repositories using short-lived tokens.
- **[Docker Registries](/docs/docker/)** -- Authenticate to container registries like ECR and GHCR.
- **[AWS CodeArtifact](/docs/codeartifact/)** -- Authenticate to CodeArtifact package repositories for Cargo, pip, and npm.
- **[AWS CodeCommit](/docs/codecommit/)** -- Authenticate to CodeCommit Git repositories.
- **[Cargo Integration](/docs/cargo/)** -- Authenticate to private Cargo registries using Vouch's credential provider.
- **[Applications (OIDC)](/docs/applications/)** -- Add "Sign in with Vouch" to your web application.
- **[SCIM Provisioning](/docs/scim/)** -- Automate user lifecycle management with your identity provider.
- **[SSM Session Manager](/docs/ssm/)** -- Connect to EC2 instances through AWS Systems Manager without opening port 22.
- **[Database Authentication](/docs/databases/)** -- Connect to RDS, Aurora, and Redshift using IAM database authentication.
- **[Infrastructure as Code](/docs/iac/)** -- Use CDK, Terraform, SAM, and other IaC tools with hardware-verified credentials.
- **[CI/CD Integration](/docs/cicd/)** -- Add human authorization gates to deployment pipelines.
- **[AI Model Access (Bedrock)](/docs/bedrock/)** -- Hardware-verified access to Amazon Bedrock with full audit trails.
- **[CLI Reference](/docs/cli-reference/)** -- Complete command reference for the Vouch CLI.
