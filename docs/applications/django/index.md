# Django (django-allauth)

> Integrate Vouch OIDC authentication into a Django application using django-allauth.

Source: https://vouch.sh/docs/applications/django/
Last updated: 2026-04-21

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[django-allauth](https://docs.allauth.org/) provides OpenID Connect support for Django. Key configuration:

- Install [`django-allauth[openid_connect]`](https://docs.allauth.org/en/latest/socialaccount/providers/openid_connect.html)
- Set `oauth_pkce_enabled: True` and `fetch_userinfo: True` in `SOCIALACCOUNT_PROVIDERS`
- Use `server_url` (not `issuer`) in provider settings
- Callback URL: `/accounts/oidc/vouch/login/callback/`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode the payload to read it

## Example

**[web/django-allauth](https://github.com/vouch-sh/examples/tree/main/web/django-allauth)** — Complete working example with django-allauth OIDC provider, PKCE, and hardware claim extraction.
