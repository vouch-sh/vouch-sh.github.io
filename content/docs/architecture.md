---
title: "Architecture Overview"
linkTitle: "Architecture"
description: "System components, protocols, and trust boundaries — how the Vouch CLI, agent, and server work together."
weight: 19
subtitle: "Understand how Vouch components interact to issue and manage credentials"
params:
  docsGroup: manage
---

This page describes the components that make up Vouch, the protocols they use, and how they interact to turn a YubiKey tap into short-lived credentials for SSH, AWS, GitHub, Docker, and more.

---

## Components

### Vouch CLI (`vouch`)

The command-line interface that developers interact with directly. It handles:

- **Enrollment** -- Registers a FIDO2 key with the Vouch server.
- **Login** -- Performs a FIDO2 assertion and establishes a session.
- **Credential helpers** -- Provides credentials to tools like `aws`, `git`, `ssh`, `docker`, and `cargo` on demand.
- **Setup commands** -- Configures local tool integrations (`vouch setup aws`, `vouch setup codecommit`, etc.).

The CLI communicates with the Vouch agent over a local Unix domain socket and with the Vouch server over HTTPS.

### Vouch Agent

A background process that holds session state in memory. The agent:

- **Caches the active session** so that credential requests do not require repeated FIDO2 assertions.
- **Serves as an SSH agent** (implementing the SSH agent protocol) so that `ssh` can request certificates without additional configuration.
- **Listens on a Unix domain socket** with filesystem permissions restricting access to the owning user.
- **Holds no persistent state** -- if the agent process stops, the session is lost and a new `vouch login` is required.

On macOS, the agent runs as a Homebrew service (`brew services start vouch`). On Linux, it runs as a systemd user service.

### Vouch Server

The server is the identity broker. It:

- **Validates FIDO2 assertions** against enrolled public keys.
- **Issues OIDC ID tokens** (signed with ES256) that AWS and other services consume via standard OIDC federation.
- **Signs SSH certificates** using an Ed25519 certificate authority key.
- **Exchanges tokens** with GitHub Apps, AWS STS, and other external services on behalf of authenticated users.
- **Manages the user directory** via SCIM 2.0 integration with identity providers.
- **Publishes OIDC metadata** at `/.well-known/openid-configuration` and JWKS at `/.well-known/jwks.json` for external services to verify tokens.

The server does not store AWS credentials, SSH private keys, or GitHub tokens. It brokers short-lived credentials from external services.

---

## Protocol details

### FIDO2 / WebAuthn

Used for all user authentication. The FIDO2 exchange happens between the YubiKey (authenticator), the Vouch CLI (client), and the Vouch server (relying party).

- **Registration** (enrollment): The YubiKey generates a key pair. The public key is sent to the server. The private key never leaves the hardware.
- **Authentication** (login): The server sends a challenge. The YubiKey signs it with the private key after PIN + touch verification. The server validates the signature against the stored public key.

### OIDC (OpenID Connect)

The Vouch server acts as an OIDC identity provider. After FIDO2 authentication, it issues a signed JWT (ID token) containing:

| Claim | Description |
|---|---|
| `iss` | Vouch server URL (e.g., `https://us.vouch.sh`) |
| `sub` | User's email address |
| `aud` | Vouch server URL (used as the OIDC client ID) |
| `exp` | Token expiration (short-lived) |
| `iat` | Token issued-at timestamp |
| `hd` | Google Workspace hosted domain |
| `amr` | Authentication methods (e.g., `["hwk", "pin"]`) |
| `acr` | Authentication context class (NIST AAL3) |
| `cnf` | DPoP key thumbprint for sender-constrained tokens |

External services (AWS, custom OIDC applications) validate these tokens using the Vouch server's JWKS endpoint.

### ES256 (ECDSA over P-256)

Used to sign OIDC ID tokens. External services fetch the public key from `/.well-known/jwks.json` to verify token signatures.

### Ed25519

Used for SSH certificate signing. The Vouch server holds an Ed25519 CA key and signs SSH certificates that include the user's identity as a principal.

### SSH Agent Protocol

