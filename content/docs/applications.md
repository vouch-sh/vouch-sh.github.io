---
title: "Applications (OIDC)"
description: "Add 'Sign in with Vouch' to your web application using OpenID Connect."
weight: 10
subtitle: "Add hardware-backed authentication to your web application"
---

Vouch is a fully compliant OpenID Connect (OIDC) provider. You can add "Sign in with Vouch" to any web application, single-page app, or native CLI tool using standard OIDC libraries. Every login is backed by a hardware security key, giving your application phishing-resistant authentication without building it yourself.

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

## Server-Side Frameworks

The following examples show how to integrate Vouch with popular server-side web frameworks using the authorization code flow.

### Rails (OmniAuth)

Add the `omniauth_openid_connect` gem to your `Gemfile`:

```ruby
gem 'omniauth_openid_connect'
```

Configure the provider in `config/initializers/omniauth.rb`:

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :openid_connect, {
    name: :vouch,
    scope: [:openid, :email],
    response_type: :code,
    issuer: "{{< instance-url >}}",
    discovery: true,
    client_options: {
      identifier: ENV["VOUCH_CLIENT_ID"],
      secret: ENV["VOUCH_CLIENT_SECRET"],
      redirect_uri: "https://your-app.example.com/auth/vouch/callback"
    }
  }
end
```

Add the callback route in `config/routes.rb`:

```ruby
get "/auth/vouch/callback", to: "sessions#create"
post "/auth/vouch/callback", to: "sessions#create"
```

Handle the callback in `app/controllers/sessions_controller.rb`:

```ruby
class SessionsController < ApplicationController
  def create
    auth = request.env["omniauth.auth"]
    user = User.find_or_create_by(vouch_id: auth.uid) do |u|
      u.email = auth.info.email
      u.name = auth.info.name
    end
    session[:user_id] = user.id
    redirect_to root_path, notice: "Signed in successfully."
  end
end
```

### Django (django-allauth)

Install django-allauth with OpenID Connect support:

```bash
pip install django-allauth[openid_connect]
```

Add the provider to your `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.openid_connect",
]

SOCIALACCOUNT_PROVIDERS = {
    "openid_connect": {
        "APPS": [
            {
                "provider_id": "vouch",
                "name": "Vouch",
                "client_id": os.environ["VOUCH_CLIENT_ID"],
                "secret": os.environ["VOUCH_CLIENT_SECRET"],
                "settings": {
                    "server_url": "{{< instance-url >}}",
                },
            },
        ],
    },
}
```

Add the allauth URLs to your `urls.py`:

```python
from django.urls import path, include

urlpatterns = [
    # ...
    path("accounts/", include("allauth.urls")),
]
```

### Express.js (Passport)

Install the required packages:

```bash
npm install passport passport-openidconnect express-session
```

Configure Passport in your Express application:

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
      issuer: "{{< instance-url >}}",
      authorizationURL: "{{< instance-url >}}/oauth/authorize",
      tokenURL: "{{< instance-url >}}/oauth/token",
      userInfoURL: "{{< instance-url >}}/oauth/userinfo",
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

### Next.js (NextAuth.js)

Install NextAuth.js:

```bash
npm install next-auth
```

Create the auth configuration in `app/api/auth/[...nextauth]/route.js`:

```javascript
import NextAuth from "next-auth";

const handler = NextAuth({
  providers: [
    {
      id: "vouch",
      name: "Vouch",
      type: "oidc",
      issuer: "{{< instance-url >}}",
      clientId: process.env.VOUCH_CLIENT_ID,
      clientSecret: process.env.VOUCH_CLIENT_SECRET,
      authorization: { params: { scope: "openid email" } },
      profile(profile) {
        return {
          id: profile.sub,
          name: profile.name,
          email: profile.email,
        };
      },
    },
  ],
  callbacks: {
    async jwt({ token, account, profile }) {
      if (account) {
        token.accessToken = account.access_token;
        token.vouchId = profile.sub;
      }
      return token;
    },
    async session({ session, token }) {
      session.accessToken = token.accessToken;
      session.user.vouchId = token.vouchId;
      return session;
    },
  },
});

