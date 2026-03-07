---
title: "Express.js (openid-client)"
description: "Integrate Vouch OIDC authentication into an Express.js application using openid-client."
weight: 3
params:
  category: "server"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install the required packages:

```bash
npm install openid-client express-session
```

Configure [openid-client](https://github.com/panva/openid-client) in your Express application:

```javascript
import express from 'express';
import session from 'express-session';
import * as client from 'openid-client';

const clientId = process.env.VOUCH_CLIENT_ID;
const clientSecret = process.env.VOUCH_CLIENT_SECRET;
const callbackUrl = 'https://your-app.example.com/auth/vouch/callback';

const config = await client.discovery(
  new URL('https://{{< instance-url >}}'),
  clientId,
  clientSecret
);

const app = express();

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
```

### Login route

Generate a PKCE code verifier and challenge, then redirect the user to the Vouch authorization endpoint:

```javascript
app.get('/auth/vouch', async (req, res) => {
  const codeVerifier = client.randomPKCECodeVerifier();
  const codeChallenge =
    await client.calculatePKCECodeChallenge(codeVerifier);
  const state = client.randomState();
  const nonce = client.randomNonce();

  req.session.oidc = { codeVerifier, state, nonce };

  const redirectTo = client.buildAuthorizationUrl(config, {
    redirect_uri: callbackUrl,
    scope: 'openid email',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state,
    nonce,
  });

  res.redirect(redirectTo.href);
});
```

### Callback route

Exchange the authorization code for tokens and extract claims from the ID token:

```javascript
app.get('/auth/vouch/callback', async (req, res) => {
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
    }
  );

  const claims = tokens.claims();
  req.session.user = {
    id: claims.sub,
    email: claims.email,
    hardwareVerified: claims.hardware_verified || false,
    hardwareAaguid: claims.hardware_aaguid || null,
  };

  res.redirect('/');
});

app.listen(3000);
```

The `hardware_verified` claim indicates whether the user authenticated with a hardware security key. When `true`, `hardware_aaguid` contains the [AAGUID](https://fidoalliance.org/metadata/) of the authenticator device.

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` when building the authorization URL:

```javascript
app.get('/auth/vouch', async (req, res) => {
  const codeVerifier = client.randomPKCECodeVerifier();
  const codeChallenge =
    await client.calculatePKCECodeChallenge(codeVerifier);
  const state = client.randomState();
  const nonce = client.randomNonce();

  req.session.oidc = { codeVerifier, state, nonce };

  const authorizationDetails = JSON.stringify([
    { type: 'account_access', actions: ['read', 'transfer'] },
  ]);

  const redirectTo = client.buildAuthorizationUrl(config, {
    redirect_uri: callbackUrl,
    scope: 'openid email',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state,
    nonce,
    authorization_details: authorizationDetails,
  });

  res.redirect(redirectTo.href);
});
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
