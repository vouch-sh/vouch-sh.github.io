---
title: "Threat Model"
linkTitle: "Threat Model"
description: "STRIDE-based threat analysis — threat actors, trust boundaries, assumptions, threats, and mitigations for the Vouch credential broker."
weight: 19
subtitle: "Structured threat analysis following the AWS Threat Composer methodology"
params:
  docsGroup: manage
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
│  ┌─────┴─────┐                   ┌─────┴──────┐                     │
│  │  YubiKey  │                   │ In-memory   │                     │
│  │  (FIDO2)  │                   │ credentials │                     │
│  └───────────┘                   └────────────┘                     │
│                                                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ TLS 1.2+
              ─────────────┼──────────── Network boundary
                           │
┌──────────────────────────┴──────────────────────────────────────────┐
│  Vouch Server                                                       │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐        │
│  │ FIDO2 RP     │  │ OIDC Provider│  │ SSH CA (Ed25519)   │        │
│  │ (assertion   │  │ (ES256 JWT   │  │ (certificate       │        │
│  │  validation) │  │  issuance)   │  │  signing)          │        │
│  └──────────────┘  └──────────────┘  └────────────────────┘        │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐                                 │
│  │ User store   │  │ Audit log    │                                 │
│  │ (public keys,│  │ (auth events,│                                 │
│  │  metadata)   │  │  issuance)   │                                 │
│  └──────────────┘  └──────────────┘                                 │
│                                                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ TLS 1.2+
              ─────────────┼──────────── Service boundary
                           │
┌──────────────────────────┴──────────────────────────────────────────┐
│  External Services                                                  │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐      │
│  │ AWS STS  │  │ GitHub   │  │ SSH Hosts│  │ Container      │      │
│  │          │  │ Apps API │  │          │  │ Registries     │      │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Three trust boundaries separate the system:

1. **Hardware boundary** — The YubiKey's secure element. Private keys are generated on-device and cannot be extracted.
2. **Workstation boundary** — The developer's machine. The agent process, Unix socket, and in-memory credentials are protected by OS-level user isolation.
3. **Network boundary** — All communication between CLI and server, and between server and external services, uses TLS 1.2+.

---

## Assumptions

These assumptions underpin the threat model. If an assumption is violated, the corresponding threats may not be adequately mitigated.

| ID | Assumption | Linked threats |
|---|---|---|
| **A1** | The YubiKey secure element correctly implements FIDO2 and does not leak private key material. | T-S1, T-S2 |
| **A2** | The operating system enforces Unix socket file permissions, preventing other users from accessing the agent socket. | T-I1, T-E1 |
| **A3** | TLS 1.2+ is not broken — an attacker cannot decrypt or tamper with data in transit. | T-T1, T-I2 |
| **A4** | AWS STS, GitHub, and other external services correctly validate OIDC tokens and enforce their own access controls. | T-E2 |
| **A5** | The developer's workstation has not been fully compromised at the kernel level (no rootkit). User-space isolation is intact. | T-I1, T-E1 |
| **A6** | SCIM de-provisioning events are delivered promptly by the identity provider. | T-S3 |
| **A7** | The Vouch server infrastructure is hardened and access-controlled (encrypted at rest, network isolation, audited access). | T-T2, T-E3 |
| **A8** | Developers keep their YubiKey PINs secret and report lost or stolen keys promptly. | T-S2 |

---

## Threats

