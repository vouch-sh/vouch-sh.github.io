---
title: "Credential Brokering Agents"
description: "Build CLI agents that authenticate with Vouch device flow and broker AWS, GitHub, and SSH credentials."
weight: 51
params:
  category: "agent"
  language: "Python"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

CLI agents and automation scripts can authenticate with Vouch using the [Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628) (no browser redirect needed), then use the Vouch credential brokering APIs to obtain temporary AWS credentials, GitHub tokens, or SSH certificates -- all tied to the user's hardware-backed identity.

## Dependencies

```bash
pip install requests boto3 cryptography
```

## Device Flow Authentication

Request a device code and poll for the user to complete authentication:

```python
import os
import sys
import time
import requests

VOUCH_ISSUER = os.environ.get('VOUCH_ISSUER', 'https://{{< instance-url >}}')
CLIENT_ID = os.environ.get('VOUCH_CLIENT_ID')

response = requests.post(
    f'{VOUCH_ISSUER}/oauth/device',
    data={
        'client_id': CLIENT_ID,
        'scope': 'openid email',
    },
)
response.raise_for_status()
device_data = response.json()

print(f"To sign in, visit: {device_data['verification_uri']}")
print(f"Enter code: {device_data['user_code']}")

interval = device_data.get('interval', 5)
while True:
    time.sleep(interval)

    token_response = requests.post(
        f'{VOUCH_ISSUER}/oauth/token',
        data={
            'grant_type':
                'urn:ietf:params:oauth:grant-type:device_code',
            'device_code': device_data['device_code'],
            'client_id': CLIENT_ID,
        },
    )

    if token_response.status_code == 200:
        tokens = token_response.json()
        access_token = tokens['access_token']
        break

    error = token_response.json().get('error')
    if error == 'authorization_pending':
        continue
    elif error == 'slow_down':
        interval += 5
    else:
        print(f'Authentication failed: {error}')
        sys.exit(1)
```

Once authenticated, use the access token to broker credentials from any of the services below.

---

## AWS Credentials

Get an AWS-specific ID token from Vouch, then exchange it for temporary AWS credentials via STS `AssumeRoleWithWebIdentity`. No ambient AWS credentials are needed for the STS call itself.

```python
import boto3
from botocore import UNSIGNED
from botocore.config import Config

AWS_ROLE_ARN = os.environ.get('AWS_ROLE_ARN')

aws_resp = requests.get(
    f'{VOUCH_ISSUER}/v1/credentials/aws/token',
    headers={'Authorization': f'Bearer {access_token}'},
)
aws_resp.raise_for_status()
aws_id_token = aws_resp.json()['id_token']

sts = boto3.client('sts', config=Config(signature_version=UNSIGNED))
assumed = sts.assume_role_with_web_identity(
    RoleArn=AWS_ROLE_ARN,
    RoleSessionName='vouch-agent',
    WebIdentityToken=aws_id_token,
)

creds = assumed['Credentials']
# Use creds['AccessKeyId'], creds['SecretAccessKey'],
# creds['SessionToken'] with any AWS SDK
```

The temporary credentials inherit the permissions of the assumed IAM role and expire automatically (typically 1 hour).

---

## GitHub Token

Request a GitHub installation token scoped to specific repositories. The token is short-lived and never written to disk:

```python
GITHUB_OWNER = os.environ.get('GITHUB_OWNER')

body = {}
if GITHUB_OWNER:
    body['owner'] = GITHUB_OWNER

gh_resp = requests.post(
    f'{VOUCH_ISSUER}/v1/credentials/github/token',
    headers={'Authorization': f'Bearer {access_token}'},
    json=body,
)
gh_resp.raise_for_status()
gh_data = gh_resp.json()

github_token = gh_data['token']
# Use github_token for GitHub API calls or git clone:
# https://x-access-token:{github_token}@github.com/owner/repo.git
```

You can scope the token to specific repositories by passing a `repositories` list in the request body.

---

## SSH Certificate

Generate an ephemeral key pair and have Vouch sign the public key as an SSH certificate:

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import (
    Ed25519PrivateKey,
)
from cryptography.hazmat.primitives import serialization

private_key = Ed25519PrivateKey.generate()
public_key_bytes = private_key.public_key().public_bytes(
    serialization.Encoding.OpenSSH,
    serialization.PublicFormat.OpenSSH,
)

ssh_resp = requests.post(
    f'{VOUCH_ISSUER}/v1/credentials/ssh',
    headers={'Authorization': f'Bearer {access_token}'},
    json={'public_key': public_key_bytes.decode('utf-8')},
)
ssh_resp.raise_for_status()
ssh_data = ssh_resp.json()

certificate = ssh_data['certificate']
principals = ssh_data.get('principals', [])
# Write certificate and private key to files for use with ssh
```

The certificate is tied to the user's Vouch identity and includes their email as a principal. Both the key pair and certificate are ephemeral.

---

## How It Works

1. The agent authenticates the user with the Vouch device flow -- the user visits a URL and confirms with their hardware key.
2. Vouch issues an access token backed by hardware attestation.
3. The agent calls Vouch credential brokering APIs with the access token to get service-specific credentials.
4. Each credential type is short-lived and scoped: AWS temporary credentials expire in ~1 hour, GitHub installation tokens in ~1 hour, and SSH certificates have a configurable validity window.
5. No long-lived secrets (AWS access keys, GitHub PATs, SSH private keys) need to be stored or distributed.
