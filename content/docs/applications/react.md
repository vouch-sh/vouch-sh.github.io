---
title: "React (react-oidc-context)"
description: "Integrate Vouch OIDC authentication into a React single-page application using react-oidc-context."
weight: 10
params:
  category: "spa"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

For client-side applications running in the browser, use the authorization code flow with PKCE. These examples do not use a client secret since the code runs in the user's browser.

[react-oidc-context](https://github.com/authts/react-oidc-context) wraps [`oidc-client-ts`](https://github.com/authts/oidc-client-ts) in a React context provider.

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
