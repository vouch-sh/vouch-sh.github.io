---
title: "Access GitHub Repos without Personal Access Tokens"
linkTitle: "GitHub"
description: "Replace GitHub PATs with short-lived tokens generated from your hardware-backed Vouch session."
weight: 2
subtitle: "Access private GitHub repositories using Vouch authentication"
params:
  docsGroup: integrations
---

Vouch replaces GitHub PATs and deploy keys with short-lived tokens (valid for up to 1 hour) issued through a [GitHub App](https://docs.github.com/en/apps) installed in your organization. Tokens are automatically scoped to the right repositories and tied to a hardware-verified identity.

{{< tldr >}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → this page.
- **Admin, once:** [install the Vouch GitHub App](#step-1----install-the-vouch-github-app-admin) in your GitHub organization.
- **Each developer:** `vouch setup github --configure`, then `git clone https://github.com/your-org/private-repo.git` just works.
{{< /tldr >}}

## Step 1 -- Install the Vouch GitHub App (admin)

{{< role admin >}}

An organization administrator must connect at least one GitHub organization to the Vouch server before any team member can use the integration.

1. Navigate to https://{{< instance-url >}}/github/connect and follow the prompts to install the GitHub App in your GitHub organization. You can choose to grant access to all repositories or select specific ones.

![Connect GitHub page showing installation flow and connected accounts](/images/admin/github-connect.png)

2. After installing the app, confirm the organization connection on the GitHub connect page. The server will verify it can issue tokens for the connected organization.

3. Optionally, adjust which repositories the GitHub App has access to at any time through your GitHub organization settings under **Settings > GitHub Apps > Vouch**.

---

## Step 2 -- Configure Git Credential Helper

{{< role developer >}}

You need a **verified identity** linked to your Vouch account (via your organization's SSO provider).

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

## Step 3 -- Use Git normally

{{< role developer >}}

{{< session-note >}}

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

## How it works

1. **Git requests credentials** -- Git calls the Vouch credential helper when it needs to authenticate to `github.com`.
2. **Vouch exchanges your session** -- The credential helper contacts the Vouch server and exchanges your active hardware-backed session for a GitHub installation access token.
3. **GitHub App issues a token** -- The Vouch server uses the GitHub App to generate an access token valid for **1 hour**, scoped to the repositories your organization has granted access to, and never written to disk. If more than one GitHub organization is connected, Vouch selects the correct token based on the repository you are accessing.
4. **Git authenticates** -- The token is returned to Git and used for the current operation.

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

Your organization administrator has not yet installed the Vouch GitHub App for this organization. Ask them to connect the organization at https://{{< instance-url >}}/github/connect.

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

---

## Related guides

- [Getting Started](/docs/getting-started/) -- Install the CLI and enroll your YubiKey.
- [AWS Integration](/docs/aws/) -- Federate into AWS with OIDC for temporary STS credentials.
- [Docker Registries](/docs/docker/) -- Authenticate to container registries like ECR and GHCR.
- [AWS CodeCommit](/docs/codecommit/) -- Authenticate to AWS CodeCommit Git repositories.
