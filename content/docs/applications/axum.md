---
title: "Axum (openidconnect-rs)"
description: "Integrate Vouch OIDC authentication into a Rust Axum application using openidconnect-rs."
weight: 9
params:
  category: "server"
  language: "Rust"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

[openidconnect-rs](https://github.com/ramosbugs/openidconnect-rs) provides a type-safe OpenID Connect client for Rust.

Add the required dependencies to your `Cargo.toml`:

```toml
[dependencies]
axum = "0.7"
axum-extra = { version = "0.9", features = ["cookie"] }
openidconnect = "3"
reqwest = { version = "0.12", features = ["rustls-tls"], default-features = false }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
tower-sessions = "0.12"
```

Define custom claims to capture Vouch-specific fields from the ID token:

```rust
use openidconnect::AdditionalClaims;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Deserialize, Serialize)]
struct VouchClaims {
    #[serde(default)]
    hardware_verified: Option<bool>,
    #[serde(default)]
    hardware_aaguid: Option<String>,
}

impl AdditionalClaims for VouchClaims {}
```

This requires defining custom type aliases for the client and token types. See the [full example](https://github.com/vouch-sh/examples/tree/main/web/axum-openidconnect) for the complete type definitions.

Implement the OIDC integration:

```rust
use axum::{
    extract::{Query, State},
    response::{Html, IntoResponse, Redirect},
    routing::get,
    Router,
};
use openidconnect::{
    core::{CoreProviderMetadata, CoreResponseType},
    AuthenticationFlow, AuthorizationCode, ClientId, ClientSecret,
    CsrfToken, IssuerUrl, Nonce, PkceCodeChallenge, PkceCodeVerifier,
    RedirectUrl, Scope, TokenResponse,
    reqwest::async_http_client,
};
use std::{collections::HashMap, sync::Arc};
use tokio::sync::RwLock;
use tower_sessions::{MemoryStore, Session, SessionManagerLayer};

#[derive(Clone)]
struct AppState {
    client: VouchClient,
    pkce_verifiers: Arc<RwLock<HashMap<String, (PkceCodeVerifier, Nonce)>>>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let issuer_url = IssuerUrl::new(
        std::env::var("VOUCH_ISSUER").unwrap_or_else(|_| "https://{{< instance-url >}}".to_string()),
    )?;

    let provider_metadata =
        CoreProviderMetadata::discover_async(issuer_url, async_http_client).await?;

    let redirect_uri = std::env::var("VOUCH_REDIRECT_URI")
        .unwrap_or_else(|_| "http://localhost:3000/callback".to_string());

    let client = VouchClient::from_provider_metadata(
        provider_metadata,
        ClientId::new(std::env::var("VOUCH_CLIENT_ID").expect("VOUCH_CLIENT_ID must be set")),
        Some(ClientSecret::new(
            std::env::var("VOUCH_CLIENT_SECRET").expect("VOUCH_CLIENT_SECRET must be set"),
        )),
    )
    .set_redirect_uri(RedirectUrl::new(redirect_uri)?);

    let state = AppState {
        client,
        pkce_verifiers: Arc::new(RwLock::new(HashMap::new())),
    };

    let session_store = MemoryStore::default();
    let session_layer = SessionManagerLayer::new(session_store);

    let app = Router::new()
        .route("/", get(home))
        .route("/login", get(login))
        .route("/callback", get(callback))
        .route("/logout", get(logout))
        .layer(session_layer)
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

The login handler generates a PKCE challenge and redirects to the authorization endpoint:

```rust
async fn login(State(state): State<AppState>) -> Redirect {
    let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();

    let (auth_url, csrf_token, nonce) = state
        .client
        .authorize_url(
            AuthenticationFlow::<CoreResponseType>::AuthorizationCode,
            CsrfToken::new_random,
            Nonce::new_random,
        )
        .add_scope(Scope::new("email".to_string()))
        .set_pkce_challenge(pkce_challenge)
        .url();

    state.pkce_verifiers.write().await.insert(
        csrf_token.secret().clone(),
        (pkce_verifier, nonce),
    );

    Redirect::to(auth_url.as_str())
}
```

The callback handler exchanges the authorization code for tokens and extracts claims:

```rust
#[derive(Deserialize)]
struct CallbackParams {
    code: String,
    state: String,
}

async fn callback(
    Query(params): Query<CallbackParams>,
    State(state): State<AppState>,
    session: Session,
) -> impl IntoResponse {
    let (pkce_verifier, nonce) = match state.pkce_verifiers.write().await.remove(&params.state) {
        Some(v) => v,
        None => return Redirect::to("/").into_response(),
    };

    let token_response = match state
        .client
        .exchange_code(AuthorizationCode::new(params.code))
        .set_pkce_verifier(pkce_verifier)
        .request_async(async_http_client)
        .await
    {
        Ok(t) => t,
        Err(_) => return Redirect::to("/").into_response(),
    };

    let id_token = match token_response.id_token() {
        Some(t) => t,
        None => return Redirect::to("/").into_response(),
    };

    let claims = match id_token.claims(&state.client.id_token_verifier(), &nonce) {
        Ok(c) => c,
        Err(_) => return Redirect::to("/").into_response(),
    };

    let email = claims
        .email()
        .map(|e| e.as_str().to_string())
        .unwrap_or_default();
    let hardware_verified = claims
        .additional_claims()
        .hardware_verified
        .unwrap_or(false);

    let user = serde_json::json!({
        "email": email,
        "hardware_verified": hardware_verified,
    });

    let _ = session.insert("user", user).await;

    Redirect::to("/").into_response()
}
```

The `additional_claims()` method returns the `VouchClaims` struct with `hardware_verified` and `hardware_aaguid` fields.

```rust
async fn logout(session: Session) -> Redirect {
    let _ = session.remove::<serde_json::Value>("user").await;
    Redirect::to("/")
}
```

**Callback URL:** Register `http://localhost:3000/callback` as a redirect URI for your application.

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` as an extra query parameter on the authorization URL:

```rust
let (auth_url, csrf_token, nonce) = state
    .client
    .authorize_url(
        AuthenticationFlow::<CoreResponseType>::AuthorizationCode,
        CsrfToken::new_random,
        Nonce::new_random,
    )
    .add_scope(Scope::new("email".to_string()))
    .set_pkce_challenge(pkce_challenge)
    .set_extra_param(
        "authorization_details",
        r#"[{"type":"account_access","actions":["read","transfer"]}]"#,
    )
    .url();
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
