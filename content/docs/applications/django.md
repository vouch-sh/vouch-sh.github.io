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
