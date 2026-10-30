---
title: "Getting Started with Vouch"
linkTitle: "Getting Started"
description: "Install the Vouch CLI, enroll your YubiKey, and replace static secrets with hardware-backed credentials."
weight: 0
subtitle: "Install the CLI, enroll a YubiKey, and start your first session"
sitemap:
  priority: 0.9
params:
  docsGroup: featured
---

Vouch replaces static developer secrets (SSH keys, AWS access keys, GitHub PATs) with short-lived credentials derived from a [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) hardware key assertion. This guide walks you through installing the CLI, enrolling your YubiKey, and performing your first login.

## Prerequisites

- A **YubiKey 5 series** (or any compatible FIDO2 security key)
- A **Vouch server instance**, such as https://{{< instance-url >}}

> **Setting up Vouch for your company?** New AWS account: follow [Startups](/docs/startups/). Existing AWS accounts: follow [Team Rollout](/docs/rollout/). Developers: continue below.

---

## Step 1 -- Install the CLI

{{< tabs >}}
{{< tab "macOS" >}}
Install with Homebrew:

```
brew install vouch-sh/tap/vouch
```

After installing, start the Vouch background service:

```
brew services start vouch
```
{{< /tab >}}
{{< tab "Debian / Ubuntu" >}}
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
{{< /tab >}}
{{< tab "Fedora / RHEL" >}}
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
{{< /tab >}}
{{< tab "Windows" >}}
Install with [winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/):

```
winget install SmokeTurner.Vouch
```

> **Note:** Windows support is limited. The SSH agent and SSH integration are not available on Windows. Only basic authentication and credential exchange commands are supported: `enroll`, `login`, `credential aws`, and `credential github`.
{{< /tab >}}
{{< /tabs >}}

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

If your account already has a registered key — for example, you are enrolling a backup YubiKey — the browser flow asks you to verify with the existing key before the new one is registered.

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

Run the command for the integration your admin configured:

```
# SSH authenticates with your Vouch certificate
ssh user@server

# AWS credentials are available through the configured profile
aws sts get-caller-identity --profile vouch

# Git prompts Vouch's credential helper when needed
git ls-remote https://github.com/example/private-repo.git
```

<div class="checkpoint">
<p><strong>You are done with first use when...</strong></p>
<ul>
  <li>At least one tool successfully uses credentials from the active Vouch session.</li>
  <li>AWS commands show your assumed role, SSH accepts your Vouch certificate, or Git uses the Vouch credential helper.</li>
  <li>No long-lived AWS keys, SSH private keys, GitHub PATs, or registry passwords were created for the test.</li>
</ul>
</div>

---

## Next

Set up the integrations your team uses: [AWS](/docs/aws/), [SSH](/docs/ssh/), or [GitHub](/docs/github/).
