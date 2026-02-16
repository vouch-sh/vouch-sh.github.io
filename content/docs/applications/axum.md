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
