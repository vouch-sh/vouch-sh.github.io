---
title: "Threat Model"
linkTitle: "Threat Model"
description: "STRIDE-based threat analysis — threat actors, trust boundaries, assumptions, threats, and mitigations for the Vouch credential broker."
weight: 4
subtitle: "Structured threat analysis following the AWS Threat Composer methodology"
params:
  docsGroup: reference
---

This threat model documents the threats Vouch is designed to address, the assumptions the design relies on, and the mitigations in place. It follows the [STRIDE](https://en.wikipedia.org/wiki/STRIDE_(security)) framework and is structured after the [AWS Threat Composer](https://awslabs.github.io/threat-composer/) methodology.

For background on Vouch's security controls, see the [Security Model](/docs/security/) page. For system design details, see the [Architecture Overview](/docs/architecture/) page.

---

## System description

Vouch is a credential broker that replaces long-lived developer secrets (AWS access keys, SSH private keys, GitHub PATs) with short-lived, hardware-backed credentials. The system has three components:

- **Vouch CLI + Agent** — runs on the developer's machine, holds session state in memory, and serves credentials to tools via standard protocols (credential helper, SSH agent).
- **Vouch Server** — validates FIDO2 assertions, issues OIDC tokens, signs SSH certificates, and brokers credentials from external services.
- **External services** — AWS STS, GitHub Apps, SSH hosts, container registries, and other services that consume Vouch-issued credentials.

---

## Dataflow

```
┌──────────┐  FIDO2 assertion   ┌──────────────┐  kms:Sign (ES256)  ┌─────────┐
│  YubiKey │─-─────────────────►│ Vouch Server │──────────────────-►│ AWS KMS │
└──────────┘                    │              │◄──────────────────-┤         │
                                │              │  JWT signature     └─────────┘
┌──────────┐  session token     │              │
│Vouch CLI │◄────────────────-──┤              │  kms:Sign (Ed25519)
│ + Agent  │  (DPoP-bound)      │              │──────────────────-►┌─────────┐
│          │                    │              │◄──────────────────-┤ AWS KMS │
│          │  OIDC ID token     │              │  SSH cert signature└─────────┘
│          │──────────────────-►│              │
│          │  STS credentials   │              │  GitHub App key
│          │◄── ── ── ── ── ──--┤              │──────────────────-►┌─────────┐
│          │  (via AWS STS)     │              │◄─────────────────-─┤ GitHub  │
│          │                    │              │  installation token│ API     │
│          │  SSH cert request  │              │                    └─────────┘
│          │──────────────────-►│              │
│          │  signed SSH cert   │              │
│          │◄──────────────────-┤              │
└──────────┘                    └──────────────┘
      │
      │  short-lived credential
      ▼
┌──────────────────┐
│ Tool (aws, ssh,  │
│ git, docker)     │
└──────────────────┘
```

Data flows:

1. **Login** — YubiKey signs FIDO2 assertion → CLI sends to server over TLS → server validates against enrolled public key → server evaluates [device posture policies](/docs/device-posture/) (if active) → returns a DPoP-bound session token, which the CLI persists to its config file (owner-only permissions) and hands to the agent for in-memory caching. The server never issues refresh tokens — a new session requires a fresh FIDO2 assertion.
2. **AWS credential** — CLI presents session token + DPoP proof → server issues OIDC ID token (signed via KMS ES256) → CLI calls AWS STS `AssumeRoleWithWebIdentity` → STS returns temporary credentials.
3. **SSH certificate** — CLI sends signing request with session token → server delegates to KMS Ed25519 CA → returns signed SSH certificate → agent serves via SSH agent protocol.
4. **GitHub token** — CLI requests token with session token → server exchanges GitHub App credentials for installation access token → returns short-lived token to CLI.

All CLI ↔ server traffic uses TLS 1.3 (TLS 1.2 is accepted, restricted to BCP 195 AEAD cipher suites). Authenticated credential-API requests carry HTTP Message Signatures ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)) for request-level integrity; OAuth endpoints are protected by private_key_jwt client assertions and DPoP proofs instead. Brokered AWS credentials live only in agent memory. The session token is persisted to the CLI config file and the SSH key and certificate to `~/.ssh/`, all with owner-only file permissions.

