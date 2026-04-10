---
title: "Authenticate to Kubernetes with OIDC"
linkTitle: "Kubernetes"
description: "Access any Kubernetes cluster using OIDC tokens from Vouch — no cloud-specific plugins required."
weight: 6
subtitle: "Authenticate to Kubernetes clusters using OIDC identity tokens"
params:
  docsGroup: infra
---

Static kubeconfig tokens and shared service account credentials are hard to audit and easy to leak. Kubernetes supports [OpenID Connect (OIDC)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) as a first-class authentication method, letting you replace static credentials with short-lived identity tokens.

Vouch acts as an OIDC provider for your Kubernetes clusters. After a YubiKey tap, the CLI fetches an OIDC ID token and presents it to the API server — no cloud-specific plugins, no static tokens, and every `kubectl` command traces back to a hardware-verified identity.

> **Using Amazon EKS?** EKS has its own token mechanism based on AWS IAM. See [Amazon EKS](/docs/eks/) for EKS-specific setup.

## How it works

The authentication flow chains three components:

```
vouch login --> vouch credential k8s --> kubectl
```

1. **`vouch login`** — The developer authenticates with their YubiKey and receives an OIDC session from the Vouch server.
2. **`vouch credential k8s`** — The CLI requests an OIDC ID token with the audience set to match the cluster's `--oidc-client-id` (default: `kubernetes`). It outputs a Kubernetes [`ExecCredential`](https://kubernetes.io/docs/reference/config-api/client-authentication.v1/) JSON object containing the token and its expiration.
3. **`kubectl`** — The Kubernetes client sends the token to the API server. The API server validates it against the Vouch server's OIDC discovery endpoint (`/.well-known/openid-configuration`) and applies RBAC rules based on the token claims.

Because every step uses short-lived credentials, there are no static kubeconfig tokens to manage or rotate.

---

## Prerequisites

Before setting up Kubernetes OIDC authentication with Vouch, ensure you have:

- **Vouch CLI installed and enrolled** — Complete the [Getting Started](/docs/getting-started/) guide.
- **kubectl** installed (`kubectl version --client`).
- **A Kubernetes cluster** with OIDC authentication configured on the API server (see below).

---

## Configuring the API server

The Kubernetes API server must be configured to trust the Vouch server as an OIDC provider. Add the following flags to your `kube-apiserver` configuration:

```
--oidc-issuer-url=https://us.vouch.sh
--oidc-client-id=kubernetes
--oidc-username-claim=sub
```

| Flag | Description |
|---|---|
| `--oidc-issuer-url` | The Vouch server URL. The API server fetches `/.well-known/openid-configuration` from this URL to discover the JWKS endpoint. |
| `--oidc-client-id` | The expected `aud` (audience) claim in the ID token. Must match the `--audience` flag used with `vouch credential k8s` (default: `kubernetes`). |
| `--oidc-username-claim` | The token claim to use as the Kubernetes username. Set to `sub` to use the developer's email address. |
| `--oidc-groups-claim` | (Optional) The token claim to use for group membership in RBAC rules. |
| `--oidc-username-prefix` | (Optional) Prefix added to usernames to avoid collisions with other authentication methods (e.g., `vouch:`). |

How you set these flags depends on your Kubernetes distribution:

- **kubeadm**: Add to `ClusterConfiguration.apiServer.extraArgs` in the kubeadm config.
- **k3s**: Pass as arguments to `k3s server`, e.g., `k3s server --kube-apiserver-arg="oidc-issuer-url=https://us.vouch.sh"`.
- **GKE**: Use [GKE Identity Service](https://cloud.google.com/kubernetes-engine/docs/how-to/oidc) to configure OIDC.
- **AKS**: Use [AKS OIDC configuration](https://learn.microsoft.com/en-us/azure/aks/use-oidc-issuer).

> **Important:** The API server must be able to reach `https://us.vouch.sh/.well-known/openid-configuration` and the JWKS endpoint to validate tokens. Ensure your network policies allow outbound HTTPS from the control plane to the Vouch server.

---

## Setup

Configure `kubectl` to use your Vouch-backed OIDC credentials:

```bash
vouch setup k8s \
  --cluster my-cluster \
  --server https://k8s.example.com:6443 \
  --certificate-authority /path/to/ca.pem
```

Required flags:

| Flag | Description |
|---|---|
| `--cluster` | A name for this cluster (used in kubeconfig entries and as a cache key) |
| `--server` | The Kubernetes API server URL |

Optional flags:

| Flag | Description |
|---|---|
| `--certificate-authority` | Path to the cluster's CA certificate file (PEM format). The certificate is base64-encoded and embedded in the kubeconfig. |
| `--audience` | OIDC audience — must match `--oidc-client-id` on the API server (default: `kubernetes`) |
| `--kubeconfig` | Path to kubeconfig file (defaults to `~/.kube/config`) |

This command writes or updates your kubeconfig with:

- A **cluster** entry named after the `--cluster` value with the server URL and CA certificate.
- A **user** entry named `vouch-k8s-{cluster}` configured to call `vouch credential k8s` as an exec-based credential plugin.
- A **context** named `{cluster}-vouch` linking the cluster and user entries.

### Verify the kubeconfig

Check that the context is set and working:

```bash
kubectl config use-context my-cluster-vouch
kubectl get pods
```

---

## Usage

With everything configured, daily usage is straightforward:

```bash
# Start your day
vouch login

# Switch to your Vouch K8s context
kubectl config use-context my-cluster-vouch

# Use kubectl as normal
kubectl get pods
kubectl get namespaces
kubectl logs deployment/my-app
```

All authentication happens transparently. The exec credential plugin fetches a fresh OIDC token for each `kubectl` invocation. If your session expires (after 8 hours), run `vouch login` again.

---

## RBAC configuration

With OIDC authentication, the Kubernetes username is the developer's email address (from the `sub` claim). Use standard Kubernetes RBAC to map users to permissions.

### Cluster-wide admin access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vouch-cluster-admin
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Namespace-scoped access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: vouch-developer-edit
  namespace: staging
subjects:
  - kind: User
    name: bob@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

### Group-based access

If you configure `--oidc-groups-claim` on the API server, you can assign permissions by group:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vouch-team-viewers
subjects:
  - kind: Group
    name: engineering@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Troubleshooting

### "error: You must be logged in to the server (Unauthorized)"

- Confirm you have an active Vouch session: `vouch status`.
- Verify the API server has OIDC configured and can reach the Vouch server's OIDC discovery endpoint.
- Check that the `--oidc-client-id` on the API server matches the `--audience` used during setup (default: `kubernetes`).
- Inspect the token: `vouch credential k8s --cluster my-cluster` and decode the JWT to check the `aud` and `iss` claims.

### "Unable to connect to the server"

- Verify the API server URL: `kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'`.
- Check that the CA certificate is correct if connecting to a cluster with a private CA.
- Ensure your network allows connections to the API server endpoint.

### "No active session"

- Run `vouch login` to authenticate with your YubiKey.

### Token audience mismatch

- The `--audience` flag in `vouch setup k8s` and `vouch credential k8s` must match the `--oidc-client-id` flag on the kube-apiserver. The default for both is `kubernetes`.
- If you used a custom audience during setup, verify with: `kubectl config view --raw -o jsonpath='{.users[?(@.name=="vouch-k8s-my-cluster")].user.exec.args}'`.

### Diagnosing configuration issues

- Run `vouch doctor` to check your Vouch configuration.
- Check API server logs for OIDC validation errors — common issues include the API server being unable to fetch the JWKS endpoint or a clock skew causing token validation failures.
