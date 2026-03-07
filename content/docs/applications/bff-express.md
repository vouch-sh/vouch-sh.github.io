---
title: "BFF Pattern (Express)"
description: "Secure a single-page application with hardware-backed authentication using the Backend-for-Frontend pattern with Express and openid-client."
weight: 25
params:
  category: "spa"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

The Backend-for-Frontend (BFF) pattern keeps all OAuth tokens on the server. The browser never sees access tokens or ID tokens -- it authenticates via `HttpOnly`, `SameSite=Strict` session cookies instead. This eliminates token theft from XSS and removes the need for secure token storage in the browser.

> **PKCE is required.** The server generates a code verifier and challenge for every authorization request, even though a client secret is also used.

[openid-client](https://github.com/panva/openid-client) (v6) provides a certified OpenID Connect client for Node.js.

Install the dependencies:

```bash
npm install express express-session openid-client
```

### Server

Create the Express BFF server (`app.js`):

```javascript
import express from "express";
import session from "express-session";
import * as client from "openid-client";

const issuer = process.env.VOUCH_ISSUER || "https://{{< instance-url >}}";
const clientId = process.env.VOUCH_CLIENT_ID;
const clientSecret = process.env.VOUCH_CLIENT_SECRET;
const callbackUrl =
  process.env.VOUCH_REDIRECT_URI || "http://localhost:3000/auth/callback";

const config = await client.discovery(
  new URL(issuer),
  clientId,
  clientSecret,
);

const app = express();

app.use(
  session({
    secret: process.env.SECRET_KEY || "dev-secret-change-in-production",
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,
      sameSite: "strict",
      secure: process.env.NODE_ENV === "production",
    },
  }),
);

app.use(express.static("public"));
```

#### Login route

Generates a PKCE code verifier, builds the authorization URL, and redirects the user to Vouch:

```javascript
app.get("/auth/login", async (req, res) => {
  const codeVerifier = client.randomPKCECodeVerifier();
  const codeChallenge =
    await client.calculatePKCECodeChallenge(codeVerifier);
  const state = client.randomState();
  const nonce = client.randomNonce();

  req.session.oidc = { codeVerifier, state, nonce };

  const redirectTo = client.buildAuthorizationUrl(config, {
    redirect_uri: callbackUrl,
    scope: "openid email",
    code_challenge: codeChallenge,
    code_challenge_method: "S256",
    state,
    nonce,
  });

  res.redirect(redirectTo.href);
});
```

#### Callback route

Exchanges the authorization code for tokens and stores the claims in the session:

```javascript
app.get("/auth/callback", async (req, res) => {
  const { codeVerifier, state, nonce } = req.session.oidc || {};
  delete req.session.oidc;

  const currentUrl = new URL(req.url, `http://${req.headers.host}`);
  const tokens = await client.authorizationCodeGrant(
    config,
    currentUrl,
    {
      pkceCodeVerifier: codeVerifier,
      expectedState: state,
      expectedNonce: nonce,
    },
  );

  const claims = tokens.claims();
  req.session.user = {
    id: claims.sub,
    email: claims.email,
    hardwareVerified: claims.hardware_verified || false,
  };
  req.session.tokens = {
    accessToken: tokens.access_token,
    expiresAt: tokens.expires_in
      ? Date.now() + tokens.expires_in * 1000
      : null,
  };

  res.redirect("/");
});
```

#### API endpoints

Expose the session data and proxy the UserInfo endpoint so the frontend never handles tokens directly:

```javascript
app.get("/api/me", (req, res) => {
  if (!req.session.user) {
    return res.json({ authenticated: false });
  }
  res.json({ authenticated: true, user: req.session.user });
});

app.get("/api/userinfo", async (req, res) => {
  if (!req.session.tokens?.accessToken) {
    return res.status(401).json({ error: "Not authenticated" });
  }
  const response = await fetch(`${issuer}/oauth/userinfo`, {
    headers: {
      Authorization: `Bearer ${req.session.tokens.accessToken}`,
    },
  });
  if (!response.ok) {
    return res
      .status(response.status)
      .json({ error: `UserInfo request failed: ${response.status}` });
  }
  res.json(await response.json());
});

app.get("/auth/logout", (req, res) => {
  req.session.destroy(() => res.redirect("/"));
});

app.listen(3000);
```

### Frontend

The static frontend calls the BFF API endpoints -- it never touches tokens (`public/index.html`):

```html
<div id="app">Loading...</div>

<script>
  const app = document.getElementById("app");

  async function checkAuth() {
    const res = await fetch("/api/me");
    const data = await res.json();
    app.textContent = "";

    if (data.authenticated) {
      const p = document.createElement("p");
      p.textContent = "Signed in as " + data.user.email;
      app.appendChild(p);

      if (data.user.hardwareVerified) {
        const hw = document.createElement("p");
        hw.innerHTML = "<strong>Hardware Verified</strong>";
        app.appendChild(hw);
      }

      const logoutLink = document.createElement("a");
      logoutLink.href = "/auth/logout";
      logoutLink.textContent = "Sign out";
      app.appendChild(logoutLink);
    } else {
      const loginLink = document.createElement("a");
      loginLink.href = "/auth/login";
      loginLink.textContent = "Sign in with Vouch";
      app.appendChild(loginLink);
    }
  }

  checkAuth();
</script>
```

### Rich Authorization Requests

To request structured permissions, add `authorization_details` when building the authorization URL:

```javascript
const redirectTo = client.buildAuthorizationUrl(config, {
  // ... other params
  authorization_details: JSON.stringify([
    { type: "account_access", actions: ["read", "transfer"] },
  ]),
});
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
