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
pip install authlib flask
```

Configure the OIDC client in your Flask application:

```python
from flask import Flask, redirect, url_for, session
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
app.secret_key = os.environ["FLASK_SECRET_KEY"]

oauth = OAuth(app)
vouch = oauth.register(
    name="vouch",
    client_id=os.environ["VOUCH_CLIENT_ID"],
    client_secret=os.environ["VOUCH_CLIENT_SECRET"],
    server_metadata_url="https://{{< instance-url >}}/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email"},
)


@app.route("/login")
def login():
    redirect_uri = url_for("callback", _external=True)
    return vouch.authorize_redirect(redirect_uri)


@app.route("/auth/callback")
def callback():
    token = vouch.authorize_access_token()
    userinfo = token["userinfo"]
    session["user"] = {
        "sub": userinfo["sub"],
        "name": userinfo.get("name"),
        "email": userinfo.get("email"),
    }
    return redirect("/")


@app.route("/")
def index():
    user = session.get("user")
    if user:
        return f"Hello, {user['name']}!"
    return '<a href="/login">Sign in with Vouch</a>'
```
