---
title: "Flask (Authlib)"
description: "Integrate Vouch OIDC authentication into a Flask application using Authlib."
weight: 6
params:
  category: "server"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install [Authlib](https://authlib.org/):

```bash
pip install authlib flask requests
```

Configure the OIDC client in your Flask application:

```python
import os
from flask import Flask, redirect, url_for, session
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
app.secret_key = os.environ.get('SECRET_KEY', 'dev-secret-change-in-production')

VOUCH_ISSUER = os.environ.get('VOUCH_ISSUER', 'https://{{< instance-url >}}')

oauth = OAuth(app)
oauth.register(
    name='vouch',
    client_id=os.environ.get('VOUCH_CLIENT_ID'),
    client_secret=os.environ.get('VOUCH_CLIENT_SECRET'),
    server_metadata_url=f"{VOUCH_ISSUER}/.well-known/openid-configuration",
    client_kwargs={'scope': 'openid email'},
    code_challenge_method='S256',
)


@app.route('/login')
def login():
    redirect_uri = os.environ.get('VOUCH_REDIRECT_URI') or url_for('callback', _external=True)
    return oauth.vouch.authorize_redirect(redirect_uri)


@app.route('/callback')
def callback():
    token = oauth.vouch.authorize_access_token()
    userinfo = token.get('userinfo')
    session['user'] = {
        'email': userinfo.get('email'),
        'hardware_verified': userinfo.get('hardware_verified', False),
        'hardware_aaguid': userinfo.get('hardware_aaguid'),
    }
    return redirect('/')


@app.route('/logout')
def logout():
    session.pop('user', None)
    return redirect('/')


@app.route('/')
def home():
    user = session.get('user')
    if user:
        verified = '<p><strong>Hardware Verified</strong></p>' if user.get('hardware_verified') else ''
        return f"<p>Signed in as {user['email']}</p>{verified}<a href='/logout'>Sign out</a>"
    return "<a href='/login'>Sign in with Vouch</a>"
```

The callback route is `/callback`. The `hardware_verified` and `hardware_aaguid` claims are extracted from the userinfo response.

**Callback URL:** Register `http://localhost:3000/callback` as a redirect URI for your application.

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` as an extra parameter in the authorization redirect:

```python
import json

@app.route('/login')
def login():
    redirect_uri = url_for('callback', _external=True)
    authorization_details = json.dumps([
        {"type": "account_access", "actions": ["read", "transfer"]}
    ])
    return oauth.vouch.authorize_redirect(
        redirect_uri,
        authorization_details=authorization_details,
    )
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