export { handler as GET, handler as POST };
```

Add environment variables to `.env.local`:

```
NEXTAUTH_URL=https://your-app.example.com
NEXTAUTH_SECRET=your-session-secret
VOUCH_CLIENT_ID=your-client-id
VOUCH_CLIENT_SECRET=your-client-secret
```

### Laravel (Socialite)

Install the Socialite OpenID Connect driver:

```bash
composer require socialiteproviders/openid-connect
```

Add the provider configuration to `config/services.php`:

```php
'vouch' => [
    'client_id' => env('VOUCH_CLIENT_ID'),
    'client_secret' => env('VOUCH_CLIENT_SECRET'),
    'redirect' => env('VOUCH_REDIRECT_URI', 'https://your-app.example.com/auth/vouch/callback'),
    'discovery_url' => '{{< instance-url >}}/.well-known/openid-configuration',
],
```

Register the event listener in `app/Providers/EventServiceProvider.php`:

```php
protected $listen = [
    \SocialiteProviders\Manager\SocialiteWasCalled::class => [
        \SocialiteProviders\OpenIDConnect\OpenIDConnectExtendSocialite::class . '@handle',
    ],
];
```

Add routes in `routes/web.php`:

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/vouch', function () {
    return Socialite::driver('openid-connect')
        ->setConfig(config('services.vouch'))
        ->scopes(['openid', 'email'])
        ->redirect();
});

Route::get('/auth/vouch/callback', function () {
    $user = Socialite::driver('openid-connect')
        ->setConfig(config('services.vouch'))
        ->user();

    $localUser = User::updateOrCreate(
        ['vouch_id' => $user->getId()],
        [
            'name' => $user->getName(),
            'email' => $user->getEmail(),
        ]
    );

    Auth::login($localUser);
    return redirect('/dashboard');
});
```

### Flask (Authlib)

Install Authlib:

```bash
pip install authlib flask
```

Configure the OIDC client in your Flask application:

```python
from flask import Flask, redirect, url_for, session
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
app.secret_key = os.environ["FLASK_SECRET_KEY"]

oauth = OAuth(app)
vouch = oauth.register(
    name="vouch",
    client_id=os.environ["VOUCH_CLIENT_ID"],
    client_secret=os.environ["VOUCH_CLIENT_SECRET"],
    server_metadata_url="{{< instance-url >}}/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email"},
)


@app.route("/login")
def login():
    redirect_uri = url_for("callback", _external=True)
    return vouch.authorize_redirect(redirect_uri)


@app.route("/auth/callback")
def callback():
    token = vouch.authorize_access_token()
    userinfo = token["userinfo"]
    session["user"] = {
        "sub": userinfo["sub"],
        "name": userinfo.get("name"),
        "email": userinfo.get("email"),
    }
    return redirect("/")


@app.route("/")
def index():
    user = session.get("user")
    if user:
        return f"Hello, {user['name']}!"
    return '<a href="/login">Sign in with Vouch</a>'
```

### FastAPI (Authlib)

Install the required packages:

```bash
pip install authlib httpx fastapi uvicorn itsdangerous
```

Configure the OIDC client in your FastAPI application:

```python
import os
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse
from authlib.integrations.starlette_client import OAuth
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI()
app.add_middleware(SessionMiddleware, secret_key=os.environ["SESSION_SECRET"])

oauth = OAuth()
oauth.register(
    name="vouch",
    client_id=os.environ["VOUCH_CLIENT_ID"],
    client_secret=os.environ["VOUCH_CLIENT_SECRET"],
    server_metadata_url="{{< instance-url >}}/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email"},
)


@app.get("/login")
async def login(request: Request):
    redirect_uri = request.url_for("callback")
    return await oauth.vouch.authorize_redirect(request, redirect_uri)


@app.get("/auth/callback")
async def callback(request: Request):
    token = await oauth.vouch.authorize_access_token(request)
    userinfo = token["userinfo"]
    request.session["user"] = {
        "sub": userinfo["sub"],
        "name": userinfo.get("name"),
        "email": userinfo.get("email"),
    }
    return RedirectResponse(url="/")


@app.get("/")
async def index(request: Request):
    user = request.session.get("user")
    if user:
        return {"message": f"Hello, {user['name']}!"}
    return {"message": "Not authenticated. Visit /login to sign in."}
```

