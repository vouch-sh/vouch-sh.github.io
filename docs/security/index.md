# Security Model

> How Vouch protects credentials at every layer — data flow, threat model, credential lifecycle, and supply chain integrity.

Source: https://vouch.sh/docs/security/
Last updated: 2026-08-18

---
Vouch brokers SSH certificates, AWS STS tokens, GitHub installation tokens, and container registry passwords. This page explains how those credentials are protected at every layer.

## Executive summary

- The Vouch server does not store users&#39; AWS credentials, SSH private keys, GitHub tokens, or registry passwords. It brokers short-lived credentials after hardware-backed authentication.
- User authentication is based on FIDO2/WebAuthn assertions from enrolled YubiKeys, requiring both the key and user verification.
- Credentials are scoped to one authenticated user, short-lived, and expire automatically; brokered AWS credentials are cached only in the local agent&#39;s memory.
- Server-side signing keys are managed by AWS KMS, and authenticated requests use modern OAuth and FAPI protections including DPoP, PAR, private-key client authentication, and HTTP Message Signatures.
- Security reviewers should also read the [Threat Model](/docs/threat-model/), [Architecture](/docs/architecture/), [Availability](/docs/availability/), and [Migration](/docs/migration/) guides.

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

1. **FIDO2 assertion** -- The YubiKey signs a challenge using a private key that never leaves the hardware. The assertion includes origin binding (preventing phishing) and user verification (PIN &#43; touch).
2. **Device posture evaluation** -- If active [posture policies](/docs/device-posture/) exist, the server evaluates the device&#39;s security state (disk encryption, firewall, EDR, screen lock, etc.) against configured policies. If any policy fails, access is denied with OS-specific remediation guidance.
3. **Session issuance** -- The Vouch server validates the signed assertion against the enrolled public key and issues a session token valid for 8 hours.
4. **Credential exchange** -- When a tool needs a credential, the CLI exchanges the session token for a service-specific credential (STS `AssumeRoleWithWebIdentity`, SSH certificate signing, GitHub App installation token, etc.). Credential API requests include [HTTP Message Signatures (RFC 9421)](https://datatracker.ietf.org/doc/html/rfc9421) for request-level integrity.
5. **Tool consumption** -- The tool receives the short-lived credential and uses it normally. The credential expires on its own -- there is nothing to revoke or rotate.

---

## Credential lifecycle

All Vouch credentials share these properties:

| Property | Detail |
|---|---|
| **Storage** | Brokered AWS credentials are cached only in agent memory. The session token is persisted to the CLI config file, and the SSH key and certificate to `~/.ssh/`, all with owner-only permissions (0600). |
| **Lifetime** | Session: 8 hours. AWS STS: up to 1 hour. SSH certificate: 8 hours (expires with the session). GitHub token: 1 hour. |
| **Scope** | Tied to a single authenticated user. The session token is DPoP-bound and unusable from another machine. |
| **Revocation** | Sessions can be revoked server-side (e.g., via SCIM de-provisioning), which also revokes issued SSH certificates via the CA revocation list. Outstanding AWS and GitHub tokens expire within the hour. |
| **Rotation** | Not applicable -- credentials are issued fresh on each request and expire automatically. |

AWS credentials never touch `~/.aws/credentials` or environment variables, so there are no static keys for malware to scan for. The on-disk state that does exist — the session token and SSH key — is owner-only and device-bound: the session token cannot be used without the client key pair, and the SSH certificate expires with the session.

---

## Trust boundaries

Vouch operates across three trust boundaries:

### 1. Hardware key (YubiKey)

- Private key material is generated on the YubiKey and **never exported**.
- FIDO2 assertions are origin-bound -- the key will not sign challenges from phishing domains.
- User verification requires both a PIN and a physical touch.

### 2. Local machine (CLI &#43; Agent)

- The Vouch agent runs as a user-level process and holds session material in memory.
- Communication between the CLI and agent uses a Unix domain socket with filesystem permissions restricting access to the owning user. Every incoming connection is verified using OS-level peer credentials (`SO_PEERCRED` on Linux, `getpeereid` on macOS) to confirm the connecting process runs as the same UID as the agent. Connections from a different UID are rejected and audit-logged.
- On startup, the agent validates that its socket directory (`$XDG_RUNTIME_DIR/vouch/`, or `~/.cache/vouch/` where `XDG_RUNTIME_DIR` is unset) is not a symlink and is owned by the current user, preventing symlink-based directory hijacking attacks.
- The agent&#39;s brokered-credential cache is memory-only. The session token is persisted to the CLI config file with owner-only permissions, so a restarted agent recovers the session until it expires — a new `vouch login` is only needed after expiry or revocation.

### 3. Vouch server

- The server validates FIDO2 assertions (with identity federation through OIDC or [SAML 2.0](/docs/saml/) identity providers) and issues signed OIDC tokens (ES256 over P-256; RS256 over RSA-3072 also supported).
- Primary signing keys (OIDC and SSH CA Ed25519) are managed by AWS KMS — signing is delegated and the key material is non-extractable. Per-organization issuer keys are stored sealed with document-level encryption.
- The server does not store users&#39; AWS credentials, SSH private keys, or GitHub tokens. It acts as an identity broker, not a secrets vault. (The GitHub App private key used to mint installation tokens is the standing configuration exception.)
- Communication between CLI and server uses TLS 1.3, with TLS 1.2 accepted using BCP 195-restricted AEAD cipher suites. Authenticated credential-API requests include HTTP Message Signatures ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)) for request-level integrity verification.

