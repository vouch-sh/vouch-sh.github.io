---
title: "Security Model"
linkTitle: "Security"
description: "How Vouch protects credentials at every layer — data flow, threat model, credential lifecycle, and supply chain integrity."
weight: 18
subtitle: "Understand how Vouch secures your credentials from hardware key to cloud service"
params:
  docsGroup: manage
---

Vouch brokers the most sensitive credentials in a developer's stack: SSH certificates, AWS STS tokens, GitHub installation tokens, and container registry passwords. This page explains exactly how those credentials are protected at every layer.

---

## Data flow

Every Vouch credential follows the same path from hardware key to cloud service:

```
YubiKey (FIDO2)
  → Vouch CLI (local machine)
    → Vouch Server (validates assertion, issues session)
      → External service (AWS STS / GitHub / SSH CA)
        → Short-lived credential returned to CLI
          → Tool uses credential (aws, ssh, git, docker)
```

1. **FIDO2 assertion** -- The YubiKey signs a challenge using a private key that never leaves the hardware. The assertion includes origin binding (preventing phishing) and user verification (PIN + touch).
2. **Session issuance** -- The Vouch server validates the signed assertion against the enrolled public key and issues a session token valid for 8 hours.
3. **Credential exchange** -- When a tool needs a credential, the CLI exchanges the session token for a service-specific credential (STS `AssumeRoleWithWebIdentity`, SSH certificate signing, GitHub App installation token, etc.).
4. **Tool consumption** -- The tool receives the short-lived credential and uses it normally. The credential expires on its own -- there is nothing to revoke or rotate.

---

## Credential lifecycle

All Vouch credentials share these properties:

| Property | Detail |
|---|---|
| **Storage** | In-memory only, held by the Vouch agent process. Never written to disk. |
| **Lifetime** | Session: 8 hours. AWS STS: up to 1 hour. SSH certificate: 8 hours. GitHub token: 1 hour. |
| **Scope** | Tied to a single authenticated user. Cannot be shared or transferred. |
| **Revocation** | Sessions can be revoked server-side (e.g., via SCIM de-provisioning). Outstanding short-lived credentials expire naturally. |
| **Rotation** | Not applicable -- credentials are issued fresh on each request and expire automatically. |

Because credentials are never written to disk, they cannot be exfiltrated by malware scanning `~/.aws/credentials`, `~/.ssh/`, or environment variables.

---

## Trust boundaries

Vouch operates across three trust boundaries:

### 1. Hardware key (YubiKey)

- Private key material is generated on the YubiKey and **never exported**.
- FIDO2 assertions are origin-bound -- the key will not sign challenges from phishing domains.
- User verification requires both a PIN and a physical touch.

### 2. Local machine (CLI + Agent)

- The Vouch agent runs as a user-level process and holds session material in memory.
- Communication between the CLI and agent uses a Unix domain socket with filesystem permissions restricting access to the owning user.
- No credentials are persisted to disk. If the agent process stops, sessions must be re-established with a new `vouch login`.

### 3. Vouch server

- The server validates FIDO2 assertions and issues signed OIDC tokens (ES256 over P-256).
- The server does not store AWS credentials, SSH private keys, or GitHub tokens. It acts as an identity broker, not a secrets vault.
- Communication between CLI and server uses TLS 1.2+.

---

## Threat model

### What Vouch mitigates

| Threat | Mitigation |
|---|---|
| **Credential theft** | No long-lived secrets on disk. Credentials are in-memory and short-lived. |
| **Phishing** | FIDO2 origin binding prevents credentials from being used on attacker-controlled domains. |
| **Lateral movement** | Credentials are scoped to a single user and expire within hours. A compromised credential limits blast radius. |
| **Insider threats** | Every credential issuance is tied to a hardware-verified identity. CloudTrail and server logs provide full attribution. |
| **Offboarding gaps** | SCIM de-provisioning revokes sessions immediately. Outstanding credentials expire within hours. |
| **Credential sharing** | Credentials are bound to a FIDO2 assertion that requires physical possession of the enrolled key. |

### What Vouch does not mitigate

