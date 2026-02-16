---
title: "Getting Started with Vouch"
linkTitle: "Getting Started"
description: "Install the Vouch CLI, enroll your YubiKey, and replace static secrets with hardware-backed credentials in minutes."
weight: 1
subtitle: "Start using Vouch in under 5 minutes"
params:
  docsGroup: featured
---

Most developer credential systems rely on static secrets: SSH private keys sitting in `~/.ssh`, AWS access keys in `~/.aws/credentials`, GitHub PATs pasted into environment variables. These secrets never expire, are trivially exfiltrated by malware, and have no proof of who used them.

Vouch replaces all of them with credentials derived from a [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) hardware key assertion -- every credential is short-lived, bound to a verified human identity, and logged. This guide walks you through installing the CLI, enrolling your YubiKey, and performing your first login. By the end you will have hardware-backed credentials ready for SSH, AWS, and Git.

## Prerequisites

- A **YubiKey 5 series** (or any compatible FIDO2 security key)
- A **Vouch server instance**, such as https://{{< instance-url >}}

> **Organization ownership:** The first person to log into Vouch from a Google Workspace domain automatically becomes the organization owner. The owner can configure integrations, manage team members, and connect services like GitHub and AWS for the rest of the team.

---

## Step 1 -- Install the CLI

### macOS

Install with Homebrew:

```
brew install vouch-sh/tap/vouch
```

After installing, start the Vouch background service:

```
brew services start vouch
```

### Debian / Ubuntu

```bash
# Import GPG key
curl -fsSL https://packages.vouch.sh/gpg/vouch.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/vouch-archive-keyring.gpg > /dev/null

# Add repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/vouch-archive-keyring.gpg] https://packages.vouch.sh/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/vouch.list > /dev/null

# Install
sudo apt-get update && sudo apt-get install -y vouch
```

### Fedora / RHEL

```bash
sudo tee /etc/yum.repos.d/vouch.repo << 'EOF'
[vouch]
name=Vouch
baseurl=https://packages.vouch.sh/rpm/$basearch/
gpgcheck=1
gpgkey=https://packages.vouch.sh/gpg/vouch.asc
enabled=1
EOF

sudo dnf install -y vouch
```

### Windows

Windows support is limited. Download the latest binary from the [GitHub releases](https://github.com/vouch-sh/vouch/releases) page.

> **Note:** The SSH agent and SSH integration are not available on Windows. Only basic authentication and credential exchange commands are supported: `enroll`, `login`, `credential aws`, and `credential github`.

### Verify the installation

After installing, confirm the CLI is available:

```
vouch --version
```

---

## Step 2 -- Enroll your YubiKey

Enrollment registers your YubiKey with the Vouch server and links it to your identity. You only need to do this once per key.

```
vouch enroll --server https://{{< instance-url >}}
```

This command will:

1. Display a URL and a one-time code in your terminal (using the [RFC 8628 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628) flow).
2. Open the URL in your browser (or you can navigate to it manually) and enter the one-time code.
3. Ask you to verify your identity through your organization's SSO provider.
4. Prompt you to register your YubiKey as a FIDO2 credential.
5. Set a PIN on the YubiKey if one has not been configured already.
6. Save the server configuration locally so future commands know where to authenticate.

Once enrollment completes, the CLI prints a confirmation and you are ready to log in.

---

## Step 3 -- Daily login

Each workday begins with a single `vouch login`. This authenticates you with your YubiKey and provisions short-lived credentials that last for 8 hours.

```
vouch login
Enter PIN: ****
Touch your YubiKey...
Authenticated for 8 hours
```

That is it. After login, every integration -- SSH, AWS, Git -- uses the session credentials automatically. When the 8-hour window expires, run `vouch login` again.

---

## Step 4 -- Set up integrations

Before your tools can use Vouch credentials, your organization needs to configure the relevant integrations on the Vouch server. Work with your administrator to enable the services you need:

- **[AWS Integration](/docs/aws/)** -- Federate into AWS with OIDC and assume IAM roles using short-lived STS credentials.
- **[SSH Certificates](/docs/ssh/)** -- Connect to servers using Vouch-signed SSH certificates instead of static keys.
- **[Amazon EKS](/docs/eks/)** -- Authenticate to Kubernetes clusters running on EKS.
- **[GitHub Integration](/docs/github/)** -- Access private GitHub repositories using short-lived tokens.
- **[Docker Registries](/docs/docker/)** -- Authenticate to container registries like ECR and GHCR.
- **[AWS CodeArtifact](/docs/codeartifact/)** -- Authenticate to CodeArtifact package repositories.
- **[AWS CodeCommit](/docs/codecommit/)** -- Authenticate to CodeCommit Git repositories.
- **[Cargo Integration](/docs/cargo/)** -- Authenticate to private Cargo registries.
- **[SSM Session Manager](/docs/ssm/)** -- Connect to EC2 instances through AWS Systems Manager.
- **[Database Authentication](/docs/databases/)** -- Connect to RDS, Aurora, and Redshift with IAM authentication.
- **[Infrastructure as Code](/docs/iac/)** -- Use CDK, Terraform, SAM, and other IaC tools.
- **[CI/CD Integration](/docs/cicd/)** -- Add human authorization gates to deployments.
- **[AI Model Access](/docs/bedrock/)** -- Hardware-verified access to Amazon Bedrock.

---

## Step 5 -- Use SSH, AWS, and Git

With an active session, your tools work without any extra flags or configuration:

```
# SSH just works
ssh user@server

# AWS credentials available
aws s3 ls --profile vouch
```

Vouch provides credentials on demand to each tool through the lightweight integrations configured in Step 4.

### What just started working?

One YubiKey tap gives you credentials that cascade across your entire toolchain:

| Command | Service |
|---|---|
| `ssh` | Servers (certificate auth) |
| `git push` | GitHub |
| `aws s3 ls` | AWS CLI |
| `cdk deploy` | Infrastructure as Code |
| `terraform apply` | Infrastructure as Code |
| `docker push` | ECR / GHCR |
| `helm push` | OCI Charts |
| `kubectl` | EKS |

These tools read your AWS config or Docker config -- no additional setup beyond the integrations in Step 4.

---

## What happens when you login

When you run `vouch login`, the following takes place behind the scenes:

1. **FIDO2 assertion** -- The CLI asks your YubiKey to sign a challenge from the Vouch server. This proves possession of the enrolled key and requires both your PIN and a physical touch.
2. **Identity verification** -- The server validates the signed assertion against the public key stored during enrollment.
3. **Credential issuance** -- On success, the server issues a session token and an **SSH certificate** signed by the Vouch CA, valid for 8 hours.
4. **On-demand credentials** -- AWS, Git, Docker, Cargo, and other credentials are obtained on-demand by their respective credential helpers when you use those tools. Each helper exchanges your active session for a short-lived, service-specific credential.
5. **Local caching** -- The CLI stores the session and SSH certificate in memory (via the Vouch agent) so subsequent commands can use them without additional YubiKey interaction.

Because every credential is short-lived and bound to a hardware key, there are no long-lived secrets on disk that can be stolen or leaked.


