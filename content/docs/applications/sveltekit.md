---
title: "SvelteKit (oidc-client-ts)"
description: "Integrate Vouch OIDC authentication into a SvelteKit single-page application using oidc-client-ts."
weight: 22
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

Create an auth service in `src/lib/auth.js`:

```javascript
import { UserManager, WebStorageStateStore } from "oidc-client-ts";

const userManager = new UserManager({
  authority: "https://{{< instance-url >}}",
  client_id: "your-client-id",
  redirect_uri: "https://your-app.example.com/callback",
  post_logout_redirect_uri: window.location.origin,
  scope: "openid email",
  response_type: "code",
  automaticSilentRenew: true,
  userStore: new WebStorageStateStore({ store: window.sessionStorage }),
});

export async function getUser() {
  return await userManager.getUser();
}

export async function login() {
  await userManager.signinRedirect();
}

export async function logout() {
  await userManager.removeUser();
  window.location.href = "/";
}

export async function handleCallback() {
  return await userManager.signinRedirectCallback();
}
```

Use it in a page component (`src/routes/+page.svelte`):

```svelte
<script>
  import { onMount } from "svelte";
  import { getUser, login, logout } from "$lib/auth";

  let user = $state(null);
  let loading = $state(true);

  onMount(async () => {
    user = await getUser();
    loading = false;
  });
</script>

{#if loading}
  <p>Loading...</p>
{:else if user}
  <p>Signed in as {user.profile.email}</p>
  {#if user.profile.hardware_verified}
    <p><strong>Hardware Verified</strong></p>
  {/if}
  <button onclick={logout}>Sign out</button>
{:else}
  <button onclick={login}>Sign in with Vouch</button>
{/if}
```

Create a callback page (`src/routes/callback/+page.svelte`):

```svelte
<script>
  import { onMount } from "svelte";
  import { goto } from "$app/navigation";
  import { handleCallback } from "$lib/auth";

  onMount(async () => {
    await handleCallback();
    goto("/");
  });
</script>

<p>Processing login...</p>
```

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` as an extra parameter in the UserManager configuration:

```javascript
const userManager = new UserManager({
  // ... other config
  extraQueryParams: {
    authorization_details: JSON.stringify([
      { type: "account_access", actions: ["read", "transfer"] },
    ]),
  },
});
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
