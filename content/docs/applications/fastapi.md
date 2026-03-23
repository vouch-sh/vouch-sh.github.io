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
pip install authlib httpx fastapi uvicorn itsdangerous starlette
```

Configure the OIDC client in your FastAPI application:

```python
import os
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse, RedirectResponse
from starlette.middleware.sessions import SessionMiddleware
from authlib.integrations.starlette_client import OAuth

app = FastAPI()
app.add_middleware(
    SessionMiddleware,
    secret_key=os.environ.get('SECRET_KEY', 'dev-secret-change-in-production'),
)

oauth = OAuth()
oauth.register(
    name='vouch',
    client_id=os.environ.get('VOUCH_CLIENT_ID'),
    client_secret=os.environ.get('VOUCH_CLIENT_SECRET'),
    server_metadata_url=f"{os.environ.get('VOUCH_ISSUER', 'https://{{< instance-url >}}')}/.well-known/openid-configuration",
    client_kwargs={'scope': 'openid email'},
    code_challenge_method='S256',
)


@app.get('/login')
async def login(request: Request):
    redirect_uri = os.environ.get('VOUCH_REDIRECT_URI') or str(request.url_for('callback'))
    return await oauth.vouch.authorize_redirect(request, redirect_uri)


@app.get('/callback')
async def callback(request: Request):
    token = await oauth.vouch.authorize_access_token(request)
    userinfo = token.get('userinfo')
    request.session['user'] = {
        'email': userinfo.get('email'),
        'hardware_verified': userinfo.get('hardware_verified', False),
    }
    return RedirectResponse(url='/')


@app.get('/logout')
async def logout(request: Request):
    request.session.pop('user', None)
    return RedirectResponse(url='/')


@app.get('/', response_class=HTMLResponse)
async def home(request: Request):
    user = request.session.get('user')
    if user:
        verified = '<p><strong>Hardware Verified</strong></p>' if user.get('hardware_verified') else ''
        return f"<p>Signed in as {user['email']}</p>{verified}<a href='/logout'>Sign out</a>"
    return "<a href='/login'>Sign in with Vouch</a>"
```

The callback route is `/callback`. The `hardware_verified` claim is extracted from the userinfo response and stored in the session.

**Callback URL:** Register `http://localhost:3000/callback` as a redirect URI for your application.