### Spring Boot (Spring Security OAuth2)

Add the Spring Security OAuth2 Client dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

Configure the OIDC provider in `application.yml`:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          vouch:
            client-id: ${VOUCH_CLIENT_ID}
            client-secret: ${VOUCH_CLIENT_SECRET}
            scope: openid, email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/vouch"
        provider:
          vouch:
            issuer-uri: "{{< instance-url >}}"
```

Configure Spring Security in your application:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("/dashboard", true)
            );
        return http.build();
    }
}
```

Create a controller to access user information:

```java
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DashboardController {

    @GetMapping("/dashboard")
    public String dashboard(@AuthenticationPrincipal OidcUser user, Model model) {
        model.addAttribute("name", user.getFullName());
        model.addAttribute("email", user.getEmail());
        model.addAttribute("sub", user.getSubject());
        return "dashboard";
    }
}
```

### Axum (openidconnect-rs)

Add the required dependencies to your `Cargo.toml`:

```toml
[dependencies]
axum = "0.7"
axum-extra = { version = "0.9", features = ["cookie"] }
openidconnect = "3"
reqwest = { version = "0.12", features = ["rustls-tls"] }
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tower-sessions = "0.13"
url = "2"
```

Implement the OIDC integration:

```rust
use axum::{
    extract::{Query, State},
    response::{IntoResponse, Redirect},
    routing::get,
    Router,
};
use openidconnect::{
    core::{CoreClient, CoreProviderMetadata, CoreResponseType},
    reqwest::async_http_client,
    AuthenticationFlow, AuthorizationCode, ClientId, ClientSecret,
    CsrfToken, IssuerUrl, Nonce, RedirectUrl, Scope, TokenResponse,
};
use serde::Deserialize;
use std::sync::Arc;

struct AppState {
    oidc_client: CoreClient,
}

#[derive(Deserialize)]
struct CallbackParams {
    code: String,
    state: String,
}

async fn login(State(state): State<Arc<AppState>>) -> impl IntoResponse {
    let (auth_url, _csrf_token, _nonce) = state
        .oidc_client
        .authorize_url(
            AuthenticationFlow::<CoreResponseType>::AuthorizationCode,
            CsrfToken::new_random,
            Nonce::new_random,
        )
        .add_scope(Scope::new("email".to_string()))
        .url();

    Redirect::to(auth_url.as_str())
}

async fn callback(
    State(state): State<Arc<AppState>>,
    Query(params): Query<CallbackParams>,
) -> impl IntoResponse {
    let token_response = state
        .oidc_client
        .exchange_code(AuthorizationCode::new(params.code))
        .request_async(async_http_client)
        .await
        .expect("Failed to exchange code");

    let id_token = token_response
        .id_token()
        .expect("Server did not return an ID token");

    let claims = id_token
        .claims(&state.oidc_client.id_token_verifier(), &Nonce::new_random())
        .expect("Failed to verify ID token");

    format!(
        "Welcome! Subject: {}, Email: {:?}",
        claims.subject(),
        claims.email()
    )
}

#[tokio::main]
async fn main() {
    let issuer_url =
        IssuerUrl::new("{{< instance-url >}}".to_string()).expect("Invalid issuer URL");

    let provider_metadata =
        CoreProviderMetadata::discover_async(issuer_url, async_http_client)
            .await
            .expect("Failed to discover provider");

    let client = CoreClient::from_provider_metadata(
        provider_metadata,
        ClientId::new(std::env::var("VOUCH_CLIENT_ID").expect("VOUCH_CLIENT_ID not set")),
        Some(ClientSecret::new(
            std::env::var("VOUCH_CLIENT_SECRET").expect("VOUCH_CLIENT_SECRET not set"),
        )),
    )
    .set_redirect_uri(
        RedirectUrl::new("https://your-app.example.com/auth/callback".to_string())
            .expect("Invalid redirect URL"),
    );

    let state = Arc::new(AppState {
        oidc_client: client,
    });

    let app = Router::new()
        .route("/login", get(login))
        .route("/auth/callback", get(callback))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .expect("Failed to bind");
    axum::serve(listener, app).await.expect("Server error");
}
```

