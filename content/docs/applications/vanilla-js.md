---
title: "Vanilla JavaScript"
description: "Integrate Vouch OIDC authentication into a plain JavaScript application using oidc-client-ts."
weight: 12
params:
  category: "spa"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

For client-side applications running in the browser, use the authorization code flow with PKCE. These examples do not use a client secret since the code runs in the user's browser.

For applications without a framework, use [`oidc-client-ts`](https://github.com/authts/oidc-client-ts) directly:

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