The Vouch agent implements the [SSH agent protocol](https://datatracker.ietf.org/doc/html/draft-miller-ssh-agent), making certificates available to `ssh` via the `SSH_AUTH_SOCK` environment variable. This is the same protocol used by `ssh-agent` and compatible with all standard SSH clients.

### AWS STS (AssumeRoleWithWebIdentity)

The Vouch CLI calls [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) with the OIDC ID token. AWS validates the token against the Vouch JWKS endpoint and returns temporary credentials (access key ID, secret access key, session token).

### FAPI 2.0

The Vouch CLI operates as a [FAPI 2.0](https://openid.net/specs/fapi-security-profile-2_0-final.html) client. On first use, it generates an ES256 key pair, stores it in the OS keychain, and auto-registers with the server ([RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)). Token requests use DPoP ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)) for sender-constrained tokens, PAR ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)) for protected authorization requests, and `private_key_jwt` ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523)) for client authentication — no shared secrets between CLI and server.

### SCIM 2.0

The Vouch server implements [SCIM 2.0](https://datatracker.ietf.org/doc/html/rfc7644) endpoints for automated user provisioning. Identity providers (Google Workspace, Okta, Azure AD) push user lifecycle events to synchronize the Vouch user directory.

---

## Authentication flow

The complete flow from YubiKey tap to credential consumption:

```
┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌─────────────────┐
│  YubiKey  │    │ Vouch CLI│    │ Vouch Server │    │ External Service│
│  (FIDO2)  │    │ + Agent  │    │  (IdP/CA)    │    │(AWS/GitHub/etc.)│
└─────┬────┘    └─────┬────┘    └──────┬───────┘    └───────┬─────────┘
      │               │               │                    │
      │  1. Challenge  │               │                    │
      │◄──────────────┤  Get challenge │                    │
      │               ├──────────────►│                    │
      │               │               │                    │
      │  2. Sign       │               │                    │
      │  (PIN+touch)  │               │                    │
      ├──────────────►│               │                    │
      │               │  3. Assertion  │                    │
      │               ├──────────────►│                    │
      │               │               │  4. Validate       │
      │               │               │                    │
      │               │  5. Session    │                    │
      │               │◄──────────────┤                    │
      │               │               │                    │
      │               │  6. Credential │                    │
      │               │     request   │                    │
      │               ├──────────────►│                    │
      │               │               │  7. Exchange       │
      │               │               ├──────────────────►│
      │               │               │  8. Short-lived    │
      │               │               │◄──────────────────┤
      │               │  9. Credential │                    │
      │               │◄──────────────┤                    │
      │               │               │                    │
      │               │  10. Tool uses │                    │
      │               │     credential │                    │
      │               ├──────────────────────────────────►│
```

Steps 1--5 happen once during `vouch login`. Steps 6--10 happen on demand each time a tool needs a credential.

---

## Agent architecture

The Vouch agent is a long-running process that provides two services:

### Unix domain socket

The CLI communicates with the agent over a Unix domain socket at a well-known path. The socket file has restrictive permissions (owner-only) to prevent other users on the system from accessing session material.

### In-memory credential cache

The agent caches:

- **Session token** -- Used to authenticate requests to the Vouch server.
- **SSH certificate** -- Served to SSH clients via the agent protocol.
- **Cached STS credentials** -- AWS credentials are cached until their 1-hour expiry to avoid redundant STS calls.

All cached material is held in process memory. Nothing is written to disk. When the agent process stops (logout, reboot, crash), all cached credentials are lost and a new `vouch login` is required.

---

## Network requirements

The Vouch CLI and agent need to reach the following endpoints:

| Destination | Port | Protocol | Purpose |
|---|---|---|---|
| Vouch server (e.g., `us.vouch.sh`) | 443 | HTTPS | Authentication, credential exchange, OIDC |
| AWS STS (`sts.amazonaws.com`) | 443 | HTTPS | `AssumeRoleWithWebIdentity` |
| GitHub API (`api.github.com`) | 443 | HTTPS | Installation token exchange (if GitHub integration is used) |
| Target SSH hosts | 22 | SSH | SSH connections (if SSH integration is used) |

The Vouch server must be reachable from the internet so that AWS can fetch the JWKS endpoint for token validation. If your organization uses a firewall or proxy, ensure these destinations are allowed.

---

## Data residency

Vouch server instances are deployed in specific geographic regions:

| Instance | Region | Status |
|---|---|---|
| `us.vouch.sh` | United States | Active |
| EU instance | Europe | Coming soon |
| APAC instance | Asia-Pacific | Coming soon |

All user data (enrolled keys, user metadata, audit logs) resides in the region of the Vouch server instance you enroll with. Credentials brokered through AWS STS, GitHub, and other external services are subject to those services' own data residency policies.