---

## Single Page Applications

For client-side applications running in the browser, use the authorization code flow with PKCE. These examples do not use a client secret since the code runs in the user's browser.

### React (react-oidc-context)

Install the library:

```bash
npm install react-oidc-context oidc-client-ts
```

Configure the OIDC provider in your application:

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { AuthProvider, useAuth } from "react-oidc-context";

const oidcConfig = {
  authority: "{{< instance-url >}}",
  client_id: "your-client-id",
  redirect_uri: "https://your-app.example.com/callback",
  scope: "openid email",
  response_type: "code",
};

function App() {
  const auth = useAuth();

  if (auth.isLoading) {
    return <div>Loading...</div>;
  }

  if (auth.error) {
    return <div>Error: {auth.error.message}</div>;
  }

  if (auth.isAuthenticated) {
    return (
      <div>
        <h1>Welcome, {auth.user?.profile.name}</h1>
        <p>Email: {auth.user?.profile.email}</p>
        <p>Subject: {auth.user?.profile.sub}</p>
        <button onClick={() => auth.removeUser()}>Sign out</button>
      </div>
    );
  }

  return (
    <div>
      <h1>My Application</h1>
      <button onClick={() => auth.signinRedirect()}>Sign in with Vouch</button>
    </div>
  );
}

function CallbackPage() {
  const auth = useAuth();

  React.useEffect(() => {
    if (!auth.isLoading && !auth.isAuthenticated) {
      auth.signinRedirect();
    }
  }, [auth]);

  return <div>Processing login...</div>;
}

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <AuthProvider {...oidcConfig}>
    <App />
  </AuthProvider>
);
```

### Vue (oidc-client-ts)

Install the library:

```bash
npm install oidc-client-ts
```

Create an auth service in `src/auth.js`:

```javascript
import { UserManager, WebStorageStateStore } from "oidc-client-ts";

const userManager = new UserManager({
  authority: "{{< instance-url >}}",
  client_id: "your-client-id",
  redirect_uri: "https://your-app.example.com/callback",
  scope: "openid email",
  response_type: "code",
  userStore: new WebStorageStateStore({ store: window.localStorage }),
});

export async function login() {
  await userManager.signinRedirect();
}

export async function handleCallback() {
  return await userManager.signinRedirectCallback();
}

export async function getUser() {
  return await userManager.getUser();
}

export async function logout() {
  await userManager.removeUser();
}

export default userManager;
```

Use it in a Vue component (`src/App.vue`):

```vue
<template>
  <div>
    <div v-if="loading">Loading...</div>
    <div v-else-if="user">
      <h1>Welcome, {{ user.profile.name }}</h1>
      <p>Email: {{ user.profile.email }}</p>
      <button @click="handleLogout">Sign out</button>
    </div>
    <div v-else>
      <h1>My Application</h1>
      <button @click="handleLogin">Sign in with Vouch</button>
    </div>
  </div>
</template>

<script>
import { login, getUser, logout } from "./auth";

export default {
  data() {
    return {
      user: null,
      loading: true,
    };
  },
  async mounted() {
    try {
      this.user = await getUser();
    } catch (e) {
      console.error("Failed to get user:", e);
    } finally {
      this.loading = false;
    }
  },
  methods: {
    handleLogin() {
      login();
    },
    async handleLogout() {
      await logout();
      this.user = null;
    },
  },
};
</script>
```

Create a callback page (`src/Callback.vue`):

```vue
<template>
  <div>Processing login...</div>
</template>

