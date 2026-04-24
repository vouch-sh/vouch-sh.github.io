---
title: "Getting Started with Vouch"
linkTitle: "Getting Started"
description: "Install the Vouch CLI, enroll your YubiKey, and replace static secrets with hardware-backed credentials in minutes."
weight: 0
subtitle: "Start using Vouch in under 5 minutes"
sitemap:
  priority: 0.9
params:
  docsGroup: featured
---

Vouch replaces static developer secrets (SSH keys, AWS access keys, GitHub PATs) with short-lived credentials derived from a [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) hardware key assertion. This guide walks you through installing the CLI, enrolling your YubiKey, and performing your first login.

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

You can manage your enrolled security keys at any time from the Vouch dashboard:

![Security Keys page showing an enrolled YubiKey](/images/admin/security-keys.png)

<div class="checkpoint">
<p><strong>You are done with enrollment when...</strong></p>
<ul>
  <li><code>vouch enroll</code> finishes without errors.</li>
  <li>Your YubiKey appears on the dashboard security keys page.</li>
  <li>Future <code>vouch</code> commands remember the server URL without another <code>--server</code> flag.</li>
</ul>
</div>

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

<div class="checkpoint">
<p><strong>You are done with login when...</strong></p>
<ul>
  <li>The command reports that you are authenticated.</li>
  <li><code>vouch status</code> shows an active session and the remaining session time.</li>
  <li>You do not need to touch the YubiKey again until the session expires or policy requires it.</li>
</ul>
</div>

---

## Step 4 -- Use your first credential

With an active session, your tools work without extra wrappers. Try the integration your administrator has already configured:

```
# SSH just works
ssh user@server

# AWS credentials are available through the configured profile
aws sts get-caller-identity --profile vouch

# Git prompts Vouch's credential helper when needed
git ls-remote https://github.com/example/private-repo.git
```

Vouch provides credentials on demand to each tool through the lightweight integrations configured by your organization.

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

These tools read your AWS config, SSH config, Git credential helper, or Docker config -- no additional wrapper commands are needed after setup.

<div class="checkpoint">
<p><strong>You are done with first use when...</strong></p>
<ul>
  <li>At least one tool successfully uses credentials from the active Vouch session.</li>
  <li>AWS commands show your assumed role, SSH accepts your Vouch certificate, or Git uses the Vouch credential helper.</li>
  <li>No long-lived AWS keys, SSH private keys, GitHub PATs, or registry passwords were created for the test.</li>
</ul>
</div>

---

## Step 5 -- Choose your next integration

Before a tool can use Vouch, your organization needs to configure the matching integration. Choose the guide that matches the credential you want to replace next.

<div class="journey-grid">
  <div class="journey-card">
    <h3>Cloud and infrastructure</h3>
    <p>Start here for AWS, servers, Kubernetes, databases, and infrastructure tooling.</p>
    <p><a href="/docs/aws/">AWS</a> · <a href="/docs/ssh/">SSH</a> · <a href="/docs/eks/">EKS</a> · <a href="/docs/kubernetes/">Kubernetes</a> · <a href="/docs/databases/">Databases</a> · <a href="/docs/iac/">IaC</a></p>
  </div>
  <div class="journey-card">
    <h3>Code and packages</h3>
    <p>Use Vouch for source control, containers, package repositories, and AWS developer services.</p>
    <p><a href="/docs/github/">GitHub</a> · <a href="/docs/docker/">Docker</a> · <a href="/docs/codeartifact/">CodeArtifact</a> · <a href="/docs/codecommit/">CodeCommit</a> · <a href="/docs/cargo/">Cargo</a></p>
  </div>
  <div class="journey-card">
    <h3>Organization rollout</h3>
    <p>Set up the hosted dashboard, onboard users, and connect identity lifecycle controls.</p>
    <p><a href="/docs/startups/">Startup setup</a> · <a href="/docs/admin/">Admin dashboard</a> · <a href="/docs/scim/">SCIM</a> · <a href="/docs/saml/">SAML</a> · <a href="/docs/migration/">Migration</a></p>
  </div>
  <div class="journey-card">
    <h3>Security review</h3>
    <p>Evaluate the architecture, credential lifecycle, threat model, and failure modes.</p>
    <p><a href="/docs/security/">Security</a> · <a href="/docs/architecture/">Architecture</a> · <a href="/docs/threat-model/">Threat model</a> · <a href="/docs/availability/">Availability</a></p>
  </div>
</div>

---

## Step 6 -- Onboard your team

Once you have Vouch working for yourself, bring the rest of your team onboard.

### Team enrollment

Each team member installs the CLI and enrolls with the same Vouch server. As long as they authenticate with the same Google Workspace domain, they automatically join your organization:

```bash
# Each team member runs:
brew install vouch-sh/tap/vouch
brew services start vouch
vouch enroll --server https://{{< instance-url >}}
```

No invite codes or admin approval are needed for enrollment. The first person to enroll from a domain becomes the organization owner; everyone else joins as a member.

### Manual user management

For small teams (under 15 people), manual management works well. As organization owner, you can view and manage team members through the admin dashboard:

![Organization Members page showing team members, roles, and status](/images/admin/admin-members.png)

When someone leaves, remove them manually to revoke their access.

### When to adopt SCIM

As your team grows, consider setting up [SCIM provisioning](/docs/scim/) to automate user lifecycle management. SCIM connects your identity provider (Google Workspace, Okta, Azure AD) to Vouch so that:

- New hires are provisioned automatically when added to your IdP.
- Departing employees are de-provisioned instantly when their IdP account is deactivated.
- You never forget to revoke someone's access.

SCIM is recommended for teams of 15+ people, but can be set up at any team size.

---

## What happens when you login

When you run `vouch login`, the following takes place behind the scenes:

1. **FIDO2 assertion** -- The CLI asks your YubiKey to sign a challenge from the Vouch server. This proves possession of the enrolled key and requires both your PIN and a physical touch.
2. **Identity verification** -- The server validates the signed assertion against the public key stored during enrollment.
3. **Credential issuance** -- On success, the server issues a session token and an **SSH certificate** signed by the Vouch CA, valid for 8 hours.
4. **On-demand credentials** -- AWS, Git, Docker, Cargo, and other credentials are obtained on-demand by their respective credential helpers when you use those tools. Each helper exchanges your active session for a short-lived, service-specific credential.
5. **Local caching** -- The CLI stores the session and SSH certificate in memory (via the Vouch agent) so subsequent commands can use them without additional YubiKey interaction.

Because every credential is short-lived and bound to a hardware key, there are no long-lived secrets on disk that can be stolen or leaked.

---

## Related guides

- [AWS Integration](/docs/aws/) -- Federate into AWS with OIDC for temporary STS credentials.
- [SSH Certificates](/docs/ssh/) -- Connect to servers using short-lived SSH certificates.
- [GitHub Integration](/docs/github/) -- Access private repositories with short-lived tokens.
- [Security Model](/docs/security/) -- How Vouch protects credentials at every layer.
- [FAQ](/docs/faq/) -- Common questions about supported hardware, session behavior, and platform support.
