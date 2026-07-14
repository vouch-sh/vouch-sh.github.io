# Access AWS CodeCommit without Git Credentials

> Clone and push to AWS CodeCommit repositories using short-lived credentials instead of HTTPS Git credentials or SSH keys.

Source: https://vouch.sh/docs/codecommit/
Last updated: 2026-07-04

---
Vouch authenticates to [AWS CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/welcome.html) using short-lived STS credentials -- no SSH keys, no HTTPS Git credentials, and no IAM access keys to manage. Both HTTPS credential helper and native `codecommit://` remote helper are supported.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once:** add `codecommit:GitPull` and `codecommit:GitPush` to the Vouch IAM role.
- **Each developer:** `vouch setup codecommit --configure`, then `git clone https://git-codecommit.&lt;region&gt;.amazonaws.com/v1/repos/&lt;repo&gt;` just works.
{{&lt; /tldr &gt;}}

## Prerequisites

{{&lt; role admin &gt;}}

Before developers can configure the AWS CodeCommit integration:

- The **[AWS integration](/docs/aws/)** must be configured (OIDC provider and IAM role)
- The IAM role must have `codecommit:GitPull` and `codecommit:GitPush` permissions on the target repositories

---

## Step 1 -- Configure the Git credential helper

{{&lt; role developer &gt;}}

Run the setup command to install the Vouch credential helper for AWS CodeCommit:

```
vouch setup codecommit [--region &lt;REGION&gt;] [--profile &lt;PROFILE&gt;] [--configure]
```

| Flag | Description |
|---|---|
| `--region` | AWS region (default: wildcard matching all regions) |
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--configure` | Apply the configuration automatically (without this flag, the command only prints the configuration) |

This configures Git to use Vouch as the credential helper for AWS CodeCommit HTTPS URLs across all supported AWS partitions. It adds the following to your `~/.gitconfig`:

```ini
[credential &#34;https://git-codecommit.*.amazonaws.com&#34;]
    helper = vouch
    useHttpPath = true

[credential &#34;https://git-codecommit.*.amazonaws.com.cn&#34;]
    helper = vouch
    useHttpPath = true

[credential &#34;https://git-codecommit.*.amazonaws.eu&#34;]
    helper = vouch
    useHttpPath = true
```

The setup also installs the native `git-remote-codecommit` helper as a symlink at `~/.local/bin/git-remote-codecommit`, enabling `codecommit://` URL support (see below).

---

## Step 2 -- Use Git normally

{{&lt; role developer &gt;}}

{{&lt; session-note &gt;}}

With the credential helper configured and an active session, Git commands work without any extra flags or tokens:

```bash
# Clone an AWS CodeCommit repository
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-repo

# Pull latest changes
cd my-repo
git pull

# Push commits
git add .
git commit -m &#34;Update feature&#34;
git push
```

Vouch handles authentication transparently. You do not need to enter a username, password, or configure IAM HTTPS Git credentials.

---

## Native `codecommit://` remote helper

Vouch ships its own native `git-remote-codecommit` remote helper, which is installed automatically by `vouch setup codecommit`. This provides `codecommit://` URL support without requiring any external dependencies -- no Python or `pip install` needed.

The remote helper reads credentials directly from the AWS credential chain, bypassing Git&#39;s credential helper system entirely. This avoids known conflicts with macOS Keychain and Git Credential Manager.

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

All AWS partitions -- standard (`aws`), China (`aws-cn`), and European Sovereign Cloud (`aws-eusc`) -- are configured automatically during setup, as the `~/.gitconfig` entries in Step 1 show.

---

## Troubleshooting

### Authentication failed

```
fatal: Authentication failed for &#39;https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-repo&#39;
```

- Verify you have an active Vouch session: `vouch login`.
- Confirm the credential helper is configured: `git config --global credential.https://git-codecommit.*.amazonaws.com.helper` should return `vouch`.
- Check that the Vouch agent is running: `vouch status`.

### &#34;Not authorized to perform codecommit:GitPull&#34;

- Verify the IAM role you are assuming has the correct AWS CodeCommit permissions.
- Check that the IAM trust policy allows `AssumeRoleWithWebIdentity` from the Vouch OIDC provider.

### Wrong region

- AWS CodeCommit repository URLs include the region (e.g., `git-codecommit.us-east-1.amazonaws.com`). Make sure you are using the correct region for your repository.
- Alternatively, use `codecommit://` URLs with an explicit region: `codecommit::us-east-1://vouch@my-repo`.

### Another credential helper is interfering

1. List all configured credential helpers:
   ```
   git config --show-origin --get-all credential.https://git-codecommit.*.amazonaws.com.helper
   ```
2. Remove or reorder conflicting entries so that `vouch` appears first.
3. Re-run `vouch setup codecommit` to ensure the configuration is correct.
4. Alternatively, switch to `codecommit://` URLs which bypass Git&#39;s credential helper system entirely.

---

## How it works

1. **Git requests credentials** -- Git calls the Vouch credential helper when it needs to authenticate to an AWS CodeCommit repository.
2. **OIDC to STS** -- Vouch exchanges your active hardware-backed session for temporary AWS STS credentials via `AssumeRoleWithWebIdentity`.
3. **SigV4 signing** -- Vouch uses the STS credentials to sign the Git HTTP request with AWS Signature Version 4, authenticating directly to AWS CodeCommit -- bypassing the legacy HTTPS Git credential system, with no stored secrets on disk.
4. **Git authenticates** -- The signed credentials are returned to Git and used for the current operation.

---

## Authentication method comparison

| Method | Credential Type | Conflicts | Works with Vouch |
|---|---|---|---|
| SSH keys | Static key pair | No | Not through Vouch |
| HTTPS Git credentials | Static username/password | macOS Keychain, GCM | Not through Vouch |
| HTTPS credential helper (Vouch) | SigV4-signed (from STS) | macOS Keychain, GCM | Yes |
| `codecommit://` remote helper (Vouch) | SigV4-signed (from STS) | None | Yes (recommended) |