<script>
import { handleCallback } from "./auth";

export default {
  async mounted() {
    try {
      await handleCallback();
      this.$router.push("/");
    } catch (e) {
      console.error("Callback error:", e);
      this.$router.push("/login");
    }
  },
};
</script>
```

### Vanilla JS

For applications without a framework, use `oidc-client-ts` directly:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Vouch OIDC Example</title>
</head>
<body>
  <div id="app">
    <div id="loading">Loading...</div>
    <div id="authenticated" style="display: none">
      <h1>Welcome, <span id="user-name"></span></h1>
      <p>Email: <span id="user-email"></span></p>
      <button id="logout-btn">Sign out</button>
    </div>
    <div id="unauthenticated" style="display: none">
      <h1>My Application</h1>
      <button id="login-btn">Sign in with Vouch</button>
    </div>
  </div>

  <script type="module">
    import { UserManager } from "https://cdn.jsdelivr.net/npm/oidc-client-ts/dist/browser/oidc-client-ts.min.js";

    const config = {
      authority: "{{< instance-url >}}",
      client_id: "your-client-id",
      redirect_uri: window.location.origin + "/callback",
      scope: "openid email",
      response_type: "code",
    };

    const userManager = new UserManager(config);

    async function init() {
      // Handle callback
      if (window.location.pathname === "/callback") {
        try {
          await userManager.signinRedirectCallback();
          window.location.href = "/";
        } catch (e) {
          console.error("Callback error:", e);
        }
        return;
      }

      // Check for existing session
      const user = await userManager.getUser();
      document.getElementById("loading").style.display = "none";

      if (user && !user.expired) {
        document.getElementById("user-name").textContent = user.profile.name;
        document.getElementById("user-email").textContent = user.profile.email;
        document.getElementById("authenticated").style.display = "block";
      } else {
        document.getElementById("unauthenticated").style.display = "block";
      }
    }

    document.getElementById("login-btn")?.addEventListener("click", () => {
      userManager.signinRedirect();
    });

    document.getElementById("logout-btn")?.addEventListener("click", async () => {
      await userManager.removeUser();
      window.location.href = "/";
    });

    init();
  </script>
</body>
</html>
```

---

## Native & CLI Applications

For native desktop applications and CLI tools that cannot open a browser redirect, use the **Device Authorization Grant** (RFC 8628). This flow displays a URL and code that the user enters in a browser on any device.

### Device Authorization Flow

1. Your application requests a device code from the Vouch server.
2. The server returns a `device_code`, a `user_code`, and a `verification_uri`.
3. Your application displays the `user_code` and `verification_uri` to the user.
4. The user opens `verification_uri` in a browser, enters the code, and authenticates with their YubiKey.
5. Your application polls the token endpoint until the user completes authentication.
6. Once approved, the token endpoint returns an access token and ID token.

### Python

```python
import os
import time
import requests

VOUCH_URL = "{{< instance-url >}}"
CLIENT_ID = os.environ["VOUCH_CLIENT_ID"]

# Step 1: Request device code
device_response = requests.post(
    f"{VOUCH_URL}/oauth/device/code",
    data={
        "client_id": CLIENT_ID,
        "scope": "openid email",
    },
)
device_data = device_response.json()

device_code = device_data["device_code"]
user_code = device_data["user_code"]
verification_uri = device_data["verification_uri"]
interval = device_data.get("interval", 5)
expires_in = device_data.get("expires_in", 600)

# Step 2: Display instructions to the user
print(f"\nTo sign in, open this URL in your browser:\n")
print(f"  {verification_uri}\n")
print(f"And enter the code: {user_code}\n")
print("Waiting for authentication...")

# Step 3: Poll for the token
deadline = time.time() + expires_in
while time.time() < deadline:
    time.sleep(interval)

    token_response = requests.post(
        f"{VOUCH_URL}/oauth/token",
        data={
            "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
            "device_code": device_code,
            "client_id": CLIENT_ID,
        },
    )

    if token_response.status_code == 200:
        tokens = token_response.json()
        print(f"\nAuthenticated successfully!")
        print(f"Access token: {tokens['access_token'][:20]}...")

        # Fetch user info
        userinfo = requests.get(
            f"{VOUCH_URL}/oauth/userinfo",
            headers={"Authorization": f"Bearer {tokens['access_token']}"},
        ).json()
        print(f"Hello, {userinfo.get('name', 'user')}!")
        break

    error_data = token_response.json()
    error = error_data.get("error")

    if error == "authorization_pending":
        continue
    elif error == "slow_down":
        interval += 5
        continue
    elif error == "expired_token":
        print("The device code has expired. Please try again.")
        break
    elif error == "access_denied":
        print("Authentication was denied.")
        break
    else:
        print(f"Unexpected error: {error}")
        break
else:
    print("Timed out waiting for authentication.")
```

