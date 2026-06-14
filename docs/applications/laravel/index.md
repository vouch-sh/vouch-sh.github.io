# Laravel (Socialite)

> Integrate Vouch OIDC authentication into a Laravel application using Socialite.

Source: https://vouch.sh/docs/applications/laravel/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[Laravel Socialite](https://laravel.com/docs/socialite) with the OIDC driver provides OAuth/OIDC integration. Key configuration:

- Install [`laravel/socialite`](https://laravel.com/docs/socialite), [`kovah/laravel-socialite-oidc`](https://github.com/Kovah/laravel-socialite-oidc), and [`socialiteproviders/manager`](https://github.com/SocialiteProviders/Manager)
- Use the `'oidc'` driver name with `->enablePKCE()` for PKCE support
- Register the Socialite service provider and event listener
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode the payload after base64url character replacement
- Callback URL: `/auth/callback`

## Example

**[web/laravel-socialite](https://github.com/vouch-sh/examples/tree/main/web/laravel-socialite)** — Complete working example with Socialite OIDC driver, PKCE, and hardware claim extraction.
