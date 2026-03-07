---
title: "Add Hardware-Backed Sign-In to Your Application"
linkTitle: "Applications (OIDC)"
description: "Integrate Vouch as an OIDC provider in your web, SPA, or native app for hardware-verified authentication."
weight: 10
subtitle: "Add hardware-backed authentication to your web application"
params:
  docsGroup: manage
---

Building phishing-resistant authentication from scratch means implementing WebAuthn flows, managing attestation, and handling device lifecycle -- a significant engineering investment for a startup. If you already have an OIDC-compatible application, you can get hardware-backed sign-in without any of that.

Vouch is a fully compliant [OpenID Connect](https://openid.net/connect/) (OIDC) provider. You add "Sign in with Vouch" to any web application, single-page app, or native CLI tool using standard OIDC libraries. Every login is backed by a hardware security key, giving your application phishing-resistant authentication without building it yourself.

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

Most OIDC libraries can auto-configure themselves from the Discovery URL alone.

---

## Vouch-Specific Claims

Vouch ID tokens include additional claims beyond the standard OIDC claims. These provide hardware attestation information that your application can use to enforce security policies.

| Claim | Type | Description |
|---|---|---|
| `hardware_verified` | boolean | `true` if the authentication was performed using a verified hardware security key. Always `true` for Vouch-issued tokens. |
| `hardware_aaguid` | string | The AAGUID (Authenticator Attestation GUID) of the hardware key used for authentication. This identifies the make and model of the security key (e.g., YubiKey 5 series). |
| `cnf` | object | Confirmation claim containing key binding information per RFC 7800. Includes a `kid` field referencing the specific credential used. |
| `amr` | array | Authentication methods used (e.g., `["hwk", "pin"]`). |
| `acr` | string | Authentication context class (e.g., NIST AAL3 for hardware MFA). |

Example ID token payload with Vouch-specific claims:

```json
{
  "iss": "https://{{< instance-url >}}",
  "sub": "user_abc123",
  "aud": "your-client-id",
  "exp": 1700000000,
  "iat": 1699996400,
  "email": "alice@example.com",
  "email_verified": true,
  "hardware_verified": true,
  "hardware_aaguid": "2fc0579f-8113-47ea-b116-bb5a8db9202a",
  "amr": ["hwk", "pin"],
  "acr": "urn:nist:authentication:assurance-level:aal3",
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