### Node.js

```javascript
const https = require("https");
const querystring = require("querystring");

const VOUCH_URL = "{{< instance-url >}}";
const CLIENT_ID = process.env.VOUCH_CLIENT_ID;

function post(path, data) {
  return new Promise((resolve, reject) => {
    const url = new URL(path, VOUCH_URL);
    const body = querystring.stringify(data);
    const req = https.request(
      url,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          "Content-Length": Buffer.byteLength(body),
        },
      },
      (res) => {
        let data = "";
        res.on("data", (chunk) => (data += chunk));
        res.on("end", () =>
          resolve({ status: res.statusCode, body: JSON.parse(data) })
        );
      }
    );
    req.on("error", reject);
    req.write(body);
    req.end();
  });
}

function get(path, token) {
  return new Promise((resolve, reject) => {
    const url = new URL(path, VOUCH_URL);
    const req = https.request(
      url,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${token}` },
      },
      (res) => {
        let data = "";
        res.on("data", (chunk) => (data += chunk));
        res.on("end", () => resolve(JSON.parse(data)));
      }
    );
    req.on("error", reject);
    req.end();
  });
}

async function main() {
  // Step 1: Request device code
  const deviceResponse = await post("/oauth/device/code", {
    client_id: CLIENT_ID,
    scope: "openid email",
  });

  const { device_code, user_code, verification_uri, interval = 5, expires_in = 600 } =
    deviceResponse.body;

  // Step 2: Display instructions
  console.log(`\nTo sign in, open this URL in your browser:\n`);
  console.log(`  ${verification_uri}\n`);
  console.log(`And enter the code: ${user_code}\n`);
  console.log("Waiting for authentication...");

  // Step 3: Poll for the token
  let pollInterval = interval;
  const deadline = Date.now() + expires_in * 1000;

  while (Date.now() < deadline) {
    await new Promise((r) => setTimeout(r, pollInterval * 1000));

    const tokenResponse = await post("/oauth/token", {
      grant_type: "urn:ietf:params:oauth:grant-type:device_code",
      device_code,
      client_id: CLIENT_ID,
    });

    if (tokenResponse.status === 200) {
      console.log("\nAuthenticated successfully!");

      const userinfo = await get("/oauth/userinfo", tokenResponse.body.access_token);
      console.log(`Hello, ${userinfo.name || "user"}!`);
      return;
    }

    const error = tokenResponse.body.error;

    if (error === "authorization_pending") continue;
    if (error === "slow_down") {
      pollInterval += 5;
      continue;
    }
    if (error === "expired_token") {
      console.log("The device code has expired. Please try again.");
      return;
    }
    if (error === "access_denied") {
      console.log("Authentication was denied.");
      return;
    }

    console.log(`Unexpected error: ${error}`);
    return;
  }

  console.log("Timed out waiting for authentication.");
}

main().catch(console.error);
```

### Rust

Add the required dependencies to your `Cargo.toml`:

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

Implement the device authorization flow:

```rust
use serde::Deserialize;
use std::env;
use std::time::{Duration, Instant};

#[derive(Deserialize)]
struct DeviceCodeResponse {
    device_code: String,
    user_code: String,
    verification_uri: String,
    #[serde(default = "default_interval")]
    interval: u64,
    #[serde(default = "default_expires_in")]
    expires_in: u64,
}