Threats are organized using the [STRIDE](https://en.wikipedia.org/wiki/STRIDE_(security)) categories. Each threat follows the structured grammar: **a [threat actor] with [prerequisites] can [action], leading to [impact]**.

### Spoofing

<div class="threat-table">

| ID | Threat | STRIDE | Severity |
|---|---|---|---|
| **T-S1** | An **external attacker** who controls a lookalike domain can **stand up a phishing site** to capture developer credentials, leading to **unauthorized access** to the developer's accounts. | Spoofing | High |
| **T-S2** | An **external attacker** with physical access to a stolen YubiKey and knowledge of the PIN can **authenticate as the enrolled user**, leading to **unauthorized credential issuance** for the session lifetime. | Spoofing | High |
| **T-S3** | A **former employee** whose SCIM de-provisioning is delayed can **continue to use an active session**, leading to **unauthorized access** to organizational resources after offboarding. | Spoofing | Medium |

</div>

**Mitigations:**

- **T-S1**: FIDO2 origin binding prevents the YubiKey from signing assertions for unregistered domains. Even if a developer visits a phishing site, the authenticator will not produce a valid assertion. → [FIDO2 security properties](/docs/security/#fido2-security-properties)
- **T-S2**: YubiKey PINs provide a second factor — physical possession alone is insufficient. Keys should be reported lost immediately, and the enrolled credential should be removed from the user's account. Session lifetime (8 hours) limits the window.
- **T-S3**: SCIM integration enables automated de-provisioning. Sessions can also be revoked server-side. Outstanding short-lived credentials (≤1 hour) expire naturally.

---

### Tampering

<div class="threat-table">

| ID | Threat | STRIDE | Severity |
|---|---|---|---|
| **T-T1** | An **external attacker** in a network position (e.g., compromised Wi-Fi) can **intercept and modify requests** between the CLI and server, leading to **session hijacking or credential injection**. | Tampering | High |
| **T-T2** | A **compromised server** operator can **modify the OIDC signing keys or SSH CA key**, leading to **issuance of fraudulent credentials** accepted by external services. | Tampering | Critical |
| **T-T3** | A **supply chain attacker** can **tamper with Vouch CLI binaries** during build or distribution, leading to **malicious code execution** on developer workstations. | Tampering | Critical |

</div>

**Mitigations:**

- **T-T1**: All CLI-to-server communication uses TLS 1.2+. DPoP ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)) binds tokens to the client's key pair — intercepted tokens cannot be used from a different machine. PAR ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)) transmits authorization parameters server-side, keeping sensitive data out of URLs and browser history.
- **T-T2**: Server access is restricted by infrastructure controls (network isolation, access auditing). The server does not store external service credentials — it brokers them on demand. Compromise allows credential issuance but not direct access to AWS keys or SSH private keys. → [Shared responsibility](/docs/security/#shared-responsibility)
- **T-T3**: Release binaries include [SLSA Level 3](https://slsa.dev/) provenance attestations and SHA256 checksums. Package manager installs (Homebrew, APT, DNF) verify signatures automatically. → [Supply chain security](/docs/security/#supply-chain-security)

---

### Repudiation

<div class="threat-table">

| ID | Threat | STRIDE | Severity |
|---|---|---|---|
| **T-R1** | A **malicious insider** can **deny performing an action** (e.g., accessing a production resource) if audit logs are insufficient, leading to **inability to attribute actions** during incident response. | Repudiation | Medium |
| **T-R2** | A **compromised server** can **tamper with or delete audit logs**, leading to **loss of forensic evidence**. | Repudiation | High |

</div>

**Mitigations:**

- **T-R1**: Every credential issuance is tied to a hardware-verified FIDO2 identity. The Vouch server logs all authentication events and credential exchanges. AWS CloudTrail records STS credential usage with the Vouch-issued identity as the principal.
- **T-R2**: Audit logs should be exported to an immutable, external log store (e.g., AWS CloudWatch, a SIEM) so that server compromise cannot erase the trail. → [Shared responsibility](/docs/security/#shared-responsibility)

---

### Information disclosure

<div class="threat-table">

| ID | Threat | STRIDE | Severity |
|---|---|---|---|
| **T-I1** | A **compromised endpoint** with code execution as the local user can **read the Vouch agent's process memory**, leading to **theft of the active session token and cached credentials**. | Information disclosure | High |
| **T-I2** | An **external attacker** who compromises a network intermediary can **observe credential exchange traffic**, leading to **exposure of tokens or credentials**. | Information disclosure | High |
| **T-I3** | An **external attacker** can **enumerate the OIDC discovery endpoint** (`/.well-known/openid-configuration`) to **learn the server's signing keys and supported configuration**, leading to **information useful for targeted attacks**. | Information disclosure | Low |

</div>

**Mitigations:**

- **T-I1**: DPoP binds tokens to the CLI's key pair — stolen tokens cannot be used from a different machine. Credentials are never written to disk (no `~/.aws/credentials`, no `~/.ssh/id_*`). Session lifetime is limited to 8 hours, and AWS STS credentials expire within 1 hour. Full endpoint compromise with kernel access is out of scope (see [assumption A5](#assumptions)).
- **T-I2**: TLS 1.2+ encrypts all traffic in transit. DPoP provides an additional layer — even if a token is somehow intercepted, it cannot be replayed from another client.
- **T-I3**: OIDC discovery is public by design (required for AWS OIDC federation). The exposed information (issuer URL, JWKS, supported algorithms) does not enable impersonation. Private keys are never exposed through these endpoints.

---

### Denial of service

<div class="threat-table">

| ID | Threat | STRIDE | Severity |
|---|---|---|---|
| **T-D1** | An **external attacker** can **flood the Vouch server** with authentication requests, leading to **developers being unable to obtain credentials**. | Denial of service | Medium |
| **T-D2** | An **external attacker** can **disrupt network connectivity** between the CLI and the Vouch server, leading to **inability to establish new sessions or obtain fresh credentials**. | Denial of service | Medium |

</div>

**Mitigations:**

- **T-D1**: The Vouch server implements rate limiting and is deployed behind infrastructure-level DDoS protection. Authentication requires a valid FIDO2 assertion, making automated abuse expensive.
- **T-D2**: Cached credentials remain valid for their remaining lifetime (up to 8 hours for sessions, 1 hour for AWS STS). Developers can continue working with existing credentials during an outage. → [Availability and Failure Modes](/docs/availability/)

---

### Elevation of privilege

<div class="threat-table">

| ID | Threat | STRIDE | Severity |
|---|---|---|---|
| **T-E1** | A **compromised endpoint** can **access the Unix domain socket** and **use the active session to request credentials for any role the user is authorized for**, leading to **unauthorized access to cloud resources** within the user's permission set. | Elevation of privilege | High |
| **T-E2** | A **malicious insider** can **use their valid Vouch session to access resources beyond their intended scope** if IAM roles are overly permissive, leading to **unauthorized access to production systems or sensitive data**. | Elevation of privilege | High |
| **T-E3** | A **compromised server** can **issue sessions for any enrolled user**, leading to **impersonation of any developer** and access to their authorized resources. | Elevation of privilege | Critical |

</div>

**Mitigations:**

- **T-E1**: The Unix socket is restricted to the owning user by filesystem permissions. DPoP prevents extracted tokens from being used on a different machine. Credential scope is limited to the user's authorized roles — the attacker cannot escalate beyond what the user could already access. This threat is bounded by session lifetime (8 hours) and credential lifetime (≤1 hour).
- **T-E2**: IAM roles should follow least-privilege principles. Vouch enables fine-grained role mapping per user via OIDC claims. CloudTrail provides full attribution of which user assumed which role. → [Shared responsibility](/docs/security/#shared-responsibility)
- **T-E3**: Server infrastructure must be hardened with network isolation, encrypted storage, audited access, and minimal attack surface. The server does not store external credentials — it brokers them — so compromise enables credential issuance but not direct access to secrets at rest.

---

## Mitigation summary

| Control | Threats addressed | Layer |
|---|---|---|
| **FIDO2 origin binding** | T-S1 (phishing) | Hardware |
| **FIDO2 user verification (PIN + touch)** | T-S2 (stolen key) | Hardware |
| **DPoP sender-constrained tokens** | T-T1, T-I1, T-I2, T-E1 (token theft, replay) | Protocol |
| **PAR + signed JWTs** | T-T1 (parameter injection) | Protocol |
| **TLS 1.2+** | T-T1, T-I2 (network interception) | Transport |
| **In-memory only credentials** | T-I1 (disk exfiltration) | Application |
| **Short credential lifetimes** | T-S3, T-I1, T-E1 (blast radius) | Application |
| **SCIM de-provisioning** | T-S3 (offboarding) | Identity |
| **OIDC audience restriction** | T-E2 (cross-service abuse) | Protocol |
| **SLSA Level 3 provenance** | T-T3 (supply chain) | Build |
| **Audit logging + CloudTrail** | T-R1, T-R2 (repudiation) | Operational |
| **Rate limiting + DDoS protection** | T-D1 (server flood) | Infrastructure |
| **Credential caching** | T-D2 (outage resilience) | Application |

---

## Out of scope

The following threats are explicitly out of scope for this threat model:

| Threat | Rationale |
|---|---|
| **Kernel-level endpoint compromise** | If an attacker has root/kernel access, all user-space isolation (process memory, socket permissions) is bypassed. Endpoint detection and response (EDR) tools are the appropriate mitigation layer. |
| **Vulnerabilities in external services** | AWS STS, GitHub APIs, container registries, and SSH implementations have their own security models. Vouch trusts their documented behavior. |
| **Cryptographic breaks** | If ECDSA (P-256), Ed25519, or TLS 1.2+ are broken, the impact extends far beyond Vouch. |
| **Physical coercion** | An attacker who can physically compel a developer to authenticate is outside the scope of a technical threat model. |
| **YubiKey hardware vulnerabilities** | Vouch trusts the FIDO2 implementation of enrolled authenticators. Hardware side-channel attacks on the YubiKey secure element are outside scope. |

---

## Revision history

| Date | Change |
|---|---|
| 2026-02-28 | Initial threat model published on vouch.sh, structured using STRIDE and the AWS Threat Composer methodology. |
