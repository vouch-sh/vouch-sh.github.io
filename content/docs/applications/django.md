---
title: "Django (django-allauth)"
description: "Integrate Vouch OIDC authentication into a Django application using django-allauth."
weight: 2
params:
  category: "server"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install [django-allauth](https://django-allauth.readthedocs.io/) with OpenID Connect support:

```bash
pip install django-allauth[openid_connect]
```

Add the provider to your `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.openid_connect",
]

SOCIALACCOUNT_PROVIDERS = {
    "openid_connect": {
        "APPS": [
            {
                "provider_id": "vouch",
                "name": "Vouch",
                "client_id": os.environ["VOUCH_CLIENT_ID"],
                "secret": os.environ["VOUCH_CLIENT_SECRET"],
                "settings": {
                    "server_url": "https://{{< instance-url >}}",
                    "oauth_pkce_enabled": True,
                    "fetch_userinfo": True,
                },
            },
        ],
    },
}
```

Add the allauth URLs to your `urls.py`:

```python
from django.urls import path, include

urlpatterns = [
    # ...
    path("accounts/", include("allauth.urls")),
]
```

### Rich Authorization Requests

django-allauth does not natively support `authorization_details`. To include Rich Authorization Requests, you can pass extra parameters by customizing the provider's login URL with additional query parameters, or by subclassing the provider adapter to inject `authorization_details` into the authorization request.

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format and how to include them in PAR requests.
