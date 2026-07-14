# Bridge Human Identity with SPIFFE Workload Identity

> Connect hardware-verified developer identity to SPIFFE workload identity — federate Vouch OIDC tokens with SPIRE for zero-trust infrastructure.

Source: https://vouch.sh/docs/spiffe/
Last updated: 2026-07-04

---
[SPIFFE](https://spiffe.io/) gives every workload a cryptographic identity, but it does not address *who deployed the workload* or *who authorized the action*. Because Vouch is a standards-compliant OIDC provider, you can configure [SPIRE](https://spiffe.io/docs/latest/spire-about/spire-concepts/) (the SPIFFE reference implementation) to trust Vouch-issued tokens -- bridging human and workload identity in a single architecture.

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
3. **Both coexist in the same infrastructure** — Humans authenticate with `vouch login` &#43; YubiKey; services authenticate with SPIFFE SVIDs. Downstream systems (Kubernetes, AWS, databases) can accept both.

---

## Prerequisites

- **Vouch CLI installed and enrolled** — Complete the [Getting Started](/docs/getting-started/) guide.
- **SPIRE Server and Agent deployed** — See the [SPIRE Getting Started guide](https://spiffe.io/docs/latest/try/getting-started-linux/) or the [Kubernetes quickstart](https://spiffe.io/docs/latest/try/getting-started-k8s/).
- **`spire-server` and `spire-agent` CLI tools** available on your path.

---

## Configure SPIRE to trust Vouch tokens

This is the integration itself: SPIRE validates Vouch OIDC tokens so that human identity can inform workload registration and authorization decisions.

### Register Vouch as a federated trust domain

Add a federation block to your SPIRE Server config so SPIRE automatically fetches and refreshes Vouch&#39;s signing keys:

```hcl
# spire-server.conf
server {
  trust_domain = &#34;example.org&#34;
  # ...
}

plugins {
  # Existing plugins...

  KeyManager &#34;disk&#34; {
    plugin_data {
      keys_path = &#34;/opt/spire/data/keys.json&#34;
    }
  }
}

# Federation with Vouch OIDC provider
federation {
  bundle_endpoint {
    address = &#34;0.0.0.0&#34;
    port = 8443
  }

  federates_with &#34;vouch.sh&#34; {
    bundle_endpoint_url = &#34;https://us.vouch.sh&#34;
    bundle_endpoint_profile &#34;https_web&#34; {}
  }
}
```

The `https_web` profile tells SPIRE to authenticate the endpoint using its public TLS certificate (standard web PKI). SPIRE fetches the JWKS from `https://us.vouch.sh/oauth/jwks` and automatically refreshes it as keys rotate.

**For air-gapped environments**, fetch the keys manually and import them:

```bash
curl -s https://us.vouch.sh/oauth/jwks -o vouch-jwks.json

spire-server bundle set \
  -id spiffe://vouch.sh \
  -format jwks \
  -path vouch-jwks.json
```

&gt; **Note:** The manual approach requires re-running these commands whenever Vouch rotates its signing keys. Prefer the automatic federation config above unless your SPIRE Server cannot reach `https://us.vouch.sh`.

### Create workload registration entries with deployer identity

With federation in place, you can register workloads and tag them with the Vouch-authenticated deployer&#39;s identity. This creates an audit trail from the human who deployed a workload to the SPIFFE identity the workload runs with:

```bash
# Register a backend API workload
# The &#34;deployer&#34; selector records who authorized this registration
spire-server entry create \
  -spiffeID spiffe://example.org/backend-api \
  -parentID spiffe://example.org/spire-agent \
  -selector k8s:ns:production \
  -selector k8s:sa:backend-api \
  -metadata &#34;deployer:alice@example.com&#34;
```

### Validate Vouch tokens in a custom attestor

For advanced use cases, you can write a [custom workload attestor plugin](https://spiffe.io/docs/latest/extending/extending/) that validates a Vouch OIDC token presented by a workload during attestation. This lets workloads bootstrap their SPIFFE identity using a short-lived Vouch token:

```bash
# A workload requests a Vouch token with a SPIFFE-specific audience
vouch credential k8s --audience spiffe://example.org

# The custom attestor validates the token against Vouch&#39;s JWKS
# and maps the `sub` claim to a SPIFFE ID
```

| Vouch token claim | SPIRE mapping |
|---|---|
| `iss` | Must match `https://us.vouch.sh` |
| `sub` | Maps to deployer identity metadata |
| `aud` | Must match the trust domain or a configured audience |
| `exp` | Token must not be expired |
| `amr` | Can require `[&#34;hwk&#34;, &#34;pin&#34;]` for hardware attestation |

---

## Patterns

Once SPIRE trusts Vouch tokens, the common architectures are standard SPIFFE/SPIRE deployments with Vouch supplying the human identity layer. The SPIRE mechanics are covered by the [SPIRE documentation](https://spiffe.io/docs/latest/); what Vouch adds to each:

- **Kubernetes with human &#43; service identity** -- Developers reach the API server with Vouch OIDC (see the [Kubernetes guide](/docs/kubernetes/)); pods authenticate to each other with X.509-SVIDs issued by SPIRE. The layers are complementary: Vouch covers human-to-cluster, SPIRE covers pod-to-pod mTLS.
- **Multi-cloud service mesh** -- SPIRE servers in each cloud federate via bundle exchange so services authenticate across trust domains, while operators use the same `vouch login` session for access to every environment.
- **Zero-trust CI/CD with human approval** -- a self-hosted runner attests its own identity via SPIFFE SVID, and the deployment additionally requires a Vouch OIDC token minted by a human with a YubiKey -- see [CI/CD approval gates](/docs/cicd/) for the Vouch half of that pattern.

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

- Verify the SPIRE Server can make outbound HTTPS requests to `https://us.vouch.sh/.well-known/openid-configuration`.
- Check network policies, firewall rules, and DNS resolution from the SPIRE Server pod or host.
- Test connectivity: `curl -s https://us.vouch.sh/.well-known/openid-configuration | jq .`

### SVID validation failures

- Ensure the trust bundles are exchanged correctly between federated SPIRE servers. Check with `spire-server bundle show`.
- Verify the workload registration entries match the actual pod selectors: `spire-server entry show`.
- Check that the SPIRE Agent is running on the node where the workload is scheduled.

### Token audience mismatch

- The `aud` claim in Vouch tokens must match what SPIRE expects. When using `vouch credential k8s`, the default audience is `kubernetes`. For SPIFFE integration, specify a custom audience: `vouch credential k8s --audience spiffe://example.org`.
- Verify with: `vouch credential k8s --audience spiffe://example.org | jq -r &#39;.status.token&#39; | step crypto jwt inspect --insecure | jq &#39;.payload.aud&#39;`

### Clock skew causing JWT validation errors

- SPIRE validates `exp` and `nbf` claims in Vouch tokens. Ensure clocks are synchronized across all nodes using NTP.
- Vouch tokens are short-lived — even a few minutes of clock skew can cause validation failures.
- Check the SPIRE Server logs: `kubectl logs -n spire-system deployment/spire-server`

### &#34;No identity issued&#34; from Workload API

- Confirm the SPIRE Agent is running: `spire-agent healthcheck -socketPath /run/spire/sockets/agent.sock`.
- Verify that a registration entry exists for the workload: `spire-server entry show -selector k8s:ns:YOUR_NAMESPACE -selector k8s:sa:YOUR_SERVICE_ACCOUNT`.
- Check that the workload&#39;s service account and namespace match the registered selectors exactly.
