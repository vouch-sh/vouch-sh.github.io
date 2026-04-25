# Architecture Overview

> System components, protocols, and trust boundaries — how the Vouch CLI, agent, and server work together.

Source: https://vouch.sh/docs/architecture/
Last updated: 2026-04-21

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

The CLI communicates with the Vouch agent over a local Unix domain socket and with the Vouch server over HTTPS. All requests to the server are authenticated using [HTTP Message Signatures (RFC 9421)](https://datatracker.ietf.org/doc/html/rfc9421) for cryptographic proof of request authenticity.

### Vouch Agent

A background process that holds session state in memory. The agent:

- **Caches the active session** so that credential requests do not require repeated FIDO2 assertions.
- **Serves as an SSH agent** (implementing the SSH agent protocol) so that `ssh` can request certificates without additional configuration.
- **Listens on a Unix domain socket** with filesystem permissions restricting access to the owning user.
- **Holds no persistent state** -- if the agent process stops, the session is lost and a new `vouch login` is required.

On macOS, the agent runs as a Homebrew service (`brew services start vouch`). On Linux, it runs as a systemd user service.

### Vouch Server

The server is the identity broker. It:

- **Authenticates users** via FIDO2/WebAuthn assertions, with identity federation through your organization's OIDC or [SAML 2.0](/docs/saml/) identity provider.
- **Issues OIDC ID tokens** (signed with ES256 via AWS KMS) that AWS and other services consume via standard OIDC federation.
- **Signs SSH certificates** using an Ed25519 certificate authority key managed by AWS KMS.
- **Exchanges tokens** with GitHub Apps, AWS STS, and other external services on behalf of authenticated users.
- **Manages the user directory** via SCIM 2.0 integration with identity providers.
- **Publishes OIDC metadata** at `/.well-known/openid-configuration`, JWKS at `/.well-known/jwks.json`, and Protected Resource Metadata at `/.well-known/oauth-protected-resource` ([RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)). When mTLS is configured, the discovery document includes `mtls_endpoint_aliases` for mTLS-capable clients.
- **Provides OIDC discovery** for automatic identity provider detection during enrollment.

The server does not store AWS credentials, SSH private keys, or GitHub tokens. It brokers short-lived credentials from external services.

---

## Protocol details

### FIDO2 / WebAuthn

Used for all user authentication. The FIDO2 exchange happens between the YubiKey (authenticator), the Vouch CLI (client), and the Vouch server (relying party).

- **Registration** (enrollment): The YubiKey generates a key pair. The public key is sent to the server along with an attestation certificate. The private key never leaves the hardware. When attestation verification is enabled, the server validates the certificate chain against pinned Yubico root CA certificates to confirm the key is a genuine hardware device.
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
| `cnf` | Confirmation claim for sender-constrained tokens — contains `jkt` (DPoP key thumbprint) or `x5t#S256` (mTLS certificate thumbprint) |

External services (AWS, custom OIDC applications) validate these tokens using the Vouch server's JWKS endpoint. For [Kubernetes OIDC authentication](/docs/kubernetes/), the server issues tokens with a configurable `aud` claim (default: `kubernetes`) that matches the API server's `--oidc-client-id` flag.

### ES256 (ECDSA over P-256)

Used to sign OIDC ID tokens and access tokens. The signing key is managed by AWS KMS — the private key never exists outside the KMS boundary. External services fetch the public key from `/.well-known/jwks.json` to verify token signatures. The JWKS endpoint supports key rotation — consuming services should re-fetch the JWKS when they encounter a token signed with an unknown `kid`.

### Ed25519

Used for SSH certificate signing. The Ed25519 CA key is managed by AWS KMS. The Vouch server delegates each signing operation to KMS and returns the signed certificate. The CA private key never leaves KMS.

### SSH Agent Protocol

The Vouch agent implements the [SSH agent protocol](https://datatracker.ietf.org/doc/html/draft-miller-ssh-agent), making certificates available to `ssh` via the `SSH_AUTH_SOCK` environment variable. This is the same protocol used by `ssh-agent` and compatible with all standard SSH clients.

### AWS STS (AssumeRoleWithWebIdentity)

The Vouch CLI calls [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) with the OIDC ID token. AWS validates the token against the Vouch JWKS endpoint and returns temporary credentials (access key ID, secret access key, session token).

### FAPI 2.0

The Vouch CLI operates as a [FAPI 2.0](https://openid.net/specs/fapi-security-profile-2_0-final.html) client. On first use, it generates an ES256 key pair, stores it in the OS keychain, and auto-registers with the server ([RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)). Token requests use DPoP ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)) for sender-constrained tokens, PAR ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)) for protected authorization requests, RAR ([RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396)) for structured authorization details, and `private_key_jwt` ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523)) for client authentication — no shared secrets between CLI and server. FAPI 2.0 also accepts Mutual TLS ([RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705)) as an alternative sender-constraining mechanism — see [Mutual TLS](#mutual-tls-rfc-8705) below.

### HTTP Message Signatures (RFC 9421)

All authenticated requests from the CLI to the Vouch server include [HTTP Message Signatures](https://datatracker.ietf.org/doc/html/rfc9421). The CLI signs each request using the FAPI key pair stored in the OS keychain. The server verifies the signature before processing the request, providing cryptographic proof that the request was not tampered with in transit and originated from the registered client.

Supported algorithms: ECDSA P-256/P-384, EdDSA, and RSA-PSS-SHA512.

### Mutual TLS (RFC 8705)

The Vouch server supports [OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens](https://datatracker.ietf.org/doc/html/rfc8705) as an alternative to DPoP for sender-constrained tokens. When configured, a separate mTLS listener runs on port 8443 and verifies client certificates during the TLS handshake.

Two client authentication methods are supported: `tls_client_auth` (PKI-validated certificates where the server verifies the certificate chain against a configured Client Certificate CA) and `self_signed_tls_client_auth` (self-signed certificates registered via the client's `jwks` or `jwks_uri` with `x5c` certificate representations).

Certificate-bound access tokens include an `x5t#S256` thumbprint (SHA-256 hash of the client's DER-encoded X.509 certificate) in the `cnf` claim. Resource servers validate that the certificate presented at the TLS layer matches the thumbprint bound to the token. The Client Certificate CA can be managed locally or via AWS KMS, following the same pattern as the SSH CA.

The OpenID Configuration discovery document advertises mTLS support via `mtls_endpoint_aliases`, which provides alternative endpoint URLs for token, revocation, and introspection endpoints on the mTLS port.

### Protected Resource Metadata (RFC 9728)

The Vouch server publishes [OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728) at `/.well-known/oauth-protected-resource`. This document describes the resource's authorization policy: which authorization server to use, the JWKS URI, supported scopes, bearer token presentation methods, DPoP and mTLS binding requirements, and descriptive URLs.

Every response includes a `signed_metadata` field — an ES256 JWS with `typ=oauth-protected-resource+jwt`, verifiable via the advertised `jwks_uri`. When a protected endpoint returns a 401, the `WWW-Authenticate` header includes a `resource_metadata` parameter pointing clients to the metadata document for automatic authorization server discovery.

Descriptive metadata fields are configurable via environment variables: `VOUCH_RESOURCE_NAME`, `VOUCH_RESOURCE_DOCUMENTATION`, `VOUCH_RESOURCE_POLICY_URI`, and `VOUCH_RESOURCE_TOS_URI`.

### Step-Up Authentication (RFC 9470)

The Vouch server supports the [OAuth 2.0 Step-Up Authentication Challenge Protocol](https://datatracker.ietf.org/doc/html/rfc9470). When a protected resource requires a higher authentication assurance level than the current token provides, it returns a `WWW-Authenticate` challenge with `error="insufficient_user_authentication"` and `acr_values` or `max_age` parameters specifying the required authentication strength or recency. Clients use these parameters in a new authorization request to obtain a token meeting the elevated requirements. Vouch's FIDO2 hardware authentication satisfies NIST AAL3 (`acr` claim), which meets most step-up requirements.

### SAML 2.0

For organizations using SAML-based identity providers, the Vouch server acts as a SAML Service Provider. It publishes SP metadata at `/saml/metadata` and accepts assertions at the Assertion Consumer Service endpoint (`/saml/acs`). Both HTTP-POST and HTTP-Redirect bindings are supported. See [SAML Identity Providers](/docs/saml/) for configuration details.

### SCIM 2.0

The Vouch server implements [SCIM 2.0](https://datatracker.ietf.org/doc/html/rfc7644) endpoints for automated user provisioning. Identity providers (Google Workspace, Okta, Azure AD) push user lifecycle events to synchronize the Vouch user directory.

### Standards compliance

The Vouch server implements the following standards, each with dedicated test coverage:

| Standard | Title | Usage in Vouch |
|---|---|---|
| [OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html) | OpenID Connect Core 1.0 | Identity provider, ID tokens, UserInfo |
| [FAPI 2.0 SP](https://openid.net/specs/fapi-security-profile-2_0-final.html) | FAPI 2.0 Security Profile | Security controls for CLI and application clients |
| [FAPI 2.0 MS](https://openid.net/specs/fapi-message-signing-2_0-final.html) | FAPI 2.0 Message Signing | HTTP Message Signatures on requests and responses |
| [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) | OAuth 2.0 Authorization Framework | Core authorization flows |
| [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009) | Token Revocation | `/oauth/revoke` endpoint |
| [RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523) | JWT Bearer Client Authentication | `private_key_jwt` client authentication |
| [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) | Dynamic Client Registration | CLI auto-registration |
| [RFC 7592](https://datatracker.ietf.org/doc/html/rfc7592) | Dynamic Client Registration Management | Client registration updates |
| [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) | PKCE | Code challenge for public clients |
| [RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644) | SCIM 2.0 | User provisioning |
| [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662) | Token Introspection | `/oauth/introspect` endpoint |
| [RFC 7800](https://datatracker.ietf.org/doc/html/rfc7800) | Proof-of-Possession Key Semantics | `cnf` claim in tokens |
| [RFC 8176](https://datatracker.ietf.org/doc/html/rfc8176) | Authentication Method Reference Values | `amr` claim values |
| [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) | Authorization Server Metadata | `/.well-known/openid-configuration` |
| [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) | Device Authorization Grant | CLI and native app authentication |
| [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) | Token Exchange | Service-to-service delegation |
| [RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705) | Mutual-TLS | mTLS client auth, certificate-bound tokens |
| [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) | Resource Indicators | Audience-restricted tokens |
| [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725) | JWT Best Current Practices | JWT security hardening |
| [RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068) | JWT Profile for Access Tokens | Access token format |
| [RFC 9101](https://datatracker.ietf.org/doc/html/rfc9101) | JWT-Secured Authorization Request | Signed authorization requests |
| [RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126) | Pushed Authorization Requests | Back-channel authorization |
| [RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207) | AS Issuer Identification | Mix-up attack prevention via `iss` parameter |
| [RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396) | Rich Authorization Requests | Structured authorization details |
| [RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421) | HTTP Message Signatures | Request-level integrity |
| [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) | DPoP | Sender-constrained tokens |
| [RFC 9470](https://datatracker.ietf.org/doc/html/rfc9470) | Step-Up Authentication | Authentication challenge protocol |
| [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) | Protected Resource Metadata | `/.well-known/oauth-protected-resource` |
| [SAML 2.0](http://docs.oasis-open.org/security/saml/v2.0/) | SAML 2.0 | Identity provider federation |

---

## Authentication flow

The complete flow from YubiKey tap to credential consumption:

```
┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌─────────────────┐
│  YubiKey │    │ Vouch CLI│    │ Vouch Server │    │ External Service│
│  (FIDO2) │    │ + Agent  │    │  (IdP/CA)    │    │(AWS/GitHub/etc.)│
└─────┬────┘    └─────┬────┘    └──────┬───────┘    └───────┬─────────┘
      │               │                │                    │
      │  1. Challenge │                │                    │
      │◄──────────────┤  Get challenge │                    │
      │               ├──────-────────►│                    │
      │               │                │                    │
      │  2. Sign      │                │                    │
      │  (PIN+touch)  │                │                    │
      ├──────────────►│                │                    │
      │               │  3. Assertion  │                    │
      │               ├─────-─────────►│                    │
      │               │                │  4. Validate       │
      │               │                │                    │
      │               │  5. Session    │                    │
      │               │◄───────-───────┤                    │
      │               │                │                    │
      │               │  6. Credential │                    │
      │               │     request    │                    │
      │               ├─────────────-─►│                    │
      │               │                │  7. Exchange       │
      │               │                ├──────────────────-►│
      │               │                │  8. Short-lived    │
      │               │                │◄──────────────────-┤
      │               │  9. Credential │                    │
      │               │◄────────────-──┤                    │
      │               │                │                    │
      │               │  10. Tool uses │                    │
      │               │     credential │                    │
      │               ├─────────────────────────────--─────►│
```

Steps 1--5 happen once during `vouch login`. Steps 6--10 happen on demand each time a tool needs a credential.

---

## Agent architecture

The Vouch agent is a long-running process that provides two services:

### Unix domain socket

The CLI communicates with the agent over a Unix domain socket at a well-known path. The socket is protected by multiple layers:

- **Filesystem permissions** — The socket file has restrictive permissions (owner-only) to prevent other users on the system from accessing session material.
- **Peer credential verification** — Every incoming connection is checked using OS-level peer credentials (`SO_PEERCRED` on Linux, `getpeereid` on macOS) to verify the connecting process has the same UID as the agent. Connections from a different UID are rejected and audit-logged, following the same approach used by `gpg-agent`.
- **Directory safety** — On startup, the agent validates that `~/.vouch/` is not a symlink and is owned by the current user, preventing symlink-based directory hijacking where an attacker pre-creates the directory pointing to an attacker-controlled location.

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
| Vouch server mTLS endpoint | 8443 | mTLS (HTTPS) | Certificate-bound token requests, mTLS client authentication (when configured) |
| AWS STS (`sts.amazonaws.com`) | 443 | HTTPS | `AssumeRoleWithWebIdentity` |
| Target SSH hosts | 22 | SSH | SSH connections (if SSH integration is used) |

The Vouch server additionally requires outbound access to GitHub (`api.github.com`, port 443, HTTPS) for installation token exchange when the GitHub integration is enabled.

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
