---
title: "Laravel (Socialite)"
description: "Integrate Vouch OIDC authentication into a Laravel application using Socialite."
weight: 5
params:
  category: "server"
  language: "PHP"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

Laravel Socialite with the OIDC driver provides OAuth/OIDC integration with PKCE support via `->enablePKCE()`.

## Example

**[web/laravel-socialite](https://github.com/vouch-sh/examples/tree/main/web/laravel-socialite)** -- Complete working example with Socialite OIDC driver, service provider configuration, and hardware attestation claim access.