---

## Threat model

For the complete STRIDE-based threat analysis — including threat actors, trust boundaries, assumptions, structured threat statements, and mitigations — see the dedicated [Threat Model](/docs/threat-model/) page.

---

## Encryption

### In transit

All communication between the Vouch CLI and server uses **TLS 1.3**, with TLS 1.2 accepted and restricted to [BCP 195](https://www.rfc-editor.org/info/bcp195) AEAD cipher suites. The FIDO2 assertion is transmitted over this encrypted channel.

### At rest

Vouch does not store credentials at rest. The server stores:

- **Enrolled public keys** -- The FIDO2 public key registered during enrollment. This is not sensitive (it cannot be used to impersonate the user).
- **User metadata** -- Email address, organization membership, and enrollment status.
- **Audit logs** -- Records of authentication events and credential issuance, stored in a separate table that is unencrypted by design for queryability, with email addresses masked to domain plus an HMAC correlation column. Organization administrators can view and filter audit events from the admin dashboard, and SIEM tooling can pull them continuously via the [export API](/docs/audit-export/) (OCSF projection).

![Audit Log page showing authentication events with type filters](/images/admin/admin-audit-log.png)

No AWS credentials, SSH keys, or GitHub tokens are stored on the server.

User data and metadata are protected with **document-level encryption** using HPKE ([RFC 9180](https://datatracker.ietf.org/doc/html/rfc9180)) with DHKEM(P-384), HKDF-SHA384, and AES-256-GCM. Each document is encrypted individually with its own encapsulated key — the encryption is bound to the document type and ID, preventing ciphertext relocation. The document encryption key pair is generated via AWS KMS (`GenerateDataKeyPairWithoutPlaintext`) and the private key is decrypted at server startup via KMS; the KMS key policy restricts that decryption to NitroTPM-attested EC2 instances, so the plaintext private key is only recoverable on attested hosts.

Blind equality indexes (for lookups by email, etc.) use HMAC-SHA256 with a key derived from the document encryption public key, so the database never contains plaintext identifiers.

---

## FIDO2 security properties

Vouch uses [FIDO2/WebAuthn](https://fidoalliance.org/fido2/) for all user authentication. Key security properties:

- **Origin binding** -- The authenticator (YubiKey) includes the relying party ID in the signed assertion. If an attacker stands up a phishing site at a different domain, the assertion will not validate against the Vouch server.
- **Hardware key storage** -- The private key is generated on the YubiKey&#39;s secure element and cannot be extracted, cloned, or backed up.
- **User verification** -- Every assertion requires the user&#39;s PIN and a physical touch of the key, providing two-factor authentication in a single gesture.
- **Replay protection** -- Each assertion includes a signature counter that the server tracks. Replayed assertions are rejected.
- **Attestation certificate chain validation** -- When `VOUCH_REQUIRE_ATTESTATION_CERT=true` is set, the server validates the authenticator&#39;s attestation certificate chain against pinned [Yubico root CA certificates](https://developers.yubico.com/PKI/). This cryptographically proves the key is a genuine Yubico device, not a software emulator or unknown authenticator. The server also extracts the FIDO AAGUID from the attestation certificate to identify the exact key model.

---

## OAuth 2.0 security architecture

FIDO2 proves the human is present. The OAuth 2.0 layer protects everything after — how the CLI identifies itself, how authorization requests are transmitted, and how tokens are bound to the device that requested them. Together, these form a [FAPI 2.0 Security Profile](https://openid.net/specs/fapi-security-profile-2_0-final.html).

**No shared secrets.** The CLI generates its own key pair and registers with the server automatically ([RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)). Client authentication uses `private_key_jwt` ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523)) — there is no client secret to extract from a binary or config file.

**Protected authorization requests.** Authorization parameters are sent directly to the server over a back-channel ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)) and signed as JWTs ([RFC 9101](https://datatracker.ietf.org/doc/html/rfc9101)). The browser redirect carries only an opaque reference — nothing sensitive in URLs, browser history, or referrer headers.

**Sender-constrained tokens.** Every access token is bound to the CLI&#39;s key pair via DPoP ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)). A stolen token cannot be used from a different machine. For service-to-service scenarios, Mutual TLS ([RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705)) provides an alternative — the token&#39;s `cnf` claim contains the SHA-256 thumbprint of the client&#39;s TLS certificate, and resource servers validate that the certificate presented at the TLS layer matches.

**Audience-restricted tokens.** Each token includes a resource indicator ([RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707)) restricting it to a specific service. A token issued for AWS cannot be presented to GitHub.

**Request-level integrity.** Authenticated credential-API requests (under `/v1/`) include an HTTP Message Signature ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)), enforced deny-by-default — the server rejects any in-scope request with a missing or invalid signature. The CLI signs each request using the FAPI key pair stored in the OS keychain. OAuth endpoints are protected by `private_key_jwt` client assertions and DPoP proofs instead.

**Fine-grained authorization requests.** Applications can request structured permissions using Rich Authorization Requests ([RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396)). Instead of flat scope strings, `authorization_details` objects describe the type, actions, and resources being requested — enabling precise, machine-readable authorization that goes beyond what scopes can express.

---

## Why request forgery is infeasible

The sections above describe individual security layers. Here is how they combine to make forging a CLI authentication request infeasible without physical possession of the user&#39;s enrolled YubiKey and knowledge of its PIN.

A successful login requires producing **all** of the following, and each is independently verified:

1. **FIDO2 assertion** -- A COSE signature that can only be produced by the YubiKey&#39;s private key, which never leaves the hardware secure element. The server verifies this signature against the public key registered during enrollment. Both the user presence (physical touch) and user verification (PIN) flags are checked server-side.
2. **Single-use challenge** -- The server generates 32 random bytes embedded in a signed state JWT with a 5-minute expiry. The challenge is atomically consumed on first use and bound into the `client_data_json` that the YubiKey signs — an old assertion cannot be paired with a new challenge.
3. **DPoP proof** -- The client proves possession of the same ES256 private key used during registration. Each proof carries a unique `jti` tracked in the database to prevent replay. The server can require a nonce for additional resistance to precomputation.
4. **Client assertion** -- OAuth client authentication uses a `private_key_jwt` (RFC 7523) signed with the device&#39;s ES256 key, with a 60-second lifetime and unique `jti`.
5. **Counter validation** -- The YubiKey&#39;s monotonic signature counter must strictly increase on each assertion. A cloned authenticator would have a stale counter, which the server detects and rejects.
6. **HTTP Message Signature** -- Every credential-API request is signed using the client&#39;s FAPI key pair ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)). The server verifies the signature covers the request method, path, and body, preventing request tampering even if TLS termination occurs at an intermediary.

### Attack scenarios

| Attack | Why it fails |
|---|---|
| **Replay a captured login** | Challenge is single-use (atomic DB check); DPoP `jti` is single-use |
| **Forge a FIDO2 assertion** | Requires the YubiKey&#39;s private key, which never leaves the hardware |
| **Steal an access token from the network** | Token is DPoP-bound or certificate-bound — unusable without the device&#39;s private key or matching TLS certificate |
| **Man-in-the-middle the challenge** | Challenge is signed in a state JWT with a server-only key; tampering is detected |
| **Tamper with a request in transit** | HTTP Message Signature verification fails — the signature covers the request method, path, and body |
| **Reuse an old assertion with a new challenge** | Challenge is embedded in `client_data_json`, which is signed by the YubiKey; mismatch is detected |
| **Clone the YubiKey** | Counter validation detects cloned authenticators |
| **Brute-force the PIN remotely** | PIN is verified locally by YubiKey hardware, which locks after 8 failed attempts |
| **Present a certificate-bound token without the matching certificate** | The `x5t#S256` thumbprint in the token&#39;s `cnf` claim is validated against the client certificate presented in the TLS handshake; mismatch is rejected |
| **Authorization server mix-up** | The `iss` parameter in authorization responses ([RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207)) lets clients verify they are communicating with the expected authorization server |
| **Access the agent socket from another process** | Socket permissions (0600) restrict access; the agent verifies the connecting process has the same UID via OS peer credentials |

---

## Supply chain security

### SLSA provenance

Vouch release binaries are built with [SLSA Build Level 3](https://slsa.dev/) provenance via GitHub Artifact Attestations (Sigstore-backed). Builds run inside a dedicated reusable workflow, so the attestation signing identity cannot be influenced by the build steps themselves. Each release includes provenance and SBOM attestations that you can verify with the GitHub CLI, pinning the builder:

```bash
gh attestation verify vouch-v2026.8.4-x86_64-unknown-linux-musl.tar.gz \
  --owner vouch-sh \
  --signer-workflow vouch-sh/vouch/.github/workflows/reusable-build.yml
```

This confirms the binary was built from the expected source repository by the pinned build workflow. Releases are also independently rebuilt and hash-compared to verify reproducibility.

### SHA256 checksums

Every release includes a `SHA256SUMS.txt` file covering all artifacts, plus a `.sha256` file per artifact. Verify downloaded binaries:

```bash
sha256sum --check SHA256SUMS.txt
```

### Package manager verification

APT and DNF installs are verified automatically by the package manager: the repositories are signed with Vouch&#39;s published GPG key, and the documented setup configures `signed-by` / `gpgcheck` accordingly. Homebrew installs are pinned to the release&#39;s SHA256 checksum by the formula.

---

## Shared responsibility

### Vouch&#39;s responsibilities

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

Vouch&#39;s FAPI 2.0 security profile and hardware-backed authentication satisfy requirements across multiple compliance frameworks:

- **NIST 800-53** — IA-2 (identification/authentication), IA-5 (authenticator management), SC-23 (session authenticity)
- **SOC 2** — CC6.1 (logical access), CC6.8 (unauthorized access prevention), CC7.1 (detection)
- **FedRAMP** — Hardware MFA, DPoP and mTLS sender-constrained tokens, non-extractable keys
- **HIPAA** — 164.312(d) (person authentication), 164.312(e) (transmission security)

Detailed control-by-control mappings are available in the [Vouch server documentation](https://docs.vouch.sh/reference/compliance.html).

---

## Incident response

If you suspect a security issue with the Vouch service or have discovered a vulnerability:

- Email **security@vouch.sh** with details.
- Include reproduction steps if possible.
- Do not disclose the issue publicly until it has been addressed.

If a YubiKey is lost or stolen, remove it from the user&#39;s account immediately to prevent unauthorized authentication.
