---
title: "Documentation"
description: "Learn how to set up and use Vouch for hardware-backed developer credentials."
---

Welcome to the Vouch documentation. These guides walk you through installing the CLI, enrolling your YubiKey, and integrating Vouch with the services your team relies on every day -- AWS, SSH, GitHub, and Kubernetes.

Vouch replaces long-lived secrets with short-lived, hardware-backed credentials. After a single `vouch login` authenticated by your YubiKey, you get up to 8 hours of seamless access to cloud infrastructure, remote servers, and Git forges without passwords, static keys, or shared secrets.

## Where to start

- **[Getting Started](/docs/getting-started/)** -- Install the CLI, enroll your YubiKey, and authenticate for the first time.
- **[AWS Integration](/docs/aws/)** -- Configure AWS IAM to trust Vouch as an OIDC identity provider for short-lived STS credentials.
- **[SSH Certificates](/docs/ssh/)** -- Configure SSH servers to trust Vouch certificates for passwordless authentication.
- **[Amazon EKS](/docs/eks/)** -- Authenticate to EKS clusters using AWS IAM and EKS Access Entries.
