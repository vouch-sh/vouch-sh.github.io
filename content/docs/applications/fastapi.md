---
title: "FastAPI (Authlib)"
description: "Integrate Vouch OIDC authentication into a FastAPI application using Authlib."
weight: 7
params:
  category: "server"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

[Authlib](https://authlib.org/) provides OAuth and OpenID Connect client support for Starlette-based applications.

Install the required packages:

```bash
pip install authlib httpx fastapi uvicorn itsdangerous
```

Configure the OIDC client in your FastAPI application:

```python
import os
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse
from authlib.integrations.starlette_client import OAuth
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI()
app.add_middleware(SessionMiddleware, secret_key=os.environ["SESSION_SECRET"])

oauth = OAuth()
oauth.register(
    name="vouch",
    client_id=os.environ["VOUCH_CLIENT_ID"],
    client_secret=os.environ["VOUCH_CLIENT_SECRET"],
    server_metadata_url="{{< instance-url >}}/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email"},
)


@app.get("/login")
async def login(request: Request):
    redirect_uri = request.url_for("callback")
    return await oauth.vouch.authorize_redirect(request, redirect_uri)


@app.get("/auth/callback")
async def callback(request: Request):
    token = await oauth.vouch.authorize_access_token(request)
    userinfo = token["userinfo"]
    request.session["user"] = {
        "sub": userinfo["sub"],
        "name": userinfo.get("name"),
        "email": userinfo.get("email"),
    }
    return RedirectResponse(url="/")


@app.get("/")
async def index(request: Request):
    user = request.session.get("user")
    if user:
        return {"message": f"Hello, {user['name']}!"}
    return {"message": "Not authenticated. Visit /login to sign in."}
```
