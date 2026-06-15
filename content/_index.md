---
title: "Vouch — Hardware-Backed Developer Credentials"
description: "Open-source credential broker that issues short-lived SSH, AWS, GitHub, and Kubernetes credentials after FIDO2 hardware verification."
sitemap:
  priority: 1.0
hero:
  title: "Vouch: Hardware-Backed Developer Credentials"
  subtitle: "Touch your key. Get credentials for everything. Vouch is an open-source credential broker that issues short-lived SSH keys, AWS sessions, GitHub tokens, and Kubernetes configs after a single FIDO2 hardware verification."
  terminal:
    touch: "🔑 Touch your YubiKey..."
    pin: "Enter PIN: ****"
    authed: "✓ Authenticated as you@company.com"
    session: "✓ Session valid for 8 hours"
    comment: "# just works"
problem:
  title: "Modern developer credentials are broken"
  cards:
    - icon: "🔓"
      title: "Credential sprawl"
      body: "SSH keys from 2019. AWS access keys in plaintext. GitHub PATs that never expire. Every tool has its own long-lived secret."
    - icon: "👻"
      title: "No presence verification"
      body: "Existing MFA verifies devices, not humans. A compromised laptop with cached credentials is indistinguishable from its owner."
    - icon: "🤖"
      title: "AI agents with full access"
      body: "AI coding assistants get your credentials with no scoping, no audit trail, and no way to distinguish human from agent actions."
howItWorks:
  title: "How it works"
  subtitle: "One tap, every credential, all day."
  steps:
    - title: "Touch your YubiKey"
      body: "FIDO2 verification with PIN ensures a human is present. Phishing-resistant by design."
    - title: "Vouch issues credentials"
      body: "Short-lived, scoped, hardware-attested, and bound to your device. SSH certificates, AWS sessions, GitHub tokens."
    - title: "Your tools just work"
      body: "Native integration with SSH, AWS CLI, git, kubectl, docker, and cargo. No wrappers."
integrations:
  title: "Integrations"
  subtitle: "Native support for the tools your team already uses."
  items:
    - url: "/docs/aws/"
      icon: "☁️"
      name: "AWS"
      body: "<code>credential_process</code> for seamless STS federation"
    - url: "/docs/ssh/"
      icon: "🔐"
      name: "SSH"
      body: "Signed certificates, no more authorized_keys"
    - url: "/docs/github/"
      icon: "🐙"
      name: "GitHub"
      body: "Short-lived tokens via git credential helper"
    - url: "/docs/eks/"
      icon: "⎈"
      name: "Kubernetes"
      body: "<code>exec</code> plugin for kubectl and EKS"
    - url: "/docs/docker/"
      icon: "🐳"
      name: "Docker"
      body: "Native credential helper for ECR and GHCR"
    - url: "/docs/cargo/"
      icon: "📦"
      name: "Cargo"
      body: "Private registry authentication"
    - url: "/docs/codeartifact/"
      icon: "📚"
      name: "AWS CodeArtifact"
      body: "Package repository tokens for pip, npm, Cargo"
    - url: "/docs/codecommit/"
      icon: "🗂️"
      name: "AWS CodeCommit"
      body: "Git credential helper for AWS repositories"
agents:
  title: "Give AI agents credentials, not your keys"
  body: "Grant scoped, time-limited credentials to AI coding assistants. Full audit trails cryptographically distinguish human actions from agent actions. Revoke instantly."
  ctaUrl: "/docs/applications/credential-brokering-agents/"
openSource:
  title: "Open source and auditable"
  body: "Vouch is fully open source under the Apache-2.0/MIT dual license. Security tools should be auditable."
  ariaLabel: "Evaluation resources"
  links:
    - url: "/docs/security/"
      label: "Security model"
    - url: "/docs/architecture/"
      label: "Architecture"
    - url: "/docs/threat-model/"
      label: "Threat model"
regions:
  title: "Choose your region"
  items:
    - flag: "🇺🇸"
      name: "United States"
      badge: "Active"
      active: true
    - flag: "🇪🇺"
      name: "Europe"
      badge: "Coming soon"
      active: false
    - flag: "🌏"
      name: "Asia Pacific"
      badge: "Coming soon"
      active: false
---