| Threat | Explanation |
|---|---|
| **Compromised endpoint with active session** | If an attacker gains access to a machine with an active Vouch session, they can use the in-memory credentials until the session expires (up to 8 hours). This is the same exposure window as any session-based system. |
| **Server compromise** | If the Vouch server is compromised, an attacker could issue sessions for any enrolled user. The server does not hold AWS keys or SSH private keys, but it can broker new credentials. |
| **Physical key theft with known PIN** | If an attacker obtains both the YubiKey and the PIN, they can authenticate as the enrolled user. Use a strong PIN and report lost keys immediately. |
| **Supply chain attacks on external services** | Vouch relies on AWS STS, GitHub APIs, and other external services. Vulnerabilities in those services are outside Vouch's control. |

---

## Encryption

### In transit

All communication between the Vouch CLI and server uses **TLS 1.2+**. The FIDO2 assertion is transmitted over this encrypted channel.

### At rest

Vouch does not store credentials at rest. The server stores:

- **Enrolled public keys** -- The FIDO2 public key registered during enrollment. This is not sensitive (it cannot be used to impersonate the user).
- **User metadata** -- Email address, organization membership, and enrollment status.
- **Audit logs** -- Records of authentication events and credential issuance.

No private keys, AWS credentials, SSH keys, or tokens are stored on the server.

---

## FIDO2 security properties

Vouch uses [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) for all user authentication. Key security properties:

- **Origin binding** -- The authenticator (YubiKey) includes the relying party ID in the signed assertion. If an attacker stands up a phishing site at a different domain, the assertion will not validate against the Vouch server.
- **Hardware key storage** -- The private key is generated on the YubiKey's secure element and cannot be extracted, cloned, or backed up.
- **User verification** -- Every assertion requires the user's PIN and a physical touch of the key, providing two-factor authentication in a single gesture.
- **Replay protection** -- Each assertion includes a signature counter that the server tracks. Replayed assertions are rejected.

---

## Supply chain security

### SLSA provenance

Vouch release binaries are built with [SLSA Level 3](https://slsa.dev/) provenance. Each release includes a provenance attestation that you can verify:

```bash
slsa-verifier verify-artifact vouch-linux-amd64 \
  --provenance-path vouch-linux-amd64.intoto.jsonl \
  --source-uri github.com/vouch-sh/vouch
```

This confirms the binary was built from the expected source repository using a tamper-resistant build process.

### SHA256 checksums

Every release includes a `checksums.txt` file. Verify downloaded binaries:

```bash
sha256sum --check checksums.txt
```

### Package manager verification

When installed via Homebrew, APT, or DNF, package signatures are verified automatically by the package manager using Vouch's published GPG key.

---

## Shared responsibility

### Vouch's responsibilities

- Secure the server infrastructure and FIDO2 registration data.
- Issue credentials with minimum necessary lifetime and scope.
- Provide SLSA-attested builds and signed packages.
- Revoke sessions when triggered by SCIM de-provisioning.
- Maintain audit logs of all authentication and credential issuance events.

### Your responsibilities

- Protect YubiKeys and PINs. Report lost or stolen keys immediately.
- Configure IAM roles with least-privilege permissions.
- Set up SCIM to automate user lifecycle management.
- Monitor CloudTrail and server audit logs for anomalous activity.
- Keep the Vouch CLI updated to receive security patches.

---

## AWS Well-Architected alignment

Vouch addresses several controls from the [AWS Well-Architected Framework Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html):

| Control | How Vouch addresses it |
|---|---|
| **SEC02 -- Manage identities** | Every credential traces to a hardware-verified human identity. No shared or service accounts for human access. |
| **SEC03 -- Manage permissions** | Short-lived STS credentials with scoped IAM roles. Session tags enable attribute-based access control (ABAC). |
| **SEC04 -- Detect and investigate** | CloudTrail logs include the authenticated user's email for every API call. Server audit logs capture all authentication events. |
| **SEC09 -- Protect data in transit** | All Vouch communication uses TLS 1.2+. FIDO2 assertions are cryptographically bound to the relying party origin. |

---

## Incident response

If you suspect a security issue with the Vouch service or have discovered a vulnerability:

- Email **security@vouch.sh** with details.
- Include reproduction steps if possible.
- Do not disclose the issue publicly until it has been addressed.

If a YubiKey is lost or stolen, remove it from the user's account immediately to prevent unauthorized authentication.
