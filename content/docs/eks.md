---
title: "Amazon EKS"
description: "Authenticate to EKS clusters using AWS IAM and EKS Access Entries with Vouch."
weight: 4
subtitle: "Authenticate to EKS clusters using AWS IAM and EKS Access Entries"
---

Vouch integrates with Amazon EKS through the standard AWS IAM OIDC flow. After running `vouch login`, the CLI provides short-lived AWS credentials that `kubectl` and other Kubernetes tools use to authenticate with your EKS clusters. Combined with EKS Access Entries, this gives you fine-grained, per-user access control backed by hardware security keys.

Managing Kubernetes access with ConfigMap-based `aws-auth` entries is error-prone and hard to audit. Who has access? When was it granted? EKS Access Entries provide a cleaner model: IAM principals map directly to Kubernetes permissions, and every authentication event is recorded in CloudTrail. Combined with Vouch, every `kubectl` command traces back to a hardware-verified human identity.

## How it works

The authentication flow chains four components:

```
vouch login --> vouch credential aws --> aws eks get-token --> kubectl
```

1. **`vouch login`** -- The developer authenticates with their YubiKey and receives an OIDC ID token from the Vouch server.
2. **`vouch credential aws`** -- The CLI exchanges the OIDC token for temporary AWS STS credentials by calling `AssumeRoleWithWebIdentity`.
3. **`aws eks get-token`** -- The AWS CLI (or SDK) uses the STS credentials to generate a short-lived Kubernetes authentication token for the target EKS cluster.
4. **`kubectl`** -- The Kubernetes client sends the token to the EKS API server, which validates it against IAM and applies the permissions defined by Access Entries or RBAC.

Because every step uses short-lived credentials, there are no static kubeconfig tokens or long-lived AWS keys to manage.

---

## Prerequisites

Before setting up EKS authentication with Vouch, ensure you have:

- **Vouch CLI installed and enrolled** -- Complete the [Getting Started](/docs/getting-started/) guide.
- **AWS integration configured** -- Complete the [AWS Integration](/docs/aws/) guide. You need a working `vouch credential aws` setup with an IAM role.
- **AWS CLI v2** installed (`aws --version`).
- **kubectl** installed (`kubectl version --client`).
- **An EKS cluster** with the API server authentication mode set to include `API` (either `API` or `API_AND_CONFIG_MAP`). Clusters created with the default `CONFIG_MAP` mode must be updated.

> See [EKS cluster authentication modes](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html) for details on switching to API mode, and [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html) for the Access Entries feature.

---

## Setup

Configure `kubectl` to use your Vouch-backed AWS credentials for cluster authentication:

```bash
aws eks update-kubeconfig \
  --name YOUR_CLUSTER_NAME \
  --region YOUR_REGION \
  --profile vouch
```

This writes or updates `~/.kube/config` with an `exec`-based user entry that calls `aws eks get-token`. Since the `vouch` AWS profile is configured with `credential_process`, the full chain from YubiKey to Kubernetes is automatic.

### Verify the kubeconfig

Check that the context is set correctly:

```bash
kubectl config current-context
```

The output should include the cluster name and region.

---

## Usage

With everything configured, daily usage is straightforward:

```bash
# Start your day
vouch login

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
- Verify AWS credentials are working: `aws sts get-caller-identity --profile vouch`.
- Check that an EKS Access Entry exists for your IAM role: `aws eks list-access-entries --cluster-name YOUR_CLUSTER_NAME`.
- Ensure the cluster's authentication mode includes `API`. Check with: `aws eks describe-cluster --name YOUR_CLUSTER_NAME --query "cluster.accessConfig.authenticationMode"`.

### "error: exec plugin is configured to use API version client.authentication.k8s.io/v1alpha1"

- Update your kubeconfig by re-running `aws eks update-kubeconfig`. Older versions of the AWS CLI may generate kubeconfig entries using a deprecated API version.
- Ensure you are running AWS CLI v2.

### Credentials expire during long operations

- STS credentials obtained through Vouch last up to 1 hour. For long-running operations such as Helm deployments or large-scale rollouts, run `vouch login` beforehand to ensure a fresh 8-hour session.
- If a command fails mid-operation, run `vouch login` and retry. The kubeconfig exec plugin will automatically pick up the new credentials.

### "AccessDeniedException" when calling EKS APIs

- The IAM role assumed by Vouch needs `eks:DescribeCluster` permission (at minimum) to run `aws eks update-kubeconfig`.
- For Access Entry management, the administrator's IAM role needs `eks:CreateAccessEntry`, `eks:AssociateAccessPolicy`, and related permissions.

### Cannot see resources in a specific namespace

- Check the access scope of the associated access policy. If the policy is scoped to specific namespaces, you can only access resources in those namespaces.
- Verify with: `aws eks list-associated-access-policies --cluster-name YOUR_CLUSTER_NAME --principal-arn YOUR_ROLE_ARN`.

### kubectl works but Helm does not

- Helm may require additional permissions beyond what `view` or `edit` policies provide (e.g., creating `ServiceAccount`, `Role`, or `RoleBinding` resources). Consider using `AmazonEKSAdminPolicy` or a custom RBAC role that grants the necessary permissions.