fn default_interval() -> u64 { 5 }
fn default_expires_in() -> u64 { 600 }

#[derive(Deserialize)]
struct TokenResponse {
    access_token: String,
}

#[derive(Deserialize)]
struct ErrorResponse {
    error: String,
}

#[derive(Deserialize)]
struct UserInfo {
    name: Option<String>,
    email: Option<String>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let vouch_url = "{{< instance-url >}}";
    let client_id = env::var("VOUCH_CLIENT_ID")
        .expect("VOUCH_CLIENT_ID environment variable not set");

    let client = reqwest::Client::new();

    // Step 1: Request device code
    let device_response: DeviceCodeResponse = client
        .post(format!("{vouch_url}/oauth/device/code"))
        .form(&[
            ("client_id", client_id.as_str()),
            ("scope", "openid email"),
        ])
        .send()
        .await?
        .json()
        .await?;

    // Step 2: Display instructions
    println!("\nTo sign in, open this URL in your browser:\n");
    println!("  {}\n", device_response.verification_uri);
    println!("And enter the code: {}\n", device_response.user_code);
    println!("Waiting for authentication...");

    // Step 3: Poll for the token
    let mut interval = Duration::from_secs(device_response.interval);
    let deadline = Instant::now() + Duration::from_secs(device_response.expires_in);

    loop {
        tokio::time::sleep(interval).await;

        if Instant::now() > deadline {
            println!("Timed out waiting for authentication.");
            break;
        }

        let response = client
            .post(format!("{vouch_url}/oauth/token"))
            .form(&[
                ("grant_type", "urn:ietf:params:oauth:grant-type:device_code"),
                ("device_code", &device_response.device_code),
                ("client_id", &client_id),
            ])
            .send()
            .await?;

        if response.status().is_success() {
            let tokens: TokenResponse = response.json().await?;
            println!("\nAuthenticated successfully!");

            let userinfo: UserInfo = client
                .get(format!("{vouch_url}/oauth/userinfo"))
                .bearer_auth(&tokens.access_token)
                .send()
                .await?
                .json()
                .await?;

            println!(
                "Hello, {}!",
                userinfo.name.unwrap_or_else(|| "user".to_string())
            );
            break;
        }

        let error: ErrorResponse = response.json().await?;
        match error.error.as_str() {
            "authorization_pending" => continue,
            "slow_down" => {
                interval += Duration::from_secs(5);
                continue;
            }
            "expired_token" => {
                println!("The device code has expired. Please try again.");
                break;
            }
            "access_denied" => {
                println!("Authentication was denied.");
                break;
            }
            other => {
                println!("Unexpected error: {other}");
                break;
            }
        }
    }

    Ok(())
}
```

---

## Device Authorization Response

When your application requests a device code, the server returns the following fields:

| Field | Type | Description |
|---|---|---|
| `device_code` | string | The device verification code. Used by your application when polling the token endpoint. Do not display this to the user. |
| `user_code` | string | The end-user verification code. Display this to the user so they can enter it in their browser. |
| `verification_uri` | string | The URL the user should visit to enter the code and authenticate. |
| `verification_uri_complete` | string | Optional. A URL that includes the `user_code`, so the user can navigate directly without manually entering the code. |
| `expires_in` | number | The lifetime of the `device_code` and `user_code` in seconds. |
| `interval` | number | The minimum number of seconds your application should wait between polling requests. |

---

## Polling Errors

When polling the token endpoint during the device authorization flow, the server may return the following error codes:

| Error | Description | Action |
|---|---|---|
| `authorization_pending` | The user has not yet completed authentication. | Continue polling at the specified interval. |
| `slow_down` | Your application is polling too frequently. | Increase the polling interval by 5 seconds and continue. |
| `expired_token` | The `device_code` has expired. | Stop polling. Restart the flow by requesting a new device code. |
| `access_denied` | The user denied the authorization request. | Stop polling. Inform the user that authentication was denied. |

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
