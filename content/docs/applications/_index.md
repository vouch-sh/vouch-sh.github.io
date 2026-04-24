---
title: "Add Hardware-Backed Sign-In to Your Application"
linkTitle: "Applications (OIDC)"
description: "Integrate Vouch as an OIDC provider in your web, SPA, or native app for hardware-verified authentication."
weight: 1
subtitle: "Add hardware-backed authentication to your web application"
params:
  docsGroup: admin
---

Building phishing-resistant authentication from scratch means implementing WebAuthn flows, managing attestation, and handling device lifecycle -- a significant engineering investment for a startup. If you already have an OIDC-compatible application, you can get hardware-backed sign-in without any of that.

Vouch is a fully compliant [OpenID Connect](https://openid.net/connect/) (OIDC) provider. You add "Sign in with Vouch" to any web application, single-page app, or native CLI tool using standard OIDC libraries. Every login is backed by a hardware security key, giving your application phishing-resistant authentication without building it yourself.

<a href="https://openid.net/certification/certified-fapi-2-0-op-security-profile-final-message-signing-final/" target="_blank" rel="noopener noreferrer"><img src="/img/openid-certified.png" alt="OpenID Certified" class="certification-badge-inline" /></a>

## Choose a guide

<div class="journey-grid">
  <div class="journey-card">
    <h3>Server-rendered web apps</h3>
    <p>Use the authorization code flow with a backend session.</p>
    <p><a href="#server-side-frameworks">Server frameworks</a></p>
  </div>
  <div class="journey-card">
    <h3>Single-page apps</h3>
    <p>Use authorization code with PKCE for browser-based clients.</p>
    <p><a href="#single-page-applications">SPA frameworks</a></p>
  </div>
  <div class="journey-card">
    <h3>Native and CLI apps</h3>
    <p>Use the Device Authorization Grant for terminals and desktop apps.</p>
    <p><a href="#native--cli-applications">Native and CLI guides</a></p>
  </div>
  <div class="journey-card">
    <h3>Agents and MCP servers</h3>
    <p>Protect tool servers and agent-to-agent calls with Vouch OIDC tokens.</p>
    <p><a href="#mcp-servers">MCP guides</a> · <a href="#agent-to-agent-a2a">A2A guides</a></p>
  </div>
</div>

## What you'll build

By following this guide, you will integrate Vouch as an OIDC identity provider into your application. Users will authenticate with their YubiKey through Vouch, and your application will receive verified identity information including:

- A unique, stable user identifier (`sub` claim) in the ID token
- The user's email address in the ID token
- Hardware attestation claim (`hardware_verified`) in the access token

This works with any framework or library that supports OpenID Connect or OAuth 2.0 authorization code flow. Beyond web applications, Vouch secures **MCP tool servers** with bearer token authentication and enables **agent-to-agent (A2A)** communication backed by hardware-verified identity.

---

## Prerequisites

Before integrating Vouch into your application, you need:

- A **registered OAuth application** at https://{{< instance-url >}}/applications
- Your application's **client ID** (`client_id`)
- Your application's **client secret** (`client_secret`)
- A configured **redirect URI** (e.g., `https://your-app.example.com/auth/callback`)

To register an application, navigate to https://{{< instance-url >}}/applications and click **New Application**:

![Register New Application form with application type, redirect URIs, and security profile options](/images/admin/applications-new.png)

After creating the application, you will see your client ID and client secret. The client secret is only shown once — store it securely.

![Application Created page showing client ID and client secret](/images/admin/application-created.png)

You can view and manage all your applications from the applications list:

![My Applications page showing registered applications with type, scope, and status](/images/admin/applications-list.png)

---

## Managing Applications

After creating an application, you can view its full configuration — including client ID, type, access scope, redirect URIs, and client secrets — from the application detail page:

![Application detail page showing configuration, redirect URIs, and client secret management](/images/admin/application-detail.png)

From this page you can edit settings, rotate client secrets, or delete the application.

---

## Access Scopes

Vouch supports two access scopes that control what identity information is included in tokens:

| Scope | Description | Claims Included |
|---|---|---|
| `openid` | **Required.** Identifies this as an OIDC authentication request. | `sub`, `iss`, `aud`, `exp`, `iat` |
| `email` | User email address. | `email`, `email_verified` |

Request scopes in the authorization request by including them in the `scope` parameter, space-separated:

```
scope=openid email
```

---

## Rich Authorization Requests

For fine-grained access control beyond what scopes can express, Vouch supports Rich Authorization Requests ([RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396)). Instead of flat scope strings, your application can pass structured `authorization_details` objects that describe the type, actions, and resources being requested.

### Format

The `authorization_details` parameter is a JSON array of objects. Each object must include a `type` field; all other fields are defined by your application:

```json
[
  {
    "type": "account_access",
    "actions": ["read", "transfer"],
    "locations": ["https://api.example.com/accounts"]
  }
]
```

### Using with PAR

Since Vouch uses Pushed Authorization Requests ([RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)), include `authorization_details` in the PAR request body alongside your other parameters:

```bash
curl -X POST https://{{< instance-url >}}/oauth/par \
  -d "client_id=your-client-id" \
  -d "client_secret=your-client-secret" \
  -d "response_type=code" \
  -d "redirect_uri=https://your-app.example.com/auth/callback" \
  -d "scope=openid email" \
  -d 'authorization_details=[{"type":"account_access","actions":["read","transfer"]}]'
```

### Token response

The granted `authorization_details` are returned in the token response, so your application can confirm exactly what was authorized:

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "authorization_details": [
    {
      "type": "account_access",
      "actions": ["read", "transfer"]
    }
  ]
}
```

Rich Authorization Requests can be used alongside scopes — they are complementary, not mutually exclusive.

---

## Configuration Reference

Use the following endpoints and values to configure your OIDC client library. All URLs are relative to your Vouch server instance.

| Parameter | Value |
|---|---|
| **Discovery URL** | `https://{{< instance-url >}}/.well-known/openid-configuration` |
| **Issuer** | `https://{{< instance-url >}}` |
| **Authorization Endpoint** | `https://{{< instance-url >}}/oauth/authorize` |
| **Token Endpoint** | `https://{{< instance-url >}}/oauth/token` |
| **UserInfo Endpoint** | `https://{{< instance-url >}}/oauth/userinfo` |
| **JWKS URI** | `https://{{< instance-url >}}/.well-known/jwks.json` |
| **Signing Algorithm** | `ES256` |
| **Supported Scopes** | `openid`, `email` |
| **Authorization Details** | Supported via `authorization_details` parameter ([RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396)) |
| **Device Authorization Endpoint** | `https://{{< instance-url >}}/oauth/device/code` |
| **Device Verification URL** | `https://{{< instance-url >}}/oauth/device` |
| **PAR Endpoint** | `https://{{< instance-url >}}/oauth/par` |
| **Token Revocation Endpoint** | `https://{{< instance-url >}}/oauth/revoke` |
| **Token Introspection Endpoint** | `https://{{< instance-url >}}/oauth/introspect` |
| **Protected Resource Metadata** | `https://{{< instance-url >}}/.well-known/oauth-protected-resource` |

