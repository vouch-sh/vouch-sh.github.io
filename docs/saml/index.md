# SAML Identity Providers

> Use SAML 2.0 identity providers with Vouch — Okta, Microsoft Entra ID, Google Workspace, and more.

Source: https://vouch.sh/docs/saml/
Last updated: 2026-04-10

---


Most organizations already have a SAML-based identity provider for single sign-on. Vouch supports **SAML 2.0** as a first-class authentication protocol alongside OIDC, so you can use your existing IdP without changes.

From a developer's perspective, there is no difference -- enrollment and login work the same way regardless of whether your organization uses SAML or OIDC. The browser-based sign-in flow routes through your IdP, and the CLI handles everything else.

---

## Supported identity providers

Vouch has been tested with:

- **Okta**
- **Microsoft Entra ID** (formerly Azure AD)
- **Google Workspace**

Any SAML 2.0 compliant identity provider that supports HTTP-POST or HTTP-Redirect bindings should work.

---

## How it works

1. A developer runs `vouch enroll` or `vouch login`.
2. The browser opens the Vouch server, which detects the configured SAML identity provider.
3. The developer is redirected to the IdP's login page (Okta, Entra ID, etc.).
4. After authentication, the IdP sends a signed SAML assertion back to Vouch's Assertion Consumer Service (ACS) endpoint.
5. Vouch validates the assertion signature, extracts the user's identity, and issues a session.
6. The CLI picks up the session and credential helpers work normally.

The SAML flow uses the same browser-based redirect pattern as OIDC. Developers do not need to know which protocol their organization uses.

---

## Configuring your identity provider

To connect your SAML identity provider to Vouch, you need two pieces of information from your Vouch server:

| Field | Value |
|---|---|
| **SP Metadata URL** | `https://<your-vouch-server>/saml/metadata` |
| **ACS URL** | `https://<your-vouch-server>/saml/acs` |

The SP metadata URL provides a machine-readable XML document containing the entity ID, ACS endpoint, and signing certificate. Most identity providers can import this directly.

### Okta

1. In the Okta Admin Console, go to **Applications > Create App Integration**.
2. Select **SAML 2.0** and click **Next**.
3. Enter an app name (e.g., "Vouch").
4. Set the **Single sign-on URL** to your ACS URL (`https://<your-vouch-server>/saml/acs`).
5. Set the **Audience URI (SP Entity ID)** to your Vouch server URL (`https://<your-vouch-server>`).
6. Under **Attribute Statements**, map `email` to `user.email`.
7. Assign users or groups to the application.

### Microsoft Entra ID

1. In the Azure portal, go to **Entra ID > Enterprise applications > New application**.
2. Select **Create your own application** and choose "Integrate any other application you don't find in the gallery (Non-gallery)".
3. Under **Single sign-on > SAML**, click **Upload metadata file** and point it at `https://<your-vouch-server>/saml/metadata`. Alternatively, set the fields manually:
   - **Identifier (Entity ID):** `https://<your-vouch-server>`
   - **Reply URL (ACS URL):** `https://<your-vouch-server>/saml/acs`
4. Under **Attributes & Claims**, ensure `emailaddress` maps to `user.mail`.
5. Assign users or groups to the application.

### Google Workspace

1. In the Google Admin console, go to **Apps > Web and mobile apps > Add app > Add custom SAML app**.
2. Enter an app name (e.g., "Vouch").
3. On the **Service provider details** page:
   - **ACS URL:** `https://<your-vouch-server>/saml/acs`
   - **Entity ID:** `https://<your-vouch-server>`
4. Add an attribute mapping: **Primary email** mapped to `email`.
5. Turn on the app for the relevant organizational units.

---

## Developer experience

From the developer's perspective, SAML and OIDC work identically:

```bash
# Enrollment (one-time)
vouch enroll --server https://<your-vouch-server>

# Daily login
vouch login
```

The browser opens, the developer signs in through the IdP, and the CLI receives a session. All credential helpers (SSH, AWS, GitHub, etc.) work the same way regardless of the underlying authentication protocol.

---

## FAQ

### Can I use both SAML and OIDC?

Each Vouch organization is configured with a single identity provider. If your IdP supports both protocols, choose whichever your organization standardizes on.

### Do developers need to know which protocol is configured?

No. The CLI and browser-based flow are identical for both SAML and OIDC. Developers do not need to take any different actions.

### Which SAML bindings are supported?

Vouch supports **HTTP-POST** and **HTTP-Redirect** bindings for authentication requests.
