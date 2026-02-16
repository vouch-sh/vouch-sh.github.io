---
title: "GitHub Integration"
description: "Access private GitHub repositories using short-lived tokens via Vouch authentication."
weight: 5
subtitle: "Access private GitHub repositories using Vouch authentication"
---

Vouch replaces static GitHub personal access tokens and deploy keys with short-lived, hardware-backed credentials. After running `vouch login`, you can clone, pull, and push to private repositories without managing any GitHub tokens yourself.

GitHub personal access tokens are long-lived, broadly scoped, and stored in plaintext in shell configs and CI environments. Deploy keys are limited to a single repository per key and require manual management on every machine. Vouch replaces both with 15-minute tokens issued through a [GitHub App](https://docs.github.com/en/apps) installed in your organization -- automatically scoped to the right repositories and tied to a hardware-verified identity.

## How it works

When you perform a Git operation against a GitHub repository, the Vouch credential helper intercepts the authentication request and obtains a short-lived GitHub access token on your behalf:

1. **Git requests credentials** -- Git calls the Vouch credential helper when it needs to authenticate to `github.com`.
2. **Vouch exchanges your session** -- The credential helper contacts the Vouch server and exchanges your active hardware-backed session for a GitHub installation access token.
3. **GitHub App issues a token** -- The Vouch server uses a GitHub App installed in your organization to generate a short-lived access token scoped to the repositories your organization has granted access to.
4. **Git authenticates** -- The token is returned to Git and used for the current operation.

Key characteristics:

- **Short-lived tokens** -- Access tokens are valid for **15 minutes** and are never written to disk.
- **Multiple GitHub organizations** -- If your Vouch server is connected to more than one GitHub organization, Vouch automatically selects the correct token based on the repository you are accessing.
- **No stored secrets** -- There are no personal access tokens, SSH keys, or deploy keys to rotate or revoke.

---

## Prerequisites

Before configuring the GitHub integration, make sure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- A **verified identity** linked to your Vouch account (via your organization's SSO provider)
- Your **organization administrator** has connected at least one GitHub organization to the Vouch server

---

## Step 1 -- Configure Git Credential Helper

Run the setup command to install the Vouch credential helper for GitHub:

```
vouch setup github
```

This prints the Git configuration that will be added. To apply it automatically:

```
vouch setup github --configure
```

The command adds the following to your `~/.gitconfig`:

```ini
[credential "https://github.com"]
    helper = vouch
```

This tells Git to use the Vouch credential helper whenever it needs credentials for `github.com` over HTTPS.

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
# Clone a private repository
git clone https://github.com/your-org/private-repo.git

# Pull latest changes
cd private-repo
git pull

# Push commits
git add .
git commit -m "Update feature"
git push
```

Vouch handles authentication transparently. You do not need to enter a username, password, or token.

---

## Admin Setup

Organization administrators must complete the following steps before team members can use the GitHub integration:

1. **Install the Vouch GitHub App** -- Navigate to the Vouch admin panel at {{< instance-url >}}/admin and follow the prompts to install the GitHub App in your organization. You can choose to grant access to all repositories or select specific ones.

2. **Authorize the GitHub organization** -- After installing the app, confirm the organization connection in the Vouch admin panel. The server will verify it can issue tokens for the connected organization.

3. **Manage repository access** -- You can adjust which repositories the GitHub App has access to at any time through your GitHub organization settings under **Settings > GitHub Apps > Vouch**.

4. **Communicate to your team** -- Let team members know they can run `vouch setup github --configure` and begin using the integration immediately.

---

## Troubleshooting

### Authentication failed

```
fatal: Authentication failed for 'https://github.com/org/repo.git'
```

- Verify you have an active Vouch session by running `vouch login`.
- Confirm the credential helper is configured: `git config --global credential.https://github.com.helper` should return `vouch`.
- Check that the Vouch agent is running: `vouch status`.

### Organization not connected

```
error: GitHub organization "org-name" is not connected to this Vouch server
```

Your organization administrator has not yet installed the Vouch GitHub App for this organization. Ask them to connect the organization through the admin panel at {{< instance-url >}}/admin.

### Requires membership

```
error: you are not a member of the GitHub organization "org-name"
```

The GitHub App is installed, but your GitHub account is not a member of the organization. Verify that your GitHub username is associated with the correct organization and that your Vouch identity is linked to the right email address.

### Repository not accessible

```
error: repository not accessible with current token scope
```

The Vouch GitHub App does not have access to the specific repository you are trying to reach. Ask your organization administrator to update the app's repository permissions in GitHub organization settings.

### Multiple GitHub accounts

If you have multiple GitHub accounts and the wrong one is being used:

1. Check which credential helpers are configured: `git config --global --get-all credential.https://github.com.helper`
2. Ensure Vouch is listed and no other helpers are overriding it.
3. If you use different GitHub accounts for different organizations, Vouch handles this automatically by issuing tokens scoped to the correct organization based on the repository URL.

### Wrong credential helper being used

If another credential helper (such as `osxkeychain` or `manager`) is taking priority over Vouch:

1. List all configured helpers:
   ```
   git config --show-origin --get-all credential.https://github.com.helper
   ```
2. Remove or reorder conflicting entries so that `vouch` appears first.
3. Re-run `vouch setup github --configure` to ensure the configuration is correct.
