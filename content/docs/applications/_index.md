---
title: "Applications (OIDC)"
description: "Add 'Sign in with Vouch' to your web application using OpenID Connect."
weight: 10
subtitle: "Add hardware-backed authentication to your web application"
---

Vouch is a fully compliant [OpenID Connect](https://openid.net/connect/) (OIDC) provider. You can add "Sign in with Vouch" to any web application, single-page app, or native CLI tool using standard OIDC libraries. Every login is backed by a hardware security key, giving your application phishing-resistant authentication without building it yourself.

## What you'll build

By following this guide, you will integrate Vouch as an OIDC identity provider into your application. Users will authenticate with their YubiKey through Vouch, and your application will receive verified identity information including:

- A unique, stable user identifier (`sub` claim)
- The user's email address
- Hardware attestation claims proving the authentication was performed with a verified security key
- Organization membership information

This works with any framework or library that supports OpenID Connect or OAuth 2.0 authorization code flow.

---

## Prerequisites

Before integrating Vouch into your application, you need:

- A **registered OAuth application** in the Vouch admin panel at {{< instance-url >}}/admin
- Your application's **client ID** (`client_id`)
- Your application's **client secret** (`client_secret`)
- A configured **redirect URI** (e.g., `https://your-app.example.com/auth/callback`)

To register an application, ask your Vouch organization administrator to create one in the admin panel, or use the API if you have admin access.

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

## Configuration Reference

Use the following endpoints and values to configure your OIDC client library. All URLs are relative to your Vouch server instance.

| Parameter | Value |
|---|---|
| **Discovery URL** | `{{< instance-url >}}/.well-known/openid-configuration` |
| **Issuer** | `{{< instance-url >}}` |
| **Authorization Endpoint** | `{{< instance-url >}}/oauth/authorize` |
| **Token Endpoint** | `{{< instance-url >}}/oauth/token` |
| **UserInfo Endpoint** | `{{< instance-url >}}/oauth/userinfo` |
| **JWKS URI** | `{{< instance-url >}}/.well-known/jwks.json` |
| **Signing Algorithm** | `ES256` |
| **Supported Scopes** | `openid`, `email` |
| **Device Authorization Endpoint** | `{{< instance-url >}}/oauth/device/code` |
| **Device Verification URL** | `{{< instance-url >}}/oauth/device` |

Most OIDC libraries can auto-configure themselves from the Discovery URL alone.

---

## Vouch-Specific Claims

Vouch ID tokens include additional claims beyond the standard OIDC claims. These provide hardware attestation information that your application can use to enforce security policies.

| Claim | Type | Description |
|---|---|---|
| `hardware_verified` | boolean | `true` if the authentication was performed using a verified hardware security key. Always `true` for Vouch-issued tokens. |
| `hardware_aaguid` | string | The AAGUID (Authenticator Attestation GUID) of the hardware key used for authentication. This identifies the make and model of the security key (e.g., YubiKey 5 series). |
| `cnf` | object | Confirmation claim containing key binding information per RFC 7800. Includes a `kid` field referencing the specific credential used. |

Example ID token payload with Vouch-specific claims:

```json
{
  "iss": "{{< instance-url >}}",
  "sub": "user_abc123",
  "aud": "your-client-id",
  "exp": 1700000000,
  "iat": 1699996400,
  "email": "alice@example.com",
  "email_verified": true,
  "hardware_verified": true,
  "hardware_aaguid": "2fc0579f-8113-47ea-b116-bb5a8db9202a",
  "cnf": {
    "kid": "credential_xyz789"
  }
}
```

Your application can use the `hardware_verified` claim to enforce that only hardware-backed authentications are accepted, and the `hardware_aaguid` claim to restrict access to specific security key models.

### Accessing Vouch claims in your application

Regardless of your framework, the Vouch-specific claims are available in the ID token's payload after standard OIDC verification. Here is a generic example:

```javascript
// After verifying the ID token through your OIDC library:
const hardwareVerified = idToken.hardware_verified; // boolean
const keyModel = idToken.hardware_aaguid;           // string (AAGUID)
const keyBinding = idToken.cnf?.kid;                // string (credential ID)

// Enforce hardware-only authentication
if (!hardwareVerified) {
  throw new Error("Hardware key verification required");
}
```

---

## Token Lifetime and Refresh

Vouch access tokens are short-lived. Your application should handle token expiry gracefully:

- **Server-side applications** -- Check token expiry before using it. If expired, redirect the user through the authorization flow again. Vouch does not issue refresh tokens.
- **Single-page applications** -- Use `automaticSilentRenew` (available in `oidc-client-ts`) to transparently renew tokens before they expire. See the [React]({{< ref "/docs/applications/react" >}}), [Vue]({{< ref "/docs/applications/vue" >}}), and [Vanilla JS]({{< ref "/docs/applications/vanilla-js" >}}) guides for configuration details.
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
curl -X POST {{< instance-url >}}/oauth/token \
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

## Troubleshooting

### Discovery endpoint not reachable

```
Error: unable to fetch OpenID configuration from {{< instance-url >}}/.well-known/openid-configuration
```

- Verify the Vouch server URL is correct and accessible from your application server.
- Check that your application can reach the Vouch server (no firewall or network restrictions).
- Ensure the URL does not have a trailing slash.

### Invalid client_id or client_secret

```
error: invalid_client
```

- Verify the `client_id` and `client_secret` match what was registered in the Vouch admin panel.
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
- Verify the JWKS endpoint is reachable: `{{< instance-url >}}/.well-known/jwks.json`
- Check that the `iss` claim matches your configured issuer URL exactly.
- Ensure your server's clock is synchronized (token validation is time-sensitive).

### CORS errors in browser applications

```
Access to fetch at '{{< instance-url >}}/oauth/token' has been blocked by CORS policy
```

For single-page applications, ensure your application's origin is registered as an allowed origin in the Vouch admin panel. The Vouch server must include your origin in its CORS `Access-Control-Allow-Origin` response header.

### Device code expired

```
error: expired_token
```

The user did not complete authentication within the allowed time window. Request a new device code and display the new `user_code` to the user. The default expiration is 10 minutes.
