---
title: "Migration Guide"
linkTitle: "Migration"
description: "Migrate from static credentials to Vouch — phased rollout, integration-by-integration checklist, and rollback plan."
weight: 21
subtitle: "Move from static secrets to hardware-backed credentials without disrupting your team"
params:
  docsGroup: manage
---

Migrating to Vouch does not have to be all-or-nothing. You can install Vouch alongside your existing credentials and migrate one integration at a time. This guide walks through a phased rollout, a per-integration checklist, and a rollback plan.

---

## Phase 1 -- Install and enroll

Install the Vouch CLI alongside your existing credential setup. Nothing changes yet.

1. **Install the CLI** on each developer's machine. See [Getting Started](/docs/getting-started/).
2. **Enroll YubiKeys** with the Vouch server. Each developer runs `vouch enroll`.
3. **Test login.** Each developer runs `vouch login` and verifies they can authenticate.

At this point, Vouch is installed but no integrations are active. Existing SSH keys, AWS access keys, and GitHub PATs continue to work as before.

---

## Phase 2 -- Migrate integrations one at a time

Pick one integration to migrate first. AWS is recommended because `credential_process` works alongside existing credentials without conflict.

### Migration order (recommended)

| Order | Integration | Why first/last |
|---|---|---|
| 1 | **AWS** | `credential_process` adds a new profile. Existing profiles and access keys are unaffected. |
| 2 | **SSH** | Vouch SSH certificates work alongside existing SSH keys. The SSH agent falls back to keys if the certificate is unavailable. |
| 3 | **GitHub** | Git credential helpers can be stacked. Vouch adds a new helper without removing existing ones. |
| 4 | **Docker** | Docker credential helpers can be configured per registry. Migrate one registry at a time. |
| 5 | **CodeCommit** | Requires AWS integration to be working first. |
| 6 | **CodeArtifact** | Requires AWS integration to be working first. |
| 7 | **EKS** | Requires AWS integration and `kubectl` configuration. |
| 8 | **Databases** | Requires AWS integration and application-level changes for IAM auth. |

### Per-integration checklist

For each integration:

- [ ] **Configure the integration** using `vouch setup <integration>`.
- [ ] **Test with one developer** before rolling out to the team.
- [ ] **Verify the tool works** with Vouch credentials (e.g., `aws s3 ls --profile vouch`, `ssh user@server`, `git push`).
- [ ] **Run for one week** with both Vouch and static credentials available.
- [ ] **Migrate the rest of the team** once the pilot developer confirms it works.

---

## Phase 3 -- Revoke old credentials

After each integration is working with Vouch for the entire team, revoke the static credentials it replaced:

### AWS access keys

```bash
# List existing access keys
aws iam list-access-keys --user-name alice

# Deactivate first (in case you need to re-enable)
aws iam update-access-key --user-name alice --access-key-id AKIAXXXXXXXX --status Inactive

# Delete after confirming everything works
aws iam delete-access-key --user-name alice --access-key-id AKIAXXXXXXXX
```

Check `~/.aws/credentials` on each developer's machine and remove static entries.

### SSH keys

1. Remove old public keys from `~/.ssh/authorized_keys` on servers (or from your configuration management tool).
2. Keep the Vouch CA public key as the only trusted signer in `sshd_config`.
3. Developers can keep their SSH key files locally as a fallback, or remove them.

### GitHub PATs

1. Navigate to GitHub **Settings > Developer settings > Personal access tokens**.
2. Revoke tokens that were used for repository access.
3. Verify that `git push` still works with the Vouch credential helper.

### Docker credentials

1. Check `~/.docker/config.json` for stored registry credentials.
2. Remove entries for registries now handled by Vouch.

---

## CI/CD considerations

CI/CD pipelines typically do not have YubiKeys. They will continue to use their existing credential mechanisms:

| CI/CD pattern | Recommendation |
|---|---|
| **GitHub Actions** | Use OIDC federation with GitHub's built-in OIDC provider (`token.actions.githubusercontent.com`). This is separate from Vouch and provides the same STS-based authentication for pipelines. |
| **AWS CodeBuild / CodePipeline** | Use IAM roles attached to the build environment. No static keys needed. |
| **Jenkins / GitLab CI** | Use IAM roles if running on EC2, or [Vouch CI/CD integration](/docs/cicd/) for human approval gates. |
| **Static keys in CI/CD** | If your pipeline currently uses static AWS keys, keep them for now. Replace them with OIDC federation (GitHub Actions) or IAM roles (EC2-based runners) as a separate project. |

Vouch's [CI/CD integration](/docs/cicd/) is designed for human approval gates (e.g., requiring a YubiKey tap before a production deployment), not for replacing machine credentials in automated pipelines.

---

## Rollback plan

If you need to revert to static credentials for any integration:

### AWS

1. Re-enable or re-create IAM access keys for affected users.
2. Update `~/.aws/credentials` with the static keys.
3. Remove or comment out the `credential_process` line in `~/.aws/config` for the Vouch profile.

### SSH

1. Re-add public keys to `~/.ssh/authorized_keys` on servers.
2. Ensure `IdentityFile` entries in `~/.ssh/config` point to the static keys.
3. SSH falls back to keys automatically if the Vouch certificate is unavailable.

### GitHub

1. Generate a new GitHub PAT.
2. Update the Git credential helper or set `GIT_ASKPASS` to use the PAT.
3. Remove the Vouch credential helper entry from `~/.gitconfig`.

### Docker

1. Run `docker login <registry>` with static credentials.
2. Remove the Vouch credential helper from `~/.docker/config.json`.

---

## Verification checklist

After completing migration for all integrations:

- [ ] All developers can `vouch login` and access all required services.
- [ ] No static AWS access keys remain active in IAM.
- [ ] No static SSH keys remain in `authorized_keys` (only the Vouch CA is trusted).
- [ ] No GitHub PATs remain active.
- [ ] CI/CD pipelines continue to function with their own credential mechanisms.
- [ ] SCIM provisioning is configured for automated onboarding/offboarding (recommended for teams > 15 people).
- [ ] Break-glass procedures are documented for emergency access without Vouch. See [Availability](/docs/availability/).