Most OIDC libraries can auto-configure themselves from the Discovery URL alone.

### Additional capabilities

The authorization endpoint supports several advanced features:

- **`response_mode=form_post`** — The authorization code is delivered via an auto-submitting HTML form POST instead of a query-string redirect. Useful for server-side applications that want the code in the request body.
- **`request_uri` by URL** — In addition to PAR (`urn:ietf:params:oauth:request_uri:...`) and inline `request` JWTs, the authorization endpoint accepts HTTPS URLs as `request_uri` values for hosting Request Objects externally (OIDC Core Section 6.2).
- **Signed UserInfo** — Applications that register `userinfo_signed_response_alg` (ES256 or RS256) during client registration receive signed JWT responses from the UserInfo endpoint instead of plain JSON.
- **Client branding** — Applications that register `logo_uri`, `policy_uri`, or `tos_uri` during client registration have these displayed on the Vouch login page.

---

## ID Token Claims

Vouch ID tokens follow the standard OIDC specification. The payload contains:

| Claim | Type | Description |
|---|---|---|
| `iss` | string | Issuer — your Vouch server URL |
| `sub` | string | Subject — stable, unique user identifier |
| `aud` | string | Audience — your application's `client_id` |
| `exp` | number | Expiration time |
| `iat` | number | Issued-at time |
| `email` | string | User's email address (when `email` scope is requested) |
| `email_verified` | boolean | Whether the email is verified (when `email` scope is requested) |
| `amr` | array | Authentication methods used (e.g., `["hwk", "pin"]`) |
| `acr` | string | Authentication context class (e.g., NIST AAL3 for hardware MFA) |
| `cnf` | object | Confirmation claim containing key binding information per [RFC 7800](https://datatracker.ietf.org/doc/html/rfc7800). Includes a `kid` field referencing the specific credential used. |

Example ID token payload:

```json
{
  "iss": "https://{{< instance-url >}}",
  "sub": "user_abc123",
  "aud": "your-client-id",
  "exp": 1700000000,
  "iat": 1699996400,
  "email": "alice@example.com",
  "email_verified": true,
  "amr": ["hwk", "pin"],
  "acr": "urn:nist:authentication:assurance-level:aal3",
  "cnf": {
    "kid": "credential_xyz789"
  }
}
```

## Access Token Claims

Vouch issues access tokens as JWTs ([RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068)). These include a hardware attestation claim that your application can use to enforce security policies. This claim is **not** in the ID token or the UserInfo response.

| Claim | Type | Description |
|---|---|---|
| `hardware_verified` | boolean | `true` if the authentication was performed using a verified hardware security key. Always `true` for Vouch-issued tokens from the authorization code flow. |

To access this claim, decode the access token JWT payload. See the [examples repository](https://github.com/vouch-sh/examples) for working implementations in each framework.

The `hardware_aaguid` claim (FIDO2 authenticator AAGUID identifying the key make and model) is available in cloud federation tokens (AWS STS, Kubernetes) but not in standard OIDC access tokens.

---

## Token Lifetime

Vouch access tokens are short-lived and Vouch does not issue refresh tokens (by design -- every session requires hardware key interaction). Your application should handle token expiry gracefully:

- **Server-side applications** -- Check token expiry before using it. If expired, redirect the user through the authorization flow again.
- **Single-page applications** -- Check `user.expired` before making API calls. If the token has expired, redirect the user through the authorization flow again using `signinRedirect()`.
- **Native CLI applications** -- Use the device authorization flow (RFC 8628) and prompt the user to re-authenticate when the token expires.

---

## Service-to-Service (M2M) Authentication

For server-to-server communication where no human user is involved, Vouch supports **Token Exchange** (RFC 8693). A service with a valid Vouch token can exchange it for a new token scoped to a different audience, enabling secure service-to-service calls.

### Token Exchange Flow

1. **Service A** authenticates a user through the standard OIDC flow and obtains an access token.
2. **Service A** calls **Service B** and needs to pass along the user's identity.
3. **Service A** exchanges its token for a new token with **Service B's** audience by calling the token endpoint.

### Token Exchange Request

```bash
curl -X POST https://{{< instance-url >}}/oauth/token \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  -d "subject_token=<access-token>" \
  -d "subject_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "audience=service-b-client-id" \
  -d "client_id=service-a-client-id" \
  -d "client_secret=service-a-client-secret"
```

### Token Exchange Response

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIs...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

The new token is scoped to Service B's audience and inherits the user's identity claims from the original token. Service B can validate this token by checking the `aud` claim and verifying the signature against the Vouch JWKS endpoint.

---

## Client Credentials (Machine-to-Machine)

For automated systems that need to authenticate without any human interaction -- CI/CD pipelines, background services, or daemon processes -- Vouch supports the OAuth **client credentials** grant ([RFC 6749 Section 4.4](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4)).

Unlike the authorization code flow (which requires a browser and a YubiKey tap), the client credentials flow lets a service authenticate directly using its client ID and client secret. Services can also use Mutual TLS ([RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705)) for client authentication (`tls_client_auth` or `self_signed_tls_client_auth`) and certificate-bound access tokens as an alternative to client secrets — see [Architecture](/docs/architecture/#mutual-tls-rfc-8705) for details.

### When to use client credentials

- **CI/CD pipelines** that need Vouch tokens without a human in the loop
- **Background services** or **daemon processes** that run unattended
- **Automated tooling** that needs to call Vouch-protected APIs

For deployments where you want a human to explicitly authorize each action (e.g., production deploys), use the [CI/CD human approval gate](/docs/cicd/) pattern instead.

### Request

```bash
curl -X POST https://{{< instance-url >}}/oauth/token \
  -d "grant_type=client_credentials" \
  -d "client_id=your-client-id" \
  -d "client_secret=your-client-secret" \
  -d "scope=openid email"
```

### Response

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

The access token can be used as a Bearer token in API requests to any service that validates tokens against the Vouch JWKS endpoint.

### Setup

1. Register an application at `https://{{< instance-url >}}/applications` (or via the [Admin Dashboard](/docs/admin/)).
2. Note the **client ID** and **client secret**.
3. Store the client secret securely (e.g., as a CI/CD secret, not in source code).
4. Use the token endpoint with `grant_type=client_credentials` to obtain tokens.

---

## Troubleshooting

### Discovery endpoint not reachable

```
Error: unable to fetch OpenID configuration from https://{{< instance-url >}}/.well-known/openid-configuration
```

- Verify the Vouch server URL is correct and accessible from your application server.
- Check that your application can reach the Vouch server (no firewall or network restrictions).
- Ensure the URL does not have a trailing slash.

### Invalid client_id or client_secret

```
error: invalid_client
```

- Verify the `client_id` and `client_secret` match what was registered on the Vouch [applications page](https://{{< instance-url >}}/applications).
- Check that the client credentials are not expired or revoked.
- Ensure the credentials are being sent correctly (as form parameters for the token endpoint, not as JSON).

### Redirect URI mismatch

```
error: redirect_uri_mismatch
```

The redirect URI in your authorization request does not match any URI registered for this client. Ensure:

- The redirect URI exactly matches what was registered (including protocol, host, port, and path).
- There are no trailing slashes or query parameters that differ.
- If using localhost for development, the port must match exactly.

### ID token validation fails

```
Error: ID token signature verification failed
```

- Ensure your library is configured to use the `ES256` signing algorithm. Vouch uses ECDSA with P-256, not RSA.
- Verify the JWKS endpoint is reachable: `https://{{< instance-url >}}/.well-known/jwks.json`
- Check that the `iss` claim matches your configured issuer URL exactly.
- Ensure your server's clock is synchronized (token validation is time-sensitive).

### CORS errors in browser applications

```
Access to fetch at 'https://{{< instance-url >}}/oauth/token' has been blocked by CORS policy
```

For single-page applications, ensure your application's origin is registered as an allowed origin on the Vouch [applications page](https://{{< instance-url >}}/applications). The Vouch server must include your origin in its CORS `Access-Control-Allow-Origin` response header.

### Device code expired

```
error: expired_token
```

The user did not complete authentication within the allowed time window. Request a new device code and display the new `user_code` to the user. The default expiration is 10 minutes.
