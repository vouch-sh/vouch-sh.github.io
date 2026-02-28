---
title: "Availability and Failure Modes"
linkTitle: "Availability"
description: "What happens when the Vouch server is unreachable — offline behavior, credential expiry, and blast radius."
weight: 21
subtitle: "Understand the blast radius when things go wrong"
params:
  docsGroup: manage
---

Vouch sits in the critical path for developer credentials. Before adopting it, you should understand what happens when the server is unreachable, when a session expires, or when individual integrations fail.

---

## Normal operation

During normal operation with an active session (up to 8 hours after `vouch login`):

| Integration | Credential source | Lifetime |
|---|---|---|
| SSH | Certificate cached in agent memory | 8 hours |
| AWS (`credential_process`) | STS credentials fetched on demand, cached | 1 hour |
| GitHub | Installation token fetched on demand | 1 hour |
| Docker (ECR) | Registry token fetched on demand | 12 hours |
| Docker (GHCR) | GitHub token fetched on demand | 1 hour |
| CodeCommit | SigV4 signature computed on demand | Per-request |
| CodeArtifact | Authorization token fetched on demand | 12 hours |
| Cargo | Token fetched on demand | Session lifetime |

---

## Offline behavior

### Server unreachable during active session

If the Vouch server becomes unreachable while you have an active session:

| What works | Why |
|---|---|
| **SSH connections** | The SSH certificate is cached in the agent's memory. It remains valid until it expires (up to 8 hours from login). No server contact is needed. |
| **AWS commands** | Cached STS credentials remain valid until their 1-hour expiry. If cached credentials exist, `credential_process` returns them without contacting the server. |
| **Docker pulls** (with cached token) | ECR tokens last up to 12 hours and are cached locally by Docker. |

| What fails | Why |
|---|---|
| **New AWS credential requests** (after cache expires) | `credential_process` calls `vouch credential aws`, which needs to exchange the session for fresh STS credentials via the server. |
| **New GitHub token requests** | GitHub installation tokens are fetched through the server. |
| **New CodeArtifact tokens** | Token exchange requires server communication. |
| **`vouch login`** | A new login always requires the server for FIDO2 validation. |

**Key point:** An active session with cached credentials continues working without server contact. The impact of a server outage depends on when cached credentials expire. In the worst case, you have up to 1 hour of AWS access and up to 8 hours of SSH access after the server goes down.

### Server unreachable with no active session

If the Vouch server is unreachable and you have no active session (e.g., at the start of a workday), you cannot authenticate. No new credentials can be obtained.

**Mitigation:** Break-glass access. Maintain a separate emergency access path (e.g., an IAM user with MFA in a sealed envelope, or AWS root account credentials in a hardware security module) for situations where Vouch is unavailable and critical access is needed.

---

## Session expiry

When your 8-hour session expires:

- **SSH connections** in progress continue until they are closed (the certificate was already presented during connection setup).
- **New SSH connections** fail because the certificate has expired.
- **AWS commands** continue working until the cached STS credentials expire (up to 1 hour after the last credential fetch).
- **New credential requests** of all types fail until you run `vouch login` again.

This is by design. Short session lifetimes limit the window of exposure if a machine is compromised.

---

## Credential helper failure modes

### `credential_process` (AWS)

The AWS CLI and SDKs call `vouch credential aws` via the `credential_process` configuration. If this command fails:

- The AWS CLI prints an error: `Error when retrieving credentials from custom-process`.
- The command does not fall back to other credential sources in the same profile. However, if you have other AWS profiles configured (e.g., a fallback profile with static keys), you can switch profiles.
- **Important:** `credential_process` errors do not cache. Each AWS command retries the credential process independently.

### SSH agent

If the Vouch agent is not running:

- `ssh` falls back to the next available authentication method (other SSH agents, keys in `~/.ssh/`, password authentication) depending on your SSH configuration.
- If no fallback is configured, the connection fails with `Permission denied`.

### Git credential helper

If the Vouch Git credential helper fails (for GitHub or CodeCommit):

- Git prompts for a username and password, or fails with `Authentication failed` depending on the remote configuration.
- This does not affect other Git remotes that do not use Vouch.

---

## De-provisioning timeline

When a user is de-provisioned (via SCIM or manual removal):

| Time | Effect |
|---|---|
| **Immediately** | Active sessions are revoked on the server. New credential requests fail. |
| **Within 1 hour** | Cached AWS STS credentials expire. AWS access stops. |
| **Within 8 hours** | SSH certificate expires. SSH access stops. |
| **Within 12 hours** | Cached ECR/CodeArtifact tokens expire. |

The maximum exposure window after de-provisioning is the longest credential lifetime (currently 12 hours for ECR tokens). For most integrations, access ends within 1 hour.

---

## Status and monitoring

Check the Vouch server's operational status:

- **Status page:** Contact your Vouch server administrator for status page URL.
- **CLI health check:** `vouch status` reports whether the agent is running and whether the current session is valid.

---

## Planning for failure

### Recommendations

1. **Establish break-glass procedures** before rolling out Vouch. Document an emergency access path that does not depend on Vouch.
2. **Stagger session starts** across the team. If everyone logs in at 9 AM, everyone's sessions expire at 5 PM. Consider encouraging re-login before critical deployments.
3. **Monitor `vouch login` failures** as an early warning of server issues.
4. **Keep the Vouch agent running** to preserve cached credentials across CLI invocations. On macOS, use `brew services`. On Linux, use systemd.
5. **Test credential expiry** in a non-production environment so the team knows what failure looks like before it happens in production.
