---
title: "AWS CodeCommit"
description: "Authenticate to AWS CodeCommit repositories using Vouch for seamless Git operations without static credentials."
weight: 8
subtitle: "Authenticate to AWS CodeCommit repositories using Vouch"
---

Vouch integrates with AWS CodeCommit to provide seamless Git authentication backed by hardware security keys. After a single `vouch login`, you can clone, pull, and push to CodeCommit repositories without managing Git credentials or IAM access keys.

AWS offers three ways to authenticate to [CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/welcome.html): SSH keys, HTTPS Git credentials (static username/password from IAM), and IAM access keys with a credential helper. All three involve long-lived secrets. Vouch provides a fourth option: short-lived STS credentials that require no static secrets at all.

## How it works

1. **Git requests credentials** -- Git calls the Vouch credential helper when it needs to authenticate to a CodeCommit repository.
2. **OIDC to STS** -- Vouch exchanges your active hardware-backed session for temporary AWS STS credentials via `AssumeRoleWithWebIdentity`.
3. **STS to CodeCommit** -- Vouch uses the STS credentials to generate short-lived HTTPS Git credentials for CodeCommit.
4. **Git authenticates** -- The credentials are returned to Git and used for the current operation.

Key characteristics:

- **Short-lived credentials** -- Git credentials are derived from your Vouch session and expire when the session ends.
- **No stored secrets** -- There are no IAM access keys, HTTPS Git credentials, or SSH keys to manage.
- **HTTPS only** -- Vouch uses the HTTPS Git credential helper protocol.

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
vouch setup codecommit
```

This configures Git to use Vouch as the credential helper for CodeCommit HTTPS URLs. It adds the following to your `~/.gitconfig`:

```ini
[credential "https://git-codecommit.*.amazonaws.com"]
    helper = vouch
```

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

### Another credential helper is interfering

1. List all configured credential helpers:
   ```
   git config --show-origin --get-all credential.https://git-codecommit.*.amazonaws.com.helper
   ```
2. Remove or reorder conflicting entries so that `vouch` appears first.
3. Re-run `vouch setup codecommit` to ensure the configuration is correct.

---

## Recommended: git-remote-codecommit

For the most reliable experience, consider using [`git-remote-codecommit`](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html) instead of the HTTPS credential helper. It uses the `codecommit://` URL protocol and reads credentials directly from the AWS credential chain, bypassing Git's credential helper system entirely.

This avoids known conflicts with macOS Keychain and Git Credential Manager that can cause "unable to access" errors when other credential helpers are installed.

### Setup

```bash
# Install git-remote-codecommit
pip install git-remote-codecommit

# Clone using your Vouch profile
git clone codecommit://vouch@my-repo

# For a specific region
git clone codecommit::us-west-2://vouch@my-repo
```

The profile name before `@` must match your Vouch AWS profile (typically `vouch`). All subsequent Git operations (`push`, `pull`, `fetch`) work normally.

### Authentication method comparison

| Method | Credential Type | Conflicts | Works with Vouch |
|---|---|---|---|
| SSH keys | Static key pair | No | Not through Vouch |
| HTTPS Git credentials | Static username/password | macOS Keychain, GCM | Yes (via credential helper) |
| `git-remote-codecommit` | STS (from AWS SDK) | None | Yes (recommended) |
