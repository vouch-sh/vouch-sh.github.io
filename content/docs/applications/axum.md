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
use tower_sessions::{MemoryStore, Session, SessionManagerLayer};

const CSRF_TOKEN_KEY: &str = "csrf_token";
const NONCE_KEY: &str = "nonce";

struct AppState {
    oidc_client: CoreClient,
}

#[derive(Deserialize)]
struct CallbackParams {
    code: String,
    state: String,
}

async fn login(
    State(state): State<Arc<AppState>>,
    session: Session,
) -> impl IntoResponse {
    let (auth_url, csrf_token, nonce) = state
        .oidc_client
        .authorize_url(
            AuthenticationFlow::<CoreResponseType>::AuthorizationCode,
            CsrfToken::new_random,
            Nonce::new_random,
        )
        .add_scope(Scope::new("email".to_string()))
        .url();

    // Store the CSRF token and nonce in the session so we can verify them
    // when the provider redirects back to the callback endpoint.
    session
        .insert(CSRF_TOKEN_KEY, csrf_token.secret().clone())
        .await
        .expect("Failed to store CSRF token");
    session
        .insert(NONCE_KEY, nonce.secret().clone())
        .await
        .expect("Failed to store nonce");

    Redirect::to(auth_url.as_str())
}

async fn callback(
    State(state): State<Arc<AppState>>,
    session: Session,
    Query(params): Query<CallbackParams>,
) -> impl IntoResponse {
    // Retrieve and remove the CSRF token from the session.
    let stored_csrf: String = session
        .remove(CSRF_TOKEN_KEY)
        .await
        .expect("Session error")
        .expect("Missing CSRF token in session -- login flow was not initiated");

    // Verify the state parameter matches the CSRF token we stored.
    if params.state != stored_csrf {
        panic!("CSRF token mismatch -- possible CSRF attack");
    }

    // Retrieve and remove the nonce from the session.
    let stored_nonce: String = session
        .remove(NONCE_KEY)
        .await
        .expect("Session error")
        .expect("Missing nonce in session");
    let nonce = Nonce::new(stored_nonce);

    let token_response = state
        .oidc_client
        .exchange_code(AuthorizationCode::new(params.code))
        .request_async(async_http_client)
        .await
        .expect("Failed to exchange code");

    let id_token = token_response
        .id_token()
        .expect("Server did not return an ID token");

    // Verify the ID token using the nonce from the original authorization request.
    let claims = id_token
        .claims(&state.oidc_client.id_token_verifier(), &nonce)
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

    // Use an in-memory session store. In production, replace with a
    // persistent store (e.g., Redis) for multi-instance deployments.
    let session_store = MemoryStore::default();
    let session_layer = SessionManagerLayer::new(session_store);

    let app = Router::new()
        .route("/login", get(login))
        .route("/auth/callback", get(callback))
        .layer(session_layer)
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .expect("Failed to bind");
    axum::serve(listener, app).await.expect("Server error");
}
```
