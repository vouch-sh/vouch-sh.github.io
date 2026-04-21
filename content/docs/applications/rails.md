---
title: "Rails (OmniAuth)"
description: "Integrate Vouch OIDC authentication into a Ruby on Rails application using OmniAuth."
weight: 1
params:
  category: "server"
  language: "Ruby"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[OmniAuth OpenID Connect](https://github.com/omniauth/omniauth_openid_connect) provides a standard OIDC strategy for Rails applications. Key configuration:

- Install [`omniauth-openid-connect`](https://github.com/omniauth/omniauth_openid_connect) and [`omniauth-rails_csrf_protection`](https://github.com/cookpad/omniauth-rails_csrf_protection) gems
- Enable PKCE in the OmniAuth provider configuration (`pkce: true`)
- CSRF protection is required for the OmniAuth request phase
- Callback URL: `/auth/vouch/callback`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode the payload to read it

## Example

**[web/rails-omniauth](https://github.com/vouch-sh/examples/tree/main/web/rails-omniauth)** — Complete working example with OmniAuth OIDC strategy, PKCE, and hardware claim extraction.
