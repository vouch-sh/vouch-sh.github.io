---
title: "Vue (oidc-client-ts)"
description: "Integrate Vouch OIDC authentication into a Vue single-page application using oidc-client-ts."
weight: 11
params:
  category: "spa"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

For client-side applications running in the browser, use the authorization code flow with PKCE. These examples do not use a client secret since the code runs in the user's browser.

> **PKCE is required** for browser-based clients. `oidc-client-ts` enables PKCE by default -- no additional configuration is needed. To handle token expiry, enable `automaticSilentRenew` in the UserManager configuration (shown below) so tokens are refreshed transparently before they expire.

[oidc-client-ts](https://github.com/authts/oidc-client-ts) provides a certified OpenID Connect client for browser-based applications.

Install the library:

```bash
npm install oidc-client-ts
```

Create an auth service in `src/auth.js`:

```javascript
import { UserManager, WebStorageStateStore } from "oidc-client-ts";

const userManager = new UserManager({
  authority: "https://{{< instance-url >}}",
  client_id: "your-client-id",
  redirect_uri: "https://your-app.example.com/callback",
  scope: "openid email",
  response_type: "code",
  automaticSilentRenew: true,
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
