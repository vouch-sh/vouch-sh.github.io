# Vanilla JavaScript

> Integrate Vouch OIDC authentication into a plain JavaScript application using oidc-client-ts.

Source: https://vouch.sh/docs/applications/vanilla-js/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[oidc-client-ts](https://github.com/authts/oidc-client-ts) works with plain JavaScript — no framework required. It can be loaded via CDN (`https://cdn.jsdelivr.net/npm/oidc-client-ts`). Key configuration:

- No client secret needed (public client with PKCE, enabled by default)
- Vouch does not issue refresh tokens — redirect the user to sign in again when the token expires
- Initialize `UserManager` and call `signinRedirectCallback()` on the callback page
- Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are in the access token JWT — decode with `atob(token.split('.')[1])` after base64url character replacement

## Example

**[spa/vanilla-js](https://github.com/vouch-sh/examples/tree/main/spa/vanilla-js)** — Complete working example with oidc-client-ts, PKCE, and hardware claim extraction.
