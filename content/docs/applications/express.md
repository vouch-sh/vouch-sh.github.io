---
title: "Express.js (Passport)"
description: "Integrate Vouch OIDC authentication into an Express.js application using Passport."
weight: 3
params:
  category: "server"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install the required packages:

```bash
npm install passport passport-openidconnect express-session
```

Configure [Passport](https://www.passportjs.org/) in your Express application:

```javascript
const express = require("express");
const session = require("express-session");
const passport = require("passport");
const OpenIDConnectStrategy = require("passport-openidconnect");

const app = express();

app.use(
  session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
  })
);

app.use(passport.initialize());
app.use(passport.session());

passport.use(
  "vouch",
  new OpenIDConnectStrategy(
    {
      issuer: "https://{{< instance-url >}}",
      authorizationURL: "https://{{< instance-url >}}/oauth/authorize",
      tokenURL: "https://{{< instance-url >}}/oauth/token",
      userInfoURL: "https://{{< instance-url >}}/oauth/userinfo",
      clientID: process.env.VOUCH_CLIENT_ID,
      clientSecret: process.env.VOUCH_CLIENT_SECRET,
      callbackURL: "https://your-app.example.com/auth/vouch/callback",
      scope: "openid email",
    },
    (issuer, profile, done) => {
      // Find or create user based on profile
      return done(null, profile);
    }
  )
);

passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((user, done) => done(null, user));

app.get("/auth/vouch", passport.authenticate("vouch"));

app.get(
  "/auth/vouch/callback",
  passport.authenticate("vouch", {
    successRedirect: "/",
    failureRedirect: "/login",
  })
);

app.listen(3000);
```

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` as an extra authorization parameter. Passport's OpenID Connect strategy does not support extra parameters directly, so include them in the authorization URL via a custom redirect:

```javascript
app.get("/auth/vouch", (req, res) => {
  const authorizationDetails = JSON.stringify([
    { type: "account_access", actions: ["read", "transfer"] },
  ]);
  const params = new URLSearchParams({
    authorization_details: authorizationDetails,
  });
  passport.authenticate("vouch", {
    state: req.query.state,
    additionalParams: { authorization_details: authorizationDetails },
  })(req, res);
});
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
