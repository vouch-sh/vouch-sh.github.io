# Getting Started with Vouch

> Install the Vouch CLI, enroll your YubiKey, and replace static secrets with hardware-backed credentials.

Source: https://vouch.sh/docs/getting-started/
Last updated: 2026-10-30

---
Vouch replaces static developer secrets (SSH keys, AWS access keys, GitHub PATs) with short-lived credentials derived from a [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) hardware key assertion. This guide walks you through installing the CLI, enrolling your YubiKey, and performing your first login.

## Prerequisites

- A **YubiKey 5 series** (or any compatible FIDO2 security key)
- A **Vouch server instance**, such as https://us.vouch.sh

&gt; **Setting up Vouch for your company?** No AWS account yet, or an AWS projects account: follow [Startups](/docs/startups/). Standard AWS accounts: follow [Team Rollout](/docs/rollout/). Joining a team that already uses Vouch: continue below.

---

## Step 1 -- Install the CLI


#### macOS

Install with Homebrew:

```
brew install vouch-sh/tap/vouch
```

After installing, start the Vouch background service:

```
brew services start vouch
```

#### Debian / Ubuntu

```bash
# Import GPG key
curl -fsSL https://packages.vouch.sh/gpg/vouch.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/vouch-archive-keyring.gpg &gt; /dev/null

# Add repository
echo &#34;deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/vouch-archive-keyring.gpg] https://packages.vouch.sh/apt stable main&#34; \
  | sudo tee /etc/apt/sources.list.d/vouch.list &gt; /dev/null

# Install
sudo apt-get update &amp;&amp; sudo apt-get install -y vouch
```

#### Fedora / RHEL

```bash
sudo tee /etc/yum.repos.d/vouch.repo &lt;&lt; &#39;EOF&#39;
[vouch]
name=Vouch
baseurl=https://packages.vouch.sh/rpm/$basearch/
gpgcheck=1
gpgkey=https://packages.vouch.sh/gpg/vouch.asc
enabled=1
EOF

sudo dnf install -y vouch
```

#### Windows

Install with [winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/):

```
winget install SmokeTurner.Vouch
```

&gt; **Note:** Windows support is limited. The SSH agent and SSH integration are not available on Windows. Only basic authentication and credential exchange commands are supported: `enroll`, `login`, `credential aws`, and `credential github`.



### Verify the installation

After installing, confirm the CLI is available:

```
vouch --version
```

---

## Step 2 -- Enroll your YubiKey

Enrollment registers your YubiKey with the Vouch server and links it to your identity. You only need to do this once per key.

```
vouch enroll --server https://us.vouch.sh
```

This command will:

1. Display a URL and a one-time code in your terminal (using the [RFC 8628 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628) flow).
2. Open the URL in your browser (or you can navigate to it manually) and enter the one-time code.
3. Ask you to verify your identity through your organization&#39;s SSO provider.
4. Prompt you to register your YubiKey as a FIDO2 credential.
5. Set a PIN on the YubiKey if one has not been configured already.
6. Save the server configuration locally so future commands know where to authenticate.

If your account already has a registered key — for example, you are enrolling a backup YubiKey — the browser flow asks you to verify with the existing key before the new one is registered.

Once enrollment completes, the CLI prints a confirmation and you are ready to log in.

You can manage your enrolled security keys at any time from the Vouch dashboard:

![Security Keys page showing an enrolled YubiKey](/images/admin/security-keys.png)

&lt;div class=&#34;checkpoint&#34;&gt;
&lt;p&gt;&lt;strong&gt;You are done with enrollment when...&lt;/strong&gt;&lt;/p&gt;
&lt;ul&gt;
  &lt;li&gt;&lt;code&gt;vouch enroll&lt;/code&gt; finishes without errors.&lt;/li&gt;
  &lt;li&gt;Your YubiKey appears on the dashboard security keys page.&lt;/li&gt;
  &lt;li&gt;Future &lt;code&gt;vouch&lt;/code&gt; commands remember the server URL without another &lt;code&gt;--server&lt;/code&gt; flag.&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;

---

## Step 3 -- Connect your tools

Run the setup commands your admin sent you. For AWS:

```
vouch setup aws --role &lt;ROLE_ARN&gt;
```

Use the exact command your admin sent; it may include `--region`. SSH needs no setup. Each integration guide lists its one setup command.

---

## Step 4 -- Daily login

Each workday begins with a single `vouch login`. This authenticates you with your YubiKey and provisions short-lived credentials that last for 8 hours.

```
vouch login
Enter PIN: ****
Touch your YubiKey...
Authenticated for 8 hours
```

That is it. After login, every integration -- SSH, AWS, Git -- uses the session credentials automatically. When the 8-hour window expires, run `vouch login` again.

&lt;div class=&#34;checkpoint&#34;&gt;
&lt;p&gt;&lt;strong&gt;You are done with login when...&lt;/strong&gt;&lt;/p&gt;
&lt;ul&gt;
  &lt;li&gt;The command reports that you are authenticated.&lt;/li&gt;
  &lt;li&gt;&lt;code&gt;vouch status&lt;/code&gt; shows an active session and the remaining session time.&lt;/li&gt;
  &lt;li&gt;You do not need to touch the YubiKey again until the session expires or policy requires it.&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;

---

## Step 5 -- Use your first credential

Run a command for a tool you connected in Step 3:

```
# SSH authenticates with your Vouch certificate
ssh user@server

# AWS credentials are available through the configured profile
aws sts get-caller-identity --profile vouch

# Git prompts Vouch&#39;s credential helper when needed
git ls-remote https://github.com/example/private-repo.git
```

&lt;div class=&#34;checkpoint&#34;&gt;
&lt;p&gt;&lt;strong&gt;You are done with first use when...&lt;/strong&gt;&lt;/p&gt;
&lt;ul&gt;
  &lt;li&gt;At least one tool successfully uses credentials from the active Vouch session.&lt;/li&gt;
  &lt;li&gt;AWS commands show your assumed role, SSH accepts your Vouch certificate, or Git uses the Vouch credential helper.&lt;/li&gt;
  &lt;li&gt;No long-lived AWS keys, SSH private keys, GitHub PATs, or registry passwords were created for the test.&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;

---

## Next

Set up the integrations your team uses: [AWS](/docs/aws/), [SSH](/docs/ssh/), or [GitHub](/docs/github/).