---

## Assets

| Asset | Location | Sensitivity | Protection |
|---|---|---|---|
| **FIDO2 public keys** | Server database | Low — cannot impersonate users | Document-level encryption (HPKE) |
| **User metadata** (email, org) | Server database | Medium — PII | Document-level encryption (HPKE) + HMAC blind indexes |
| **OIDC signing keys** (ES256, RS256) | AWS KMS | Critical — issuance authority | KMS access policy, non-extractable |
| **Per-organization issuer keys** | Server database (sealed) | Critical — per-org issuance authority | Document-level encryption (HPKE) |
| **SSH CA key** (Ed25519) | AWS KMS | Critical — certificate authority | KMS access policy, non-extractable |
| **Document encryption key** (P-384) | Encrypted by KMS, decrypted at runtime | Critical — protects data at rest | KMS key policy restricts decryption to NitroTPM-attested instances |
| **Session MAC key** | AWS KMS | High — session token integrity | KMS HMAC operations, key never leaves KMS |
| **Session tokens** | Agent memory + CLI config file | High — grants credential access | DPoP-bound; on-disk copy restricted to owner (0600) |
| **Client key pair** (DPoP/FAPI) | OS keychain | High — token binding | Keychain-protected where available; owner-only file fallback |
| **SSH key + certificate** | Developer's `~/.ssh/` | Medium — SSH access for certificate lifetime | Owner-only file permissions; certificate expires with the session |
| **Audit logs** | Server database | Medium — forensic evidence | Unencrypted by design for queryability; emails masked to domain + HMAC correlation column; pull-based SIEM export (OCSF) |
| **SCIM tokens** | Server database (hashed) | High — provisioning authority | Stored as hashes, not reversible |

---

## Threat actors

| Actor | Description | Capability |
|---|---|---|
| **External attacker** | An adversary with no prior access to the organization's systems. Operates over the network. | Phishing, credential stuffing, man-in-the-middle attacks, domain spoofing, supply chain attacks on public packages. |
| **Malicious insider** | An authenticated employee or contractor who abuses legitimate access. | Valid Vouch session, access to internal systems, knowledge of organizational structure and tooling. |
| **Compromised endpoint** | Malware or an attacker with code execution on a developer's workstation. | Can read process memory, intercept IPC, access filesystem, and make network requests as the local user. |
| **Compromised server** | An attacker who has gained access to the Vouch server infrastructure. | Can issue sessions, sign tokens, and read enrolled public keys and audit logs. |
| **Supply chain attacker** | An adversary who tampers with Vouch binaries, dependencies, or distribution channels. | Can inject malicious code into CLI binaries or modify package repositories. |

---

## Trust boundaries

