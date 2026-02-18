---
title: "Access AWS CodeCommit without Git Credentials"
linkTitle: "AWS CodeCommit"
description: "Clone and push to CodeCommit repositories using short-lived credentials instead of HTTPS Git credentials or SSH keys."
weight: 8
subtitle: "Authenticate to AWS CodeCommit repositories using Vouch"
params:
  docsGroup: code
---

AWS offers three ways to authenticate to [CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/welcome.html): SSH keys, HTTPS Git credentials (static username/password from IAM), and IAM access keys with a credential helper. All three involve long-lived secrets that need to be distributed and rotated.

Vouch provides a fourth option: short-lived STS credentials that require no static secrets at all. After a single `vouch login`, you can clone, pull, and push to CodeCommit repositories without managing Git credentials or IAM access keys. Vouch supports both HTTPS credential helper and native `codecommit://` remote helper authentication.

## How it works

1. **Git requests credentials** -- Git calls the Vouch credential helper when it needs to authenticate to a CodeCommit repository.
2. **OIDC to STS** -- Vouch exchanges your active hardware-backed session for temporary AWS STS credentials via `AssumeRoleWithWebIdentity`.
3. **SigV4 signing** -- Vouch uses the STS credentials to sign the Git HTTP request with AWS Signature Version 4, authenticating directly to CodeCommit without generating intermediate HTTPS Git credentials.
4. **Git authenticates** -- The signed credentials are returned to Git and used for the current operation.

Key characteristics:

- **Short-lived credentials** -- Git credentials are derived from your Vouch session and expire when the session ends.
- **No stored secrets** -- There are no IAM access keys, HTTPS Git credentials, or SSH keys to manage.
- **Native SigV4** -- Vouch signs requests directly using the AWS SigV4 protocol, bypassing the legacy HTTPS Git credential system.
- **Two authentication methods** -- Use HTTPS URLs with the credential helper, or `codecommit://` URLs with the native remote helper.

---

## Prerequisites

Before configuring the CodeCommit integration, make sure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- The **[AWS integration](/docs/aws/)** configured (OIDC provider and IAM role)
- The IAM role must have `codecommit:GitPull` and `codecommit:GitPush` permissions on the target repositories

---

## Step 1 -- Configure the Git credential helper

Run the setup command to install the Vouch credential helper for CodeCommit:

```
vouch setup codecommit [--region <REGION>] [--profile <PROFILE>] [--configure]
```

| Flag | Description |
|---|---|
| `--region` | AWS region (default: wildcard matching all regions) |
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--configure` | Apply the configuration automatically (without this flag, the command only prints the configuration) |

This configures Git to use Vouch as the credential helper for CodeCommit HTTPS URLs across all supported AWS partitions. It adds the following to your `~/.gitconfig`:

```ini
[credential "https://git-codecommit.*.amazonaws.com"]
    helper = vouch
    useHttpPath = true

[credential "https://git-codecommit.*.amazonaws.com.cn"]
    helper = vouch
    useHttpPath = true

[credential "https://git-codecommit.*.amazonaws.eu"]
    helper = vouch
    useHttpPath = true
```

The setup also installs the native `git-remote-codecommit` helper as a symlink at `~/.local/bin/git-remote-codecommit`, enabling `codecommit://` URL support (see below).

---

## Step 2 -- Authenticate

If you have not already logged in today, authenticate with your YubiKey:

```
vouch login
```

Your session lasts for 8 hours. All Git operations during that window use the session automatically.

---

## Step 3 -- Use Git normally

With the credential helper configured and an active session, Git commands work without any extra flags or tokens:

```bash
# Clone a CodeCommit repository
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-repo

# Pull latest changes
cd my-repo
git pull

# Push commits
git add .
git commit -m "Update feature"
git push
```

Vouch handles authentication transparently. You do not need to enter a username, password, or configure IAM HTTPS Git credentials.

---

## Native `codecommit://` remote helper

Vouch ships its own native `git-remote-codecommit` remote helper, which is installed automatically by `vouch setup codecommit`. This provides `codecommit://` URL support without requiring any external dependencies -- no Python or `pip install` needed.

The remote helper reads credentials directly from the AWS credential chain, bypassing Git's credential helper system entirely. This avoids known conflicts with macOS Keychain and Git Credential Manager.

### URL formats

| Format | Description |
|---|---|
| `codecommit://my-repo` | Uses the default AWS profile and region |
| `codecommit://vouch@my-repo` | Uses the `vouch` AWS profile |
| `codecommit::us-west-2://my-repo` | Uses a specific region |
| `codecommit::us-west-2://vouch@my-repo` | Uses a specific region and profile |

### Usage

```bash
# Clone using the default profile
git clone codecommit://my-repo

# Clone using a specific AWS profile
git clone codecommit://vouch@my-repo

# Clone from a specific region
git clone codecommit::us-west-2://vouch@my-repo
```

The profile name before `@` must match your AWS profile (typically `vouch`). All subsequent Git operations (`push`, `pull`, `fetch`) work normally.

### When to use `codecommit://` URLs

Use `codecommit://` URLs when:

- You have multiple credential helpers installed and experience conflicts (macOS Keychain, Git Credential Manager)
- You want to avoid HTTPS URL region-specific hostnames
- You need to work with multiple AWS profiles or regions across repositories

---

## Cross-partition support

Vouch supports CodeCommit across all AWS partitions:

| Partition | URL Suffix | Region Examples |
|---|---|---|
| **Standard** (`aws`) | `.amazonaws.com` | `us-east-1`, `eu-west-1`, `ap-southeast-1` |
| **China** (`aws-cn`) | `.amazonaws.com.cn` | `cn-north-1`, `cn-northwest-1` |
| **EU Sovereign Cloud** | `.amazonaws.eu` | EU sovereign regions |

The credential helper is configured for all three partitions automatically during setup.

---

## Troubleshooting

### Authentication failed

```
fatal: Authentication failed for 'https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-repo'
```

- Verify you have an active Vouch session: `vouch login`.
- Confirm the credential helper is configured: `git config --global credential.https://git-codecommit.*.amazonaws.com.helper` should return `vouch`.
- Check that the Vouch agent is running: `vouch status`.

### "Not authorized to perform codecommit:GitPull"

- Verify the IAM role you are assuming has the correct CodeCommit permissions.
- Check that the IAM trust policy allows `AssumeRoleWithWebIdentity` from the Vouch OIDC provider.

### Wrong region

- CodeCommit repository URLs include the region (e.g., `git-codecommit.us-east-1.amazonaws.com`). Make sure you are using the correct region for your repository.
- Alternatively, use `codecommit://` URLs with an explicit region: `codecommit::us-east-1://vouch@my-repo`.

### Another credential helper is interfering

1. List all configured credential helpers:
   ```
   git config --show-origin --get-all credential.https://git-codecommit.*.amazonaws.com.helper
   ```
2. Remove or reorder conflicting entries so that `vouch` appears first.
3. Re-run `vouch setup codecommit` to ensure the configuration is correct.
4. Alternatively, switch to `codecommit://` URLs which bypass Git's credential helper system entirely.

---

## Authentication method comparison

| Method | Credential Type | Conflicts | Works with Vouch |
|---|---|---|---|
| SSH keys | Static key pair | No | Not through Vouch |
| HTTPS Git credentials | Static username/password | macOS Keychain, GCM | Not through Vouch |
| HTTPS credential helper (Vouch) | SigV4-signed (from STS) | macOS Keychain, GCM | Yes |
| `codecommit://` remote helper (Vouch) | SigV4-signed (from STS) | None | Yes (recommended) |
