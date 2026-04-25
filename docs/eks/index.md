# Authenticate to Amazon EKS without Static Credentials

> Access EKS clusters using OIDC-federated IAM credentials instead of long-lived kubeconfig tokens.

Source: https://vouch.sh/docs/eks/
Last updated: 2026-04-10

---


> **Not using EKS?** For standard Kubernetes clusters (self-hosted, GKE, AKS, k3s, etc.) that use OIDC authentication, see [Kubernetes](/docs/kubernetes/).

[EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html) map IAM principals directly to Kubernetes permissions, with every authentication event recorded in CloudTrail. Combined with Vouch, every `kubectl` command traces back to a hardware-verified human identity -- no static tokens, no shared kubeconfigs.

### EKS authentication modes compared

EKS supports three ways to manage cluster authentication. The table below summarizes the trade-offs:

| | **`aws-auth` ConfigMap** | **EKS Access Entries** | **Access Entries + Vouch** |
|---|---|---|---|
| **How it works** | A Kubernetes ConfigMap (`kube-system/aws-auth`) maps IAM principals to Kubernetes users/groups. | IAM principals are mapped to Kubernetes permissions via the EKS API, outside the cluster. | Same as Access Entries, but credentials are issued through Vouch's OIDC-backed flow. |
| **Credential type** | Long-lived kubeconfig tokens or static IAM keys. | Temporary STS credentials. | Short-lived STS credentials; no local AWS keys needed. |
| **Access management** | Edit a ConfigMap with `kubectl`. Changes are immediate but unversioned. | Create and modify entries via the AWS API, CLI, or Terraform. | Same AWS API/Terraform workflow as Access Entries. |
| **Audit trail** | No native audit trail for ConfigMap edits. Kubernetes audit logs show API calls but not who edited the map. | All changes recorded in CloudTrail. Authentication events logged. | CloudTrail logs plus Vouch audit trail tying every action to a hardware-verified identity. |
| **Granularity** | Map IAM roles/users to Kubernetes groups; RBAC handles the rest. | Built-in access policies (view, edit, admin, cluster-admin) with cluster or namespace scope, plus custom RBAC. | Same granularity as Access Entries. |
| **Revocation** | Edit or delete the ConfigMap entry. Easy to make mistakes. | Delete the access entry via the AWS API. | Remove the IAM role mapping or revoke the user's Vouch enrollment. |
| **Risk** | Misconfigured ConfigMap can lock out all users, including admins. Shared tokens hard to revoke per-user. | No cluster lockout risk -- cluster creator always retains access. Per-principal entries are independent. | Same safety as Access Entries, with the added benefit of no static credentials on developer machines. |
| **EKS auth mode** | `CONFIG_MAP` or `API_AND_CONFIG_MAP` | `API` or `API_AND_CONFIG_MAP` | `API` or `API_AND_CONFIG_MAP` |

> **Recommendation:** Use **EKS Access Entries with Vouch** for the strongest security posture -- every `kubectl` command ties back to a hardware-verified identity with no static credentials on developer machines. The `aws-auth` ConfigMap is considered legacy; AWS recommends Access Entries for all new clusters.

## How it works

The authentication flow chains three components:

```
vouch login --> vouch credential eks --> kubectl
```

1. **`vouch login`** -- The developer authenticates with their YubiKey and receives an OIDC ID token from the Vouch server.
2. **`vouch credential eks`** -- The CLI exchanges the OIDC token for temporary AWS STS credentials, then builds a presigned STS `GetCallerIdentity` URL with the `x-k8s-aws-id` header, base64url-encodes it, and outputs a Kubernetes `ExecCredential` JSON.
3. **`kubectl`** -- The Kubernetes client sends the token to the EKS API server, which validates it against IAM and applies the permissions defined by Access Entries or RBAC.

Because every step uses short-lived credentials, there are no static kubeconfig tokens or long-lived AWS keys to manage. No AWS CLI installation is required -- Vouch handles STS and EKS API calls natively.

---

## Prerequisites

Before setting up EKS authentication with Vouch, ensure you have:

- **Vouch CLI installed and enrolled** -- Complete the [Getting Started](/docs/getting-started/) guide.
- **AWS integration configured** -- Complete the [AWS Integration](/docs/aws/) guide. You need a working `vouch credential aws` setup with an IAM role.
- **kubectl** installed (`kubectl version --client`).
- **An EKS cluster** with the API server authentication mode set to include `API` (either `API` or `API_AND_CONFIG_MAP`). Clusters created with the default `CONFIG_MAP` mode must be updated.

> See [EKS cluster authentication modes](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html) for details on switching to API mode, and [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html) for the Access Entries feature.

---

## Setup

Configure `kubectl` to use your Vouch-backed credentials for cluster authentication:

```bash
vouch setup eks --cluster YOUR_CLUSTER_NAME
```

Optional flags:

- `--region` -- AWS region (auto-detected from your AWS profile or environment if not specified).
- `--profile` -- AWS profile to use (defaults to the auto-detected Vouch profile).
- `--kubeconfig` -- Path to kubeconfig file (defaults to `~/.kube/config`).

This command fetches the cluster endpoint and CA certificate via a native SigV4-signed EKS `DescribeCluster` API call, then writes or updates your kubeconfig with an `exec`-based user entry that calls `vouch credential eks`. The context is named `YOUR_CLUSTER_NAME-vouch`.

### Verify the kubeconfig

Check that the context is set and working:

```bash
kubectl config use-context YOUR_CLUSTER_NAME-vouch
kubectl get pods
```

---

## Usage

With everything configured, daily usage is straightforward:

