---
title: "Go (go-oidc)"
description: "Integrate Vouch OIDC authentication into a Go application using go-oidc and oauth2."
weight: 10
params:
  category: "server"
  language: "Go"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

[go-oidc](https://github.com/coreos/go-oidc) and the standard [oauth2](https://pkg.go.dev/golang.org/x/oauth2) package provide OIDC support for Go applications.

Install the dependencies:

```bash
go get github.com/coreos/go-oidc/v3/oidc
go get golang.org/x/oauth2
```

### Provider discovery

Use OIDC discovery to configure the provider automatically:

```go
import (
    "github.com/coreos/go-oidc/v3/oidc"
    "golang.org/x/oauth2"
)

issuer := os.Getenv("VOUCH_ISSUER")
if issuer == "" {
    issuer = "https://{{< instance-url >}}"
}

provider, err := oidc.NewProvider(ctx, issuer)
if err != nil {
    log.Fatalf("Failed to create OIDC provider: %v", err)
}

redirectURL := os.Getenv("VOUCH_REDIRECT_URI")
if redirectURL == "" {
    redirectURL = "http://localhost:3000/callback"
}

verifier := provider.Verifier(&oidc.Config{ClientID: os.Getenv("VOUCH_CLIENT_ID")})

oauth2Config := &oauth2.Config{
    ClientID:     os.Getenv("VOUCH_CLIENT_ID"),
    ClientSecret: os.Getenv("VOUCH_CLIENT_SECRET"),
    RedirectURL:  redirectURL,
    Endpoint:     provider.Endpoint(),
    Scopes:       []string{oidc.ScopeOpenID, "email"},
}
```

### Login handler

Generate a PKCE verifier and redirect to the authorization endpoint:

```go
func handleLogin(w http.ResponseWriter, r *http.Request) {
    state := generateRandomState()
    pkceVerifier := oauth2.GenerateVerifier()

    // Store state and verifier in session (in-memory map shown for brevity)
    states[state] = true
    pkceVerifiers[state] = pkceVerifier

    http.Redirect(w, r, oauth2Config.AuthCodeURL(
        state,
        oauth2.S256ChallengeOption(pkceVerifier),
    ), http.StatusFound)
}
```

### Callback handler

Exchange the authorization code using the PKCE verifier, then verify and extract claims from the ID token:

```go
func handleCallback(w http.ResponseWriter, r *http.Request) {
    state := r.URL.Query().Get("state")
    if !states[state] {
        http.Error(w, "Invalid state", http.StatusBadRequest)
        return
    }
    delete(states, state)

    pkceVerifier := pkceVerifiers[state]
    delete(pkceVerifiers, state)

    code := r.URL.Query().Get("code")
    token, err := oauth2Config.Exchange(
        r.Context(), code, oauth2.VerifierOption(pkceVerifier),
    )
    if err != nil {
        http.Error(w, "Token exchange failed", http.StatusInternalServerError)
        return
    }

    rawIDToken, ok := token.Extra("id_token").(string)
    if !ok {
        http.Error(w, "No ID token", http.StatusInternalServerError)
        return
    }

    idToken, err := verifier.Verify(r.Context(), rawIDToken)
    if err != nil {
        http.Error(w, "Token verification failed", http.StatusInternalServerError)
        return
    }

    var claims struct {
        Email            string `json:"email"`
        HardwareVerified bool   `json:"hardware_verified"`
    }
    if err := idToken.Claims(&claims); err != nil {
        http.Error(w, "Failed to parse claims", http.StatusInternalServerError)
        return
    }

    // claims.Email and claims.HardwareVerified are now available
}
```

### Rich Authorization Requests

To request structured permissions beyond scopes, include `authorization_details` as a query parameter when building the authorization URL:

```go
import "encoding/json"

authzDetails, _ := json.Marshal([]map[string]interface{}{
    {"type": "account_access", "actions": []string{"read", "transfer"}},
})

url := oauth2Config.AuthCodeURL(
    state,
    oauth2.S256ChallengeOption(pkceVerifier),
    oauth2.SetAuthURLParam("authorization_details", string(authzDetails)),
)
http.Redirect(w, r, url, http.StatusFound)
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
