---
title: "Bridge Human Identity with SPIFFE Workload Identity"
linkTitle: "SPIFFE"
description: "Connect hardware-verified developer identity to SPIFFE workload identity — federate Vouch OIDC tokens with SPIRE for zero-trust infrastructure."
weight: 5
subtitle: "Federate Vouch with SPIRE to bridge human and workload identity"
sitemap:
  priority: 0.8
params:
  docsGroup: infra
---

Most organizations end up with two identity systems: one for humans (SSO, hardware keys) and one for workloads (service accounts, static secrets). [SPIFFE](https://spiffe.io/) (Secure Production Identity Framework for Everyone) solves the workload side — it gives every service a cryptographic identity (an SVID) that is short-lived, automatically rotated, and platform-agnostic. But SPIFFE does not address *who deployed the workload* or *who authorized the action*.

Vouch bridges that gap. Because Vouch is a standards-compliant OIDC provider, you can configure [SPIRE](https://spiffe.io/docs/latest/spire-about/spire-concepts/) (the SPIFFE reference implementation) to trust Vouch-issued tokens. This gives you a unified identity architecture where every credential — human or machine — is short-lived, cryptographically verified, and traceable.

## How it works

Vouch and SPIRE operate at different layers of the identity stack:

```
Human layer (Vouch)                    Workload layer (SPIFFE/SPIRE)
─────────────────────                  ─────────────────────────────
YubiKey tap                            Workload attestation
  → Vouch OIDC ID token                 → X.509-SVID or JWT-SVID
    → kubectl, AWS, SSH, etc.              → mTLS, service-to-service auth
```

The integration points:

1. **SPIRE trusts Vouch as an OIDC issuer** — SPIRE validates Vouch tokens using the `/.well-known/openid-configuration` and JWKS endpoints to make authorization decisions based on hardware-verified human identity.
2. **Workloads get SPIFFE SVIDs** — SPIRE issues short-lived X.509 or JWT credentials to workloads via the Workload API, independent of any human session.
3. **Both coexist in the same infrastructure** — Humans authenticate with `vouch login` + YubiKey; services authenticate with SPIFFE SVIDs. Downstream systems (Kubernetes, AWS, databases) can accept both.

---

## Prerequisites

- **Vouch CLI installed and enrolled** — Complete the [Getting Started](/docs/getting-started/) guide.
- **SPIRE Server and Agent deployed** — See the [SPIRE Getting Started guide](https://spiffe.io/docs/latest/try/getting-started-linux/) or the [Kubernetes quickstart](https://spiffe.io/docs/latest/try/getting-started-k8s/).
- **`spire-server` and `spire-agent` CLI tools** available on your path.

---

## Example 1: OIDC federation — Configure SPIRE to trust Vouch tokens

This is the foundational integration. You configure SPIRE to validate Vouch OIDC tokens so that human identity can inform workload registration and authorization decisions.

### Register Vouch as a federated trust domain

Add a federation block to your SPIRE Server config so SPIRE automatically fetches and refreshes Vouch's signing keys:

```hcl
# spire-server.conf
server {
  trust_domain = "example.org"
  # ...
}

plugins {
  # Existing plugins...

  KeyManager "disk" {
    plugin_data {
      keys_path = "/opt/spire/data/keys.json"
    }
  }
}

# Federation with Vouch OIDC provider
federation {
  bundle_endpoint {
    address = "0.0.0.0"
    port = 8443
  }

  federates_with "vouch.sh" {
    bundle_endpoint_url = "https://{{< instance-url >}}"
    bundle_endpoint_profile "https_web" {}
  }
}
```

The `https_web` profile tells SPIRE to authenticate the endpoint using its public TLS certificate (standard web PKI). SPIRE fetches the JWKS from `https://{{< instance-url >}}/.well-known/jwks.json` and automatically refreshes it as keys rotate.

**For air-gapped environments**, fetch the keys manually and import them:

```bash
curl -s https://{{< instance-url >}}/.well-known/jwks.json -o vouch-jwks.json

spire-server bundle set \
  -id spiffe://vouch.sh \
  -format jwks \
  -path vouch-jwks.json
```

> **Note:** The manual approach requires re-running these commands whenever Vouch rotates its signing keys. Prefer the automatic federation config above unless your SPIRE Server cannot reach `https://{{< instance-url >}}`.

### Create workload registration entries with deployer identity

With federation in place, you can register workloads and tag them with the Vouch-authenticated deployer's identity. This creates an audit trail from the human who deployed a workload to the SPIFFE identity the workload runs with:

```bash
# Register a backend API workload
# The "deployer" selector records who authorized this registration
spire-server entry create \
  -spiffeID spiffe://example.org/backend-api \
  -parentID spiffe://example.org/spire-agent \
  -selector k8s:ns:production \
  -selector k8s:sa:backend-api \
  -metadata "deployer:alice@example.com"
```

### Validate Vouch tokens in a custom attestor

For advanced use cases, you can write a [custom workload attestor plugin](https://spiffe.io/docs/latest/extending/extending/) that validates a Vouch OIDC token presented by a workload during attestation. This lets workloads bootstrap their SPIFFE identity using a short-lived Vouch token:

```bash
# A workload requests a Vouch token with a SPIFFE-specific audience
vouch credential k8s --audience spiffe://example.org

# The custom attestor validates the token against Vouch's JWKS
# and maps the `sub` claim to a SPIFFE ID
```

| Vouch token claim | SPIRE mapping |
|---|---|
| `iss` | Must match `https://{{< instance-url >}}` |
| `sub` | Maps to deployer identity metadata |
| `aud` | Must match the trust domain or a configured audience |
| `exp` | Token must not be expired |
| `amr` | Can require `["hwk", "pin"]` for hardware attestation |

---

## Example 2: Kubernetes with Vouch (human) + SPIFFE (service) identity

This is the most common deployment: a Kubernetes cluster where developers use Vouch for `kubectl` access and services use SPIFFE SVIDs for mutual authentication.

### Architecture

```
Developer workstation              Kubernetes cluster
──────────────────────             ──────────────────
vouch login (YubiKey)              SPIRE Agent (DaemonSet)
  → vouch credential k8s             → Workload API (Unix socket)
    → kubectl (OIDC token)              → X.509-SVID per pod
                                          → mTLS between services

API Server validates both:
  - Vouch OIDC tokens (human access)
  - ServiceAccount tokens (SPIRE-managed workload access)
```

### Configure kubectl with Vouch OIDC

Follow the [Kubernetes guide](/docs/kubernetes/) to set up Vouch OIDC authentication for human access:

```bash
vouch setup k8s \
  --cluster production \
  --server https://k8s.example.com:6443 \
  --certificate-authority /path/to/ca.pem
```

### Deploy SPIRE on Kubernetes

Deploy SPIRE Server and Agent using the official Helm chart:

```bash
helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/
helm repo update

helm install spire spiffe/spire \
  --namespace spire-system \
  --create-namespace \
  --set global.spire.trustDomain=example.org \
  --set spire-server.controllerManager.enabled=true
```

### Register workloads with SPIFFE IDs

Create registration entries that map Kubernetes workloads to SPIFFE identities:

```bash
# Backend API — gets an X.509-SVID for mTLS
spire-server entry create \
  -spiffeID spiffe://example.org/ns/production/sa/backend-api \
  -parentID spiffe://example.org/spire-agent \
  -selector k8s:ns:production \
  -selector k8s:sa:backend-api

# Frontend service — gets an X.509-SVID to call the backend
spire-server entry create \
  -spiffeID spiffe://example.org/ns/production/sa/frontend \
  -parentID spiffe://example.org/spire-agent \
  -selector k8s:ns:production \
  -selector k8s:sa:frontend
```

### Application-level mTLS with SPIFFE

Services retrieve their SVIDs from the SPIRE Workload API and use them for mTLS. Here is a Go service that accepts connections only from peers with valid SPIFFE identities:

```go
package main

import (
    "context"
    "log"
    "net/http"

    "github.com/spiffe/go-spiffe/v2/spiffeid"
    "github.com/spiffe/go-spiffe/v2/spiffetls"
    "github.com/spiffe/go-spiffe/v2/spiffetls/tlsconfig"
)

func main() {
    // Only accept connections from the frontend service
    allowedCaller := spiffeid.RequireFromString(
        "spiffe://example.org/ns/production/sa/frontend",
    )

    listener, err := spiffetls.Listen(
        context.Background(),
        "tcp", ":8443",
        tlsconfig.AuthorizeID(allowedCaller),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer listener.Close()

    http.Serve(listener, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("authenticated via SPIFFE mTLS"))
    }))
}
```

And the frontend calling the backend over mTLS:

```go
package main

import (
    "context"
    "io"
    "log"
    "net/http"

    "github.com/spiffe/go-spiffe/v2/spiffeid"
    "github.com/spiffe/go-spiffe/v2/spiffetls/tlsconfig"
    "github.com/spiffe/go-spiffe/v2/workloadapi"
)

func main() {
    ctx := context.Background()

    // Connect to the SPIRE Workload API
    source, err := workloadapi.NewX509Source(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer source.Close()

    // Create an HTTP client that presents our SVID
    // and verifies the backend's SPIFFE ID
    backendID := spiffeid.RequireFromString(
        "spiffe://example.org/ns/production/sa/backend-api",
    )

    tlsConfig := tlsconfig.MTLSClientConfig(source, source, tlsconfig.AuthorizeID(backendID))
    client := &http.Client{
        Transport: &http.Transport{TLSClientConfig: tlsConfig},
    }

    resp, err := client.Get("https://backend-api.production.svc:8443/data")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    log.Printf("Backend response: %s", body)
}
```

### RBAC: both identity types on one cluster

Configure Kubernetes RBAC to handle both Vouch-authenticated humans and SPIFFE-authenticated services:

```yaml
# Human access — developers authenticate via Vouch OIDC
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vouch-developers
subjects:
  - kind: User
    name: alice@example.com   # From Vouch OIDC `sub` claim
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
---
# Service access — workloads authenticate via ServiceAccount tokens
# SPIRE manages the workload identity layer (mTLS between pods)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-api-access
  namespace: production
subjects:
  - kind: ServiceAccount
    name: backend-api
    namespace: production
roleRef:
  kind: Role
  name: backend-api-role
  apiGroup: rbac.authorization.k8s.io
```

> **Note:** Vouch handles the human → API server authentication path via OIDC. SPIRE handles the pod → pod authentication path via X.509-SVIDs. The two systems are complementary — they do not replace each other.

---

## Example 3: Multi-cloud service mesh with SPIFFE federation

For organizations running workloads across multiple clouds and on-premises, SPIFFE federation lets services authenticate across trust domain boundaries. Vouch provides the consistent human identity layer that operates across all environments.

### Architecture

```
AWS (trust domain: aws.example.org)       GCP (trust domain: gcp.example.org)
──────────────────────────────────         ──────────────────────────────────
SPIRE Server (AWS)                         SPIRE Server (GCP)
  ↕ federation bundle exchange               ↕ federation bundle exchange
SPIRE Agents                               SPIRE Agents
  → X.509-SVIDs for AWS workloads            → X.509-SVIDs for GCP workloads

                    ↕ cross-cloud mTLS ↕

Human operators authenticate with Vouch across both environments:
  vouch login → vouch credential aws   (AWS access)
  vouch login → kubectl                (GCP GKE access)
```

### Configure SPIFFE federation between clouds

Each SPIRE server exposes a bundle endpoint and federates with the other:

```hcl
# AWS SPIRE server config
server {
  trust_domain = "aws.example.org"
}

federation {
  bundle_endpoint {
    address = "0.0.0.0"
    port = 8443
  }

  federates_with "gcp.example.org" {
    bundle_endpoint_url = "https://spire-server.gcp.example.org:8443"
    bundle_endpoint_profile "https_spiffe" {
      endpoint_spiffe_id = "spiffe://gcp.example.org/spire-server"
    }
  }
}
```

```hcl
# GCP SPIRE server config
server {
  trust_domain = "gcp.example.org"
}

federation {
  bundle_endpoint {
    address = "0.0.0.0"
    port = 8443
  }

  federates_with "aws.example.org" {
    bundle_endpoint_url = "https://spire-server.aws.example.org:8443"
    bundle_endpoint_profile "https_spiffe" {
      endpoint_spiffe_id = "spiffe://aws.example.org/spire-server"
    }
  }
}
```

### Register cross-cloud workloads

Allow a payment service in AWS to call a fraud detection service in GCP:

```bash
# On AWS SPIRE server — register the payment service
spire-server entry create \
  -spiffeID spiffe://aws.example.org/payment-service \
  -parentID spiffe://aws.example.org/spire-agent \
  -selector k8s:ns:payments \
  -selector k8s:sa:payment-service \
  -federatesWith spiffe://gcp.example.org

# On GCP SPIRE server — register the fraud detection service
# and allow calls from the AWS payment service
spire-server entry create \
  -spiffeID spiffe://gcp.example.org/fraud-detection \
  -parentID spiffe://gcp.example.org/spire-agent \
  -selector k8s:ns:ml-services \
  -selector k8s:sa:fraud-detection \
  -federatesWith spiffe://aws.example.org
```

### Human access across clouds with Vouch

While services authenticate to each other via SPIFFE federation, human operators use Vouch for access to both environments:

```bash
# Authenticate once with YubiKey
vouch login

# Access AWS resources
vouch credential aws --role arn:aws:iam::111111111111:role/platform-engineer
aws eks update-kubeconfig --name production-aws

# Access GCP Kubernetes (via Vouch OIDC)
vouch setup k8s \
  --cluster production-gcp \
  --server https://gke.gcp.example.org:6443
kubectl --context production-gcp-vouch get pods
```

The result: every credential in the system is short-lived and traceable — human access via Vouch OIDC tokens, service-to-service via SPIFFE SVIDs, cross-cloud via SPIFFE federation.

---

## Example 4: Zero-trust CI/CD with human approval gates

Combine SPIFFE workload identity with Vouch human approval to build a CI/CD pipeline where *both* the runner's identity and a human authorization are cryptographically verified before a production deployment can proceed.

### Architecture

```
Developer                    CI Runner                      Production
──────────                   ─────────                      ──────────
vouch login (YubiKey)        SPIFFE SVID (workload)         AWS / Kubernetes
  → OIDC token                 → attests runner identity      → requires both:
    → passed to CI job           → proves legitimate runner       1. valid SVID
                                                                  2. valid Vouch token
```

### GitHub Actions workflow

```yaml
name: Production Deploy

on:
  workflow_dispatch:
    inputs:
      vouch_token:
        description: 'Vouch OIDC token from deployer'
        required: true
        type: string

jobs:
  deploy:
    runs-on: self-hosted  # Runner with SPIRE Agent installed
    steps:
      - name: Verify deployer identity
        run: |
          # Validate the Vouch OIDC token
          # The token proves a human with a YubiKey authorized this deploy
          DEPLOYER=$(echo "${{ inputs.vouch_token }}" | \
            step crypto jwt inspect --insecure | \
            jq -r '.payload.sub')
          echo "Deployment authorized by: ${DEPLOYER}"
          echo "DEPLOYER=${DEPLOYER}" >> $GITHUB_ENV

      - name: Obtain SPIFFE identity
        run: |
          # The CI runner fetches its SVID from the local SPIRE Agent
          # This proves the runner is a legitimate CI workload
          /opt/spire/bin/spire-agent api fetch jwt \
            -audience sts.amazonaws.com \
            -socketPath /run/spire/sockets/agent.sock \
            -write /tmp/svid.jwt
          echo "Runner SPIFFE ID: $(cat /tmp/svid.jwt | \
            step crypto jwt inspect --insecure | \
            jq -r '.payload.sub')"

      - name: Deploy to production
        env:
          VOUCH_TOKEN: ${{ inputs.vouch_token }}
        run: |
          # Exchange the Vouch token for AWS credentials
          # The IAM role trust policy requires BOTH:
          #   1. A valid Vouch token (human authorization)
          #   2. The call to originate from an attested CI runner
          AWS_CREDS=$(aws sts assume-role-with-web-identity \
            --role-arn arn:aws:iam::ACCOUNT_ID:role/production-deployer \
            --role-session-name "deploy-${DEPLOYER}" \
            --web-identity-token "${VOUCH_TOKEN}")

          export AWS_ACCESS_KEY_ID=$(echo $AWS_CREDS | jq -r '.Credentials.AccessKeyId')
          export AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDS | jq -r '.Credentials.SecretAccessKey')
          export AWS_SESSION_TOKEN=$(echo $AWS_CREDS | jq -r '.Credentials.SessionToken')

          # Deploy with credentials tied to a specific human + runner
          kubectl apply -f k8s/production/
```

### Trigger the deployment from a developer workstation

```bash
# Authenticate with YubiKey
vouch login

# Get a short-lived OIDC token for the CI/CD audience
DEPLOY_TOKEN=$(vouch credential aws \
  --role arn:aws:iam::ACCOUNT_ID:role/production-deployer \
  --format token)

# Trigger the deployment, passing the human authorization token
gh workflow run "Production Deploy" \
  --field vouch_token="${DEPLOY_TOKEN}"
```

### IAM trust policy requiring both identities

The IAM role trust policy ensures that *both* the runner's SPIFFE identity and the deployer's Vouch identity are present:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/{{< instance-url >}}"
      },
      "Action": ["sts:AssumeRoleWithWebIdentity", "sts:TagSession"],
      "Condition": {
        "StringEquals": {
          "{{< instance-url >}}:aud": "{{< instance-url >}}",
          "{{< instance-url >}}:sub": [
            "alice@example.com",
            "bob@example.com"
          ]
        },
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

> **Note:** The `IpAddress` condition restricts token exchange to your CI runner network. Combined with the SPIFFE SVID attesting the runner's workload identity and the Vouch token attesting a human deployer, this creates a three-factor authorization chain: legitimate runner + authorized human + trusted network.

---

## SPIFFE concepts reference

| Concept | Description |
|---|---|
| **SPIFFE ID** | A URI that uniquely identifies a workload: `spiffe://trust-domain/path` |
| **SVID** | SPIFFE Verifiable Identity Document — an X.509 certificate or JWT that encodes a SPIFFE ID |
| **Trust domain** | The root of trust for a SPIFFE deployment (e.g., `example.org`) |
| **Workload API** | Local API (Unix socket) that workloads call to get their SVIDs and trust bundles |
| **SPIRE Server** | Central component that manages identities and issues SVIDs |
| **SPIRE Agent** | Per-node component that attests workloads and exposes the Workload API |
| **Federation** | Cross-trust-domain authentication via bundle exchange |

For full details, see the [SPIFFE specification](https://github.com/spiffe/spiffe/tree/main/standards) and [SPIRE documentation](https://spiffe.io/docs/latest/).

---

## Troubleshooting

### SPIRE cannot reach the Vouch OIDC discovery endpoint

- Verify the SPIRE Server can make outbound HTTPS requests to `https://{{< instance-url >}}/.well-known/openid-configuration`.
- Check network policies, firewall rules, and DNS resolution from the SPIRE Server pod or host.
- Test connectivity: `curl -s https://{{< instance-url >}}/.well-known/openid-configuration | jq .`

### SVID validation failures

- Ensure the trust bundles are exchanged correctly between federated SPIRE servers. Check with `spire-server bundle show`.
- Verify the workload registration entries match the actual pod selectors: `spire-server entry show`.
- Check that the SPIRE Agent is running on the node where the workload is scheduled.

### Token audience mismatch

- The `aud` claim in Vouch tokens must match what SPIRE expects. When using `vouch credential k8s`, the default audience is `kubernetes`. For SPIFFE integration, specify a custom audience: `vouch credential k8s --audience spiffe://example.org`.
- Verify with: `vouch credential k8s --audience spiffe://example.org | jq -r '.status.token' | step crypto jwt inspect --insecure | jq '.payload.aud'`

### Clock skew causing JWT validation errors

- SPIRE validates `exp` and `nbf` claims in Vouch tokens. Ensure clocks are synchronized across all nodes using NTP.
- Vouch tokens are short-lived — even a few minutes of clock skew can cause validation failures.
- Check the SPIRE Server logs: `kubectl logs -n spire-system deployment/spire-server`

### "No identity issued" from Workload API

- Confirm the SPIRE Agent is running: `spire-agent healthcheck -socketPath /run/spire/sockets/agent.sock`.
- Verify that a registration entry exists for the workload: `spire-server entry show -selector k8s:ns:YOUR_NAMESPACE -selector k8s:sa:YOUR_SERVICE_ACCOUNT`.
- Check that the workload's service account and namespace match the registered selectors exactly.
