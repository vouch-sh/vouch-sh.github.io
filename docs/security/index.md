# Security Model

> How Vouch protects credentials at every layer — data flow, threat model, credential lifecycle, and supply chain integrity.

Source: https://vouch.sh/docs/security/
Last updated: 2026-04-10

---


Vouch brokers the most sensitive credentials in a developer's stack: SSH certificates, AWS STS tokens, GitHub installation tokens, and container registry passwords. This page explains exactly how those credentials are protected at every layer.

---

## Data flow

Every Vouch credential follows the same path from hardware key to cloud service:

```
YubiKey (FIDO2)
  → Vouch CLI (local machine)
    → Vouch Server (validates assertion, evaluates device posture, issues session)
      → External service (AWS STS / GitHub / SSH CA)
        → Short-lived credential returned to CLI
          → Tool uses credential (aws, ssh, git, docker)
```

1. **FIDO2 assertion** -- The YubiKey signs a challenge using a private key that never leaves the hardware. The assertion includes origin binding (preventing phishing) and user verification (PIN + touch).
2. **Device posture evaluation** -- If active [posture policies](/docs/device-posture/) exist, the server evaluates the device's security state (disk encryption, firewall, EDR, screen lock, etc.) against configured policies. If any policy fails, access is denied with OS-specific remediation guidance.
3. **Session issuance** -- The Vouch server validates the signed assertion against the enrolled public key and issues a session token valid for 8 hours.
4. **Credential exchange** -- When a tool needs a credential, the CLI exchanges the session token for a service-specific credential (STS `AssumeRoleWithWebIdentity`, SSH certificate signing, GitHub App installation token, etc.). All requests include [HTTP Message Signatures (RFC 9421)](https://datatracker.ietf.org/doc/html/rfc9421) for request-level integrity.
5. **Tool consumption** -- The tool receives the short-lived credential and uses it normally. The credential expires on its own -- there is nothing to revoke or rotate.

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
- Communication between the CLI and agent uses a Unix domain socket with filesystem permissions restricting access to the owning user. Every incoming connection is verified using OS-level peer credentials (`SO_PEERCRED` on Linux, `getpeereid` on macOS) to confirm the connecting process runs as the same UID as the agent. Connections from a different UID are rejected and audit-logged.
- On startup, the agent validates that `~/.vouch/` is not a symlink and is owned by the current user, preventing symlink-based directory hijacking attacks.
- No credentials are persisted to disk. If the agent process stops, sessions must be re-established with a new `vouch login`.

### 3. Vouch server

- The server validates FIDO2 assertions (with identity federation through OIDC or [SAML 2.0](/docs/saml/) identity providers) and issues signed OIDC tokens (ES256 over P-256).
- Signing keys (OIDC ES256 and SSH CA Ed25519) are managed by AWS KMS — the server delegates signing operations and never holds private key material.
- The server does not store AWS credentials, SSH private keys, or GitHub tokens. It acts as an identity broker, not a secrets vault.
- Communication between CLI and server uses TLS 1.3. Authenticated requests include HTTP Message Signatures ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)) for request-level integrity verification.

---

## Threat model

For the complete STRIDE-based threat analysis — including threat actors, trust boundaries, assumptions, structured threat statements, and mitigations — see the dedicated [Threat Model](/docs/threat-model/) page.

---

## Encryption

### In transit

All communication between the Vouch CLI and server uses **TLS 1.3**. The FIDO2 assertion is transmitted over this encrypted channel.

### At rest

Vouch does not store credentials at rest. The server stores:

- **Enrolled public keys** -- The FIDO2 public key registered during enrollment. This is not sensitive (it cannot be used to impersonate the user).
- **User metadata** -- Email address, organization membership, and enrollment status.
- **Audit logs** -- Records of authentication events and credential issuance. Organization administrators can view and filter audit events from the admin dashboard.

![Audit Log page showing authentication events with type filters](/images/admin/admin-audit-log.png)

No AWS credentials, SSH keys, or GitHub tokens are stored on the server.

