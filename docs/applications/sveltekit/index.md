# SvelteKit (oidc-client-ts)

> Integrate Vouch OIDC authentication into a SvelteKit single-page application using oidc-client-ts.

Source: https://vouch.sh/docs/applications/sveltekit/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[oidc-client-ts](https://github.com/authts/oidc-client-ts) integrates with SvelteKit using Svelte 5 reactivity (`$state` and `$derived`). Key configuration:

- No client secret needed (public client with PKCE, enabled by default)
- Vouch does not issue refresh tokens — redirect the user to sign in again when the token expires
- Uses `sessionStorage` for state persistence
- Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are in the access token JWT — decode with `atob(token.split('.')[1])` after base64url character replacement

## Example

**[spa/sveltekit](https://github.com/vouch-sh/examples/tree/main/spa/sveltekit)** — Complete working example with oidc-client-ts, Svelte 5 reactivity, PKCE, and hardware claim extraction.