```
┌─────────────────────────────────────────────────────────────────────┐
│  Developer Workstation                                              │
│                                                                     │
│  ┌───────────┐    Unix socket    ┌────────────┐                     │
│  │ Vouch CLI │◄─────────────────►│ Vouch Agent│                     │
│  └─────┬─────┘  (owner-only)     └─────┬──────┘                     │
│        │                               │                            │
│  ┌─────┴─────┐                   ┌─────┴─────-─┐                    │
│  │  YubiKey  │                   │ In-memory   │                    │
│  │  (FIDO2)  │                   │ credentials │                    │
│  └───────────┘                   └────────────-┘                    │
│                                                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ TLS 1.3/1.2
              ─────────────┼──────────── Network boundary
                           │
┌──────────────────────────┴──────────────────────────────────────────┐
│  Vouch Server                                                       │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐         │
│  │ FIDO2 RP     │  │ OIDC Provider│  │ SSH CA (Ed25519)   │         │
│  │ (assertion   │  │ (ES256 via   │  │ (signing via       │         │
│  │  validation) │  │  AWS KMS)    │  │  AWS KMS)          │         │
│  └──────────────┘  └──────────────┘  └────────────────────┘         │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐                                 │
│  │ User store   │  │ Audit log    │                                 │
│  │ (public keys,│  │ (auth events,│                                 │
│  │  metadata)   │  │  issuance)   │                                 │
│  └──────────────┘  └──────────────┘                                 │
│                                                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ TLS 1.3/1.2
              ─────────────┼──────────── Service boundary
                           │
┌──────────────────────────┴──────────────────────────────────────────┐
│  External Services                                                  │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐       │
│  │ AWS STS  │  │ GitHub   │  │ SSH Hosts│  │ Container      │       │
│  │          │  │ Apps API │  │          │  │ Registries     │       │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Three trust boundaries separate the system:

1. **Hardware boundary** — The YubiKey's secure element. Private keys are generated on-device and cannot be extracted.
2. **Workstation boundary** — The developer's machine. The agent process, Unix socket, and in-memory credentials are protected by OS-level user isolation.
3. **Network boundary** — All communication between CLI and server, and between server and external services, uses TLS 1.3, with TLS 1.2 accepted using BCP 195-restricted AEAD cipher suites.

---

## Assumptions

These assumptions underpin the threat model. If an assumption is violated, the corresponding threats may not be adequately mitigated.

| ID | Assumption | Linked threats | Linked mitigations |
|---|---|---|---|
| **A1** | The YubiKey secure element correctly implements FIDO2 and does not leak private key material. | T-S1, T-S2 | FIDO2 origin binding, FIDO2 user verification (PIN + touch) |
| **A2** | The operating system enforces Unix socket file permissions and peer credential APIs (`SO_PEERCRED` / `getpeereid`), preventing other users from accessing the agent socket. | T-I1, T-E1 | Unix socket permissions, peer credential verification (T-E1 mitigation) |
| **A3** | TLS (1.3, or 1.2 restricted to BCP 195 AEAD suites) is not broken — an attacker cannot decrypt or tamper with data in transit. | T-T1, T-I2 | TLS transport encryption |
| **A4** | AWS STS, GitHub, and other external services correctly validate OIDC tokens and enforce their own access controls. | T-E2 | OIDC audience restriction |
| **A5** | The developer's workstation has not been fully compromised at the kernel level (no rootkit). User-space isolation is intact. | T-I1, T-E1 | In-memory only credentials, Unix socket permissions |
| **A6** | SCIM de-provisioning events are delivered promptly by the identity provider. | T-S3 | SCIM de-provisioning |
| **A7** | The Vouch server infrastructure is hardened and access-controlled (encrypted at rest, network isolation, audited access). | T-T2, T-E3 | Infrastructure hardening, audit log export |
| **A8** | Developers keep their YubiKey PINs secret and report lost or stolen keys promptly. | T-S2 | FIDO2 user verification (PIN + touch) |
| **A9** | AWS KMS correctly protects signing key material and enforces access controls. The KMS key policy for the document encryption key restricts decryption to NitroTPM-attested instances. | T-T2, T-E3 | KMS-managed signing keys, NitroTPM attestation, document-level encryption |

---

## Threats

Threats are organized using the [STRIDE](https://en.wikipedia.org/wiki/STRIDE_(security)) categories. Each threat follows the [AWS Threat Composer grammar](https://github.com/awslabs/threat-composer): **a [threat source] with [prerequisites] can [threat action], leading to [threat impact], negatively impacting [impacted assets]**.

### Spoofing

<div class="threat-table">

| ID | Threat | STRIDE | Severity | Priority |
|---|---|---|---|---|
| **T-S1** | An **external attacker** who controls a lookalike domain can **stand up a phishing site** to capture developer credentials, leading to **unauthorized access** to the developer's accounts, negatively impacting **session tokens**. | Spoofing | High | Low |
| **T-S2** | An **external attacker** with physical access to a stolen YubiKey and knowledge of the PIN can **authenticate as the enrolled user**, leading to **unauthorized credential issuance** for the session lifetime, negatively impacting **session tokens**. | Spoofing | High | Medium |
| **T-S3** | A **former employee** whose SCIM de-provisioning is delayed can **continue to use an active session**, leading to **unauthorized access** to organizational resources after offboarding, negatively impacting **session tokens**. | Spoofing | Medium | Low |

</div>

**Mitigations:**

- **T-S1**: FIDO2 origin binding prevents the YubiKey from signing assertions for unregistered domains. Even if a developer visits a phishing site, the authenticator will not produce a valid assertion. → [FIDO2 security properties](/docs/security/#fido2-security-properties)
- **T-S2**: YubiKey PINs provide a second factor — physical possession alone is insufficient. Keys should be reported lost immediately, and the enrolled credential should be removed from the user's account. Session lifetime (8 hours) limits the window.
- **T-S3**: SCIM integration enables automated de-provisioning — deactivation revokes active sessions and issued SSH certificates (via the CA revocation list). Sessions can also be revoked server-side. Outstanding AWS credentials (≤1 hour) expire naturally.

---

### Tampering

<div class="threat-table">

| ID | Threat | STRIDE | Severity | Priority |
|---|---|---|---|---|
| **T-T1** | An **external attacker** in a network position (e.g., compromised Wi-Fi) can **intercept and modify requests** between the CLI and server, leading to **session hijacking or credential injection**, negatively impacting **session tokens**. | Tampering | High | Low |
| **T-T2** | A **compromised server** operator can **modify the OIDC signing keys or SSH CA key**, leading to **issuance of fraudulent credentials** accepted by external services, negatively impacting **OIDC signing key** and **SSH CA key**. | Tampering | Critical | Low |
| **T-T3** | A **supply chain attacker** can **tamper with Vouch CLI binaries** during build or distribution, leading to **malicious code execution** on developer workstations, negatively impacting **session tokens**. | Tampering | Critical | Medium |

</div>

**Mitigations:**

- **T-T1**: All CLI-to-server communication uses TLS 1.3 (TLS 1.2 accepted with BCP 195 AEAD ciphers only). HTTP Message Signatures ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)) provide request-level integrity on the credential API — the CLI signs requests to `/v1/` endpoints using the FAPI key pair, and the server rejects any in-scope request with an invalid or missing signature (deny-by-default). OAuth endpoints are protected by private_key_jwt client assertions and DPoP proofs instead. DPoP ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)) binds tokens to the client's key pair — intercepted tokens cannot be used from a different machine, and presenting a DPoP-bound token as a plain Bearer token is rejected. For browser-based OIDC applications, PAR ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)) transmits authorization parameters server-side, keeping sensitive data out of URLs and browser history.
- **T-T2**: Signing keys (OIDC ES256/RS256 and SSH CA Ed25519) are managed by AWS KMS and cannot be extracted — server compromise does not expose signing key material. The document encryption private key is KMS-wrapped, with the key policy restricting decryption to NitroTPM-attested EC2 instances — preventing extraction even with full server access. The server stores no user credentials for external services — AWS and GitHub tokens are brokered on demand; the standing exception is the GitHub App private key, held in server configuration to authenticate as the App. Infrastructure controls (network isolation, access auditing) provide defense in depth. → [Shared responsibility](/docs/security/#shared-responsibility)
- **T-T3**: Release binaries include [SLSA Build Level 2](https://slsa.dev/) provenance and SBOM attestations (Sigstore-backed, verifiable with `gh attestation verify`) plus SHA256 checksums, and releases are independently rebuilt to verify reproducibility. APT and DNF repositories are GPG-signed and verified automatically by the package manager; Homebrew installs are pinned by SHA256 checksum. → [Supply chain security](/docs/security/#supply-chain-security)

---

### Repudiation

<div class="threat-table">

| ID | Threat | STRIDE | Severity | Priority |
|---|---|---|---|---|
| **T-R1** | A **malicious insider** can **deny performing an action** (e.g., accessing a production resource) if audit logs are insufficient, leading to **inability to attribute actions** during incident response, negatively impacting **audit logs**. | Repudiation | Medium | Low |
| **T-R2** | A **compromised server** can **tamper with or delete audit logs**, leading to **loss of forensic evidence**, negatively impacting **audit logs**. | Repudiation | High | Medium |

</div>

**Mitigations:**

- **T-R1**: Every credential issuance is tied to a hardware-verified FIDO2 identity. The Vouch server logs all authentication events and credential exchanges. AWS CloudTrail records STS credential usage with the Vouch-issued identity as the principal.
- **T-R2**: Audit logs should be continuously pulled into an immutable, external log store via the [SIEM export API](/docs/audit-export/) (OCSF projection, cursor-based polling) so that server compromise cannot erase the trail. → [Shared responsibility](/docs/security/#shared-responsibility)

---

### Information disclosure

<div class="threat-table">

| ID | Threat | STRIDE | Severity | Priority |
|---|---|---|---|---|
| **T-I1** | A **compromised endpoint** with code execution as the local user can **read the Vouch agent's process memory**, leading to **theft of the active session token and cached credentials**, negatively impacting **session tokens**. | Information disclosure | High | Medium |
| **T-I2** | An **external attacker** who compromises a network intermediary can **observe credential exchange traffic**, leading to **exposure of tokens or credentials**, negatively impacting **session tokens**. | Information disclosure | High | Low |
| **T-I3** | An **external attacker** can **enumerate the OIDC discovery endpoint** (`/.well-known/openid-configuration`) to **learn the server's signing keys and supported configuration**, leading to **information useful for targeted attacks**, negatively impacting **OIDC signing key** (public component only). | Information disclosure | Low | Low |

</div>

**Mitigations:**

- **T-I1**: DPoP binds tokens to the CLI's key pair — stolen tokens cannot be used from a different machine, and no refresh tokens exist to steal. Brokered AWS credentials live only in agent memory and are never written to disk (no `~/.aws/credentials`). The session token is persisted to the CLI config file and the SSH key and certificate to `~/.ssh/`, all with owner-only permissions — an attacker running as the local user can read these, which is why DPoP binding and short lifetimes bound the impact. Session lifetime is limited to 8 hours (SSH certificates expire with the session), and AWS STS credentials expire within 1 hour. Full endpoint compromise with kernel access is out of scope (see [assumption A5](#assumptions)).
- **T-I2**: TLS encrypts all traffic in transit (1.3 preferred, 1.2 restricted to BCP 195 AEAD suites). DPoP provides an additional layer — even if a token is somehow intercepted, it cannot be replayed from another client.
- **T-I3**: OIDC discovery is public by design (required for AWS OIDC federation). The exposed information (issuer URL, JWKS, supported algorithms) does not enable impersonation. Private keys are never exposed through these endpoints.

---

### Denial of service

<div class="threat-table">

| ID | Threat | STRIDE | Severity | Priority |
|---|---|---|---|---|
| **T-D1** | An **external attacker** can **flood the Vouch server** with authentication requests, leading to **developers being unable to obtain credentials**, negatively impacting **session tokens**. | Denial of service | Medium | Low |
| **T-D2** | An **external attacker** can **disrupt network connectivity** between the CLI and the Vouch server, leading to **inability to establish new sessions or obtain fresh credentials**, negatively impacting **session tokens**. | Denial of service | Medium | Low |

</div>

**Mitigations:**

- **T-D1**: The Vouch server implements rate limiting and is deployed behind infrastructure-level DDoS protection. Authentication requires a valid FIDO2 assertion, making automated abuse expensive.
- **T-D2**: Cached credentials remain valid for their remaining lifetime (up to 8 hours for sessions and SSH certificates, 1 hour for AWS STS). Developers can continue working with existing credentials during an outage. → [Availability and Failure Modes](/docs/availability/)

---

### Elevation of privilege

<div class="threat-table">

| ID | Threat | STRIDE | Severity | Priority |
|---|---|---|---|---|
| **T-E1** | A **compromised endpoint** can **access the Unix domain socket** and **use the active session to request credentials for any role the user is authorized for**, leading to **unauthorized access to cloud resources** within the user's permission set, negatively impacting **session tokens**. | Elevation of privilege | High | Medium |
| **T-E2** | A **malicious insider** can **use their valid Vouch session to access resources beyond their intended scope** if IAM roles are overly permissive, leading to **unauthorized access to production systems or sensitive data**, negatively impacting **session tokens**. | Elevation of privilege | High | Medium |
| **T-E3** | A **compromised server** can **issue sessions for any enrolled user**, leading to **impersonation of any developer** and access to their authorized resources, negatively impacting **OIDC signing key**, **SSH CA key**, **user metadata**, and **audit logs**. | Elevation of privilege | Critical | Low |

</div>

**Mitigations:**

- **T-E1**: The Unix socket is restricted to the owning user by filesystem permissions. Additionally, the agent verifies peer credentials (`SO_PEERCRED` / `getpeereid`) on every connection to its IPC socket, confirming the connecting process has the same UID — rejected connections are audit-logged for forensic visibility. The companion SSH agent socket relies on the same owner-only directory and socket permissions. On startup, the agent validates that its socket directory (`$XDG_RUNTIME_DIR/vouch/`, or `~/.cache/vouch/` where `XDG_RUNTIME_DIR` is unset) is not a symlink and is owned by the current user, preventing directory hijacking. DPoP prevents extracted tokens from being used on a different machine. Credential scope is limited to the user's authorized roles — the attacker cannot escalate beyond what the user could already access. This threat is bounded by session lifetime (8 hours, which also bounds SSH certificates) and AWS credential lifetime (≤1 hour).
- **T-E2**: IAM roles should follow least-privilege principles. Vouch enables fine-grained role mapping per user via OIDC claims. CloudTrail provides full attribution of which user assumed which role. → [Shared responsibility](/docs/security/#shared-responsibility)
- **T-E3**: Signing keys are in AWS KMS (non-extractable). The document encryption private key is KMS-wrapped, with the key policy restricting decryption to NitroTPM-attested instances — an attacker with disk or database access alone cannot decrypt user data. Server infrastructure is hardened with network isolation, encrypted storage, audited access, and minimal attack surface. The server does not store user credentials for external services — it brokers them — so compromise enables credential issuance (while the attacker maintains access) but not extraction of stored user secrets.

---

## Mitigation summary

| Control | Threats addressed | Layer |
|---|---|---|
| **FIDO2 origin binding** | T-S1 (phishing) | Hardware |
| **FIDO2 user verification (PIN + touch)** | T-S2 (stolen key) | Hardware |
| **DPoP sender-constrained tokens** | T-T1, T-I1, T-I2, T-E1 (token theft, replay) | Protocol |
| **PAR + signed JWTs** | T-T1 (parameter injection) | Protocol |
| **TLS 1.3 / 1.2 (BCP 195 ciphers)** | T-T1, T-I2 (network interception) | Transport |
| **In-memory credential cache** (brokered AWS credentials) | T-I1 (disk exfiltration) | Application |
| **Short credential lifetimes** | T-S3, T-I1, T-E1 (blast radius) | Application |
| **SCIM de-provisioning** | T-S3 (offboarding) | Identity |
| **OIDC audience restriction** | T-E2 (cross-service abuse) | Protocol |
| **SLSA Build Level 2 provenance** | T-T3 (supply chain) | Build |
| **Audit logging + CloudTrail** | T-R1, T-R2 (repudiation) | Operational |
| **Rate limiting + DDoS protection** | T-D1 (server flood) | Infrastructure |
| **IPC peer credential verification** | T-E1, T-I1 (cross-user socket access) | Application |
| **Directory symlink and ownership validation** | T-E1 (directory hijacking) | Application |
| **Credential caching** | T-D2 (outage resilience) | Application |
| **KMS-managed signing keys** | T-T2, T-E3 (key extraction) | Cryptographic |
| **NitroTPM attestation** | T-T2, T-E3 (runtime key protection) | Infrastructure |
| **Document-level encryption (HPKE)** | T-E3 (data-at-rest protection) | Application |
| **HMAC blind indexes** | T-I3 (database-level identifier protection) | Application |
| **HTTP message signatures (RFC 9421)** | T-T1 (request tampering) | Protocol |
| **Device posture policies (Dogwood)** | T-E1, T-E2 (compromised endpoint, insider abuse) | Application |

---

## Out of scope

The following threats are explicitly out of scope for this threat model:

| Threat | Rationale |
|---|---|
| **Kernel-level endpoint compromise** | If an attacker has root/kernel access, all user-space isolation (process memory, socket permissions) is bypassed. Endpoint detection and response (EDR) tools are the appropriate mitigation layer. |
| **Vulnerabilities in external services** | AWS STS, GitHub APIs, container registries, and SSH implementations have their own security models. Vouch trusts their documented behavior. |
| **Cryptographic breaks** | If ECDSA (P-256), Ed25519, or TLS 1.3 are broken, the impact extends far beyond Vouch. |
| **Physical coercion** | An attacker who can physically compel a developer to authenticate is outside the scope of a technical threat model. |
| **YubiKey hardware vulnerabilities** | Vouch trusts the FIDO2 implementation of enrolled authenticators. Hardware side-channel attacks on the YubiKey secure element are outside scope. |

---

## Validation

- **Automated scanning** -- Dependency auditing (`cargo deny`: advisories, licenses, sources) and a deny-level lint gate (unsafe code and panics denied workspace-wide) run on every commit; dependency review runs on every pull request; container images are vulnerability-scanned on push.
- **SLSA provenance** -- Release binaries include Build Level 2 provenance and SBOM attestations, verifiable against the source repository with `gh attestation verify`; releases are independently rebuilt to verify reproducibility.
- **Protocol conformance** -- WebAuthn assertion validation is covered by a spec-referenced test suite (WebAuthn Level 2), OIDC behavior is validated against the OpenID Foundation conformance suite, and HTTP Message Signatures are tested against the official RFC 9421 test vectors.
- **Client diagnostics** -- `vouch doctor` performs runtime checks (YubiKey and agent health, server connectivity and clock skew, session validity, SSH/EKS/SSM configuration) to validate the local setup.

## Review schedule

This threat model is reviewed quarterly and after any significant architecture change. The revision history below tracks updates.

---

## Revision history

| Date | Change |
|---|---|
| 2026-08-14 | Accuracy audit against the v2026.8.3 source. TLS: 1.2 accepted alongside 1.3 (BCP 195 AEAD ciphers only). Credentials at rest: session token persists to the CLI config file and the SSH key/certificate to `~/.ssh/` (owner-only permissions); the agent's brokered-credential cache is memory-only; no refresh tokens are issued. SSH certificates are session-bound (8 hours), revoked via CA KRL on de-provisioning. Blind-index keys are locally derived (the KMS HMAC key protects session tokens). Audit logs are unencrypted by design with a pull-based OCSF SIEM export. Supply chain: SLSA Build Level 2; APT/DNF GPG-verified, Homebrew checksum-pinned. NitroTPM binding enforced via KMS key policy. Documented the GitHub App key exception, per-org issuer keys, RS256 signing key, and RFC 9421 signing scope (`/v1/` deny-by-default). Corrected the validation section. |
| 2026-08-14 | Replaced CEL with Dogwood (Cedar-based) as the device posture policy engine, adding history-aware policies evaluated at token issuance and token exchange (shipped in v2026.8.3). Updated mitigation summary table. |
| 2026-03-23 | Added HTTP Message Signatures (RFC 9421) as a mitigation for request tampering (T-T1). Added device posture policies (CEL) as a mitigation for compromised endpoints and insider abuse (T-E1, T-E2). Updated login dataflow to include device posture evaluation step. Updated mitigation summary table. |
| 2026-03-02 | Updated TLS requirement to 1.3 (TLS 1.2 removed). Added KMS signing architecture, NitroTPM attestation, and document-level encryption. Added assets inventory, validation, and review schedule sections. Aligned to AWS Threat Composer methodology: added dataflow diagram, impacted assets to all threat statements, priority metadata, and assumption-to-mitigation links. Updated mitigation summary table. |
| 2026-02-28 | Initial threat model published on vouch.sh, structured using STRIDE and the AWS Threat Composer methodology. |