User data and metadata are protected with **document-level encryption** using HPKE ([RFC 9180](https://datatracker.ietf.org/doc/html/rfc9180)) with DHKEM(P-384), HKDF-SHA384, and AES-256-GCM. Each document is encrypted individually with its own encapsulated key — the encryption is bound to the document type and ID, preventing ciphertext relocation. The document encryption key pair is generated via AWS KMS (`GenerateDataKeyPairWithoutPlaintext`), and the private key is only decrypted at server startup using a KMS key with NitroTPM attestation (when available), ensuring the plaintext private key is only recoverable on attested EC2 instances.

Blind equality indexes (for lookups by email, etc.) use HMAC-SHA256 with a KMS-managed key, so the database never contains plaintext identifiers.

---

## FIDO2 security properties

Vouch uses [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) for all user authentication. Key security properties:

- **Origin binding** -- The authenticator (YubiKey) includes the relying party ID in the signed assertion. If an attacker stands up a phishing site at a different domain, the assertion will not validate against the Vouch server.
- **Hardware key storage** -- The private key is generated on the YubiKey's secure element and cannot be extracted, cloned, or backed up.
- **User verification** -- Every assertion requires the user's PIN and a physical touch of the key, providing two-factor authentication in a single gesture.
- **Replay protection** -- Each assertion includes a signature counter that the server tracks. Replayed assertions are rejected.
- **Attestation certificate chain validation** -- When `VOUCH_REQUIRE_ATTESTATION_CERT=true` is set, the server validates the authenticator's attestation certificate chain against pinned [Yubico root CA certificates](https://developers.yubico.com/PKI/). This cryptographically proves the key is a genuine Yubico device, not a software emulator or unknown authenticator. The server also extracts the FIDO AAGUID from the attestation certificate to identify the exact key model.

---

## OAuth 2.0 security architecture

FIDO2 proves the human is present. The OAuth 2.0 layer protects everything after — how the CLI identifies itself, how authorization requests are transmitted, and how tokens are bound to the device that requested them. Together, these form a [FAPI 2.0 Security Profile](https://openid.net/specs/fapi-security-profile-2_0-final.html).

**No shared secrets.** The CLI generates its own key pair and registers with the server automatically ([RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)). Client authentication uses `private_key_jwt` ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523)) — there is no client secret to extract from a binary or config file.

**Protected authorization requests.** Authorization parameters are sent directly to the server over a back-channel ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)) and signed as JWTs ([RFC 9101](https://datatracker.ietf.org/doc/html/rfc9101)). The browser redirect carries only an opaque reference — nothing sensitive in URLs, browser history, or referrer headers.

**Sender-constrained tokens.** Every access token is bound to the CLI's key pair via DPoP ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)). A stolen token cannot be used from a different machine.

**Audience-restricted tokens.** Each token includes a resource indicator ([RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707)) restricting it to a specific service. A token issued for AWS cannot be presented to GitHub.

**Request-level integrity.** Every authenticated request includes an HTTP Message Signature ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)). The CLI signs each request using the FAPI key pair stored in the OS keychain. The server verifies the signature before processing, providing cryptographic proof that the request was not tampered with and originated from the registered client.

**Fine-grained authorization requests.** Applications can request structured permissions using Rich Authorization Requests ([RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396)). Instead of flat scope strings, `authorization_details` objects describe the type, actions, and resources being requested — enabling precise, machine-readable authorization that goes beyond what scopes can express.

---

## Why request forgery is infeasible

The sections above describe individual security layers. Here is how they combine to make forging a CLI authentication request infeasible without physical possession of the user's enrolled YubiKey and knowledge of its PIN.

A successful login requires producing **all** of the following, and each is independently verified:

1. **FIDO2 assertion** -- A COSE signature that can only be produced by the YubiKey's private key, which never leaves the hardware secure element. The server verifies this signature against the public key registered during enrollment. Both the user presence (physical touch) and user verification (PIN) flags are checked server-side.
2. **Single-use challenge** -- The server generates 32 random bytes embedded in a signed state JWT with a 5-minute expiry. The challenge is atomically consumed on first use and bound into the `client_data_json` that the YubiKey signs — an old assertion cannot be paired with a new challenge.
3. **DPoP proof** -- The client proves possession of the same ES256 private key used during registration. Each proof carries a unique `jti` tracked in the database to prevent replay. The server can require a nonce for additional resistance to precomputation.
4. **Client assertion** -- OAuth client authentication uses a `private_key_jwt` (RFC 7523) signed with the device's ES256 key, with a 60-second lifetime and unique `jti`.
5. **Counter validation** -- The YubiKey's monotonic signature counter must strictly increase on each assertion. A cloned authenticator would have a stale counter, which the server detects and rejects.
6. **HTTP Message Signature** -- Every authenticated request is signed using the client's FAPI key pair ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)). The server verifies the signature covers the request method, path, and body, preventing request tampering even if TLS termination occurs at an intermediary.

### Attack scenarios

| Attack | Why it fails |
|---|---|
| **Replay a captured login** | Challenge is single-use (atomic DB check); DPoP `jti` is single-use |
| **Forge a FIDO2 assertion** | Requires the YubiKey's private key, which never leaves the hardware |
| **Steal an access token from the network** | Token is DPoP-bound — unusable without the device's private key |
| **Man-in-the-middle the challenge** | Challenge is signed in a state JWT with a server-only key; tampering is detected |
| **Tamper with a request in transit** | HTTP Message Signature verification fails — the signature covers the request method, path, and body |
| **Reuse an old assertion with a new challenge** | Challenge is embedded in `client_data_json`, which is signed by the YubiKey; mismatch is detected |
| **Clone the YubiKey** | Counter validation detects cloned authenticators |
| **Brute-force the PIN remotely** | PIN is verified locally by YubiKey hardware, which locks after 8 failed attempts |
| **Access the agent socket from another process** | Socket permissions (0600) restrict access; the agent verifies the connecting process has the same UID and PID |

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

## Compliance

Vouch's FAPI 2.0 security profile and hardware-backed authentication satisfy requirements across multiple compliance frameworks:

- **NIST 800-53** — IA-2 (identification/authentication), IA-5 (authenticator management), SC-23 (session authenticity)
- **SOC 2** — CC6.1 (logical access), CC6.8 (unauthorized access prevention), CC7.1 (detection)
- **FedRAMP** — Hardware MFA, DPoP sender-constrained tokens, non-extractable keys
- **HIPAA** — 164.312(d) (person authentication), 164.312(e) (transmission security)

Detailed control-by-control mappings are available in the [Vouch server documentation](https://docs.vouch.sh/reference/compliance.html).

---

## Incident response

If you suspect a security issue with the Vouch service or have discovered a vulnerability:

- Email **security@vouch.sh** with details.
- Include reproduction steps if possible.
- Do not disclose the issue publicly until it has been addressed.

If a YubiKey is lost or stolen, remove it from the user's account immediately to prevent unauthorized authentication.