```bash
# Start your day
vouch login

# Switch to your Vouch EKS context
kubectl config use-context YOUR_CLUSTER_NAME-vouch

# Use kubectl as normal
kubectl get pods
kubectl get namespaces
kubectl logs deployment/my-app
```

All authentication happens transparently. If your session expires (after 8 hours), run `vouch login` again.

---

## Creating EKS Access Entries

EKS Access Entries map IAM principals (users or roles) to Kubernetes permissions. An administrator must create an Access Entry for the IAM role used by Vouch.

### AWS CLI

```bash
# Create the Access Entry for the Vouch IAM role
aws eks create-access-entry \
  --cluster-name YOUR_CLUSTER_NAME \
  --principal-arn arn:aws:iam::123456789012:role/VouchDeveloper \
  --type STANDARD

# Associate an access policy (e.g., cluster admin)
aws eks associate-access-policy \
  --cluster-name YOUR_CLUSTER_NAME \
  --principal-arn arn:aws:iam::123456789012:role/VouchDeveloper \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope '{"type": "cluster"}'
```

To restrict access to specific namespaces:

```bash
aws eks associate-access-policy \
  --cluster-name YOUR_CLUSTER_NAME \
  --principal-arn arn:aws:iam::123456789012:role/VouchDeveloper \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope '{"type": "namespace", "namespaces": ["default", "staging"]}'
```

### Terraform

```hcl
resource "aws_eks_access_entry" "vouch_developer" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.vouch_developer.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "vouch_developer_admin" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.vouch_developer.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }
}
```

For namespace-scoped access:

```hcl
resource "aws_eks_access_policy_association" "vouch_developer_edit" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.vouch_developer.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["default", "staging"]
  }
}
```

---

## Available Access Policies

EKS provides several built-in [access policies](https://docs.aws.amazon.com/eks/latest/userguide/access-policies.html) that map to standard Kubernetes RBAC roles:

| Access Policy ARN | Kubernetes Equivalent | Description |
|---|---|---|
| `arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy` | `cluster-admin` | Full access to all resources in the cluster. |
| `arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminPolicy` | `admin` | Full access within namespaces, plus limited cluster-scoped access. |
| `arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy` | `edit` | Read/write access to most resources in a namespace (no role or role-binding changes). |
| `arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy` | `view` | Read-only access to most resources in a namespace. |
| `arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminViewPolicy` | N/A | Read-only access to all resources in the cluster, including secrets. |

When associating a policy, choose the appropriate **access scope**:

- **`cluster`** -- The policy applies across all namespaces.
- **`namespace`** -- The policy applies only to the specified namespaces.

Cluster-scoped policies (like `AmazonEKSClusterAdminPolicy`) must use `type: cluster`. Namespace-scoped policies (like `AmazonEKSEditPolicy`) can use either scope type.

---

## Custom RBAC

If the built-in access policies do not fit your needs, you can use standard Kubernetes RBAC instead. Create the Access Entry with type `STANDARD` and do not associate any EKS access policies. Then create Kubernetes `ClusterRoleBinding` or `RoleBinding` resources that reference the IAM role's assumed-role ARN.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vouch-developer-custom
subjects:
  - kind: Group
    name: "arn:aws:iam::123456789012:role/VouchDeveloper"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: your-custom-role
  apiGroup: rbac.authorization.k8s.io
```

For per-user permissions using session tags from Vouch, you can create separate IAM roles per team or per access level and map each to different Kubernetes roles.

---

## Troubleshooting

### "error: You must be logged in to the server (Unauthorized)"

- Confirm you have an active Vouch session: `vouch status`.
- Verify AWS credentials are working: `vouch credential aws`.
- Check that an EKS Access Entry exists for your IAM role: `aws eks list-access-entries --cluster-name YOUR_CLUSTER_NAME`.
- Ensure the cluster's authentication mode includes `API`. Check with: `aws eks describe-cluster --name YOUR_CLUSTER_NAME --query "cluster.accessConfig.authenticationMode"`.

### "AWS not configured" error

- Run `vouch setup aws --role <role-arn>` first to configure the AWS integration before setting up EKS. See the [AWS Integration](/docs/aws/) guide.

### Credentials expire during long operations

- STS credentials obtained through Vouch last up to 1 hour. The EKS token itself is valid for 45 seconds, but kubectl re-fetches it automatically via the exec plugin on each command.
- For long-running operations such as Helm deployments or large-scale rollouts, run `vouch login` beforehand to ensure a fresh 8-hour session.
- If a command fails mid-operation, run `vouch login` and retry. The kubeconfig exec plugin will automatically pick up the new credentials.

### "AccessDeniedException" when calling EKS APIs

- The IAM role assumed by Vouch needs `eks:DescribeCluster` permission (at minimum) to run `vouch setup eks`.
- For Access Entry management, the administrator's IAM role needs `eks:CreateAccessEntry`, `eks:AssociateAccessPolicy`, and related permissions.

### Cannot see resources in a specific namespace

- Check the access scope of the associated access policy. If the policy is scoped to specific namespaces, you can only access resources in those namespaces.
- Verify with: `aws eks list-associated-access-policies --cluster-name YOUR_CLUSTER_NAME --principal-arn YOUR_ROLE_ARN`.

### kubectl works but Helm does not

- Helm may require additional permissions beyond what `view` or `edit` policies provide (e.g., creating `ServiceAccount`, `Role`, or `RoleBinding` resources). Consider using `AmazonEKSAdminPolicy` or a custom RBAC role that grants the necessary permissions.

### Diagnosing configuration issues

- Run `vouch doctor` to detect Vouch-configured EKS contexts and check for common misconfigurations.
