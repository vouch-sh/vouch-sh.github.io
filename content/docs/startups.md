---
title: "Vouch for Startups"
aliases:
  - "/blog/your-startup-doesnt-need-iam-users/"
  - "/blog/google-workspace-to-aws/"
linkTitle: "Startups"
description: "Connect Google Workspace to AWS with OIDC federation — no IAM users, access keys, or IAM Identity Center required."
weight: 2
subtitle: "Set up a new AWS account with federated access instead of IAM users"
params:
  docsGroup: featured
---

A new AWS account does not need IAM users. IAM users come with long-lived access keys that never expire and can be committed to Git, leaked in logs, or compromised by malware. Rotation is a manual process, and offboarding requires finding and deleting every key the user created. AWS supports OIDC federation as an alternative: authentication happens through the identity system your team already uses.

If your team uses **Google Workspace**, Vouch bridges it directly into AWS. One `vouch login` gives every developer short-lived credentials for AWS, SSH, GitHub, Docker registries, databases, and more -- all tied to their Google Workspace identity, all backed by a hardware key.

> **Already have AWS accounts?** Follow [Team Rollout](/docs/rollout/).

---

## Step 1 -- Get YubiKeys for the team

Order [YubiKey 5 series](https://www.yubico.com/products/yubikey-5-overview/) keys for each team member. Any FIDO2-compatible security key works, but YubiKey 5 series is recommended.

---

## Step 2 -- Enroll the team

Each person installs the CLI:

{{< tabs >}}
{{< tab "macOS" >}}
Install with Homebrew:

```
brew install vouch-sh/tap/vouch
```

After installing, start the Vouch background service:

```
brew services start vouch
```
{{< /tab >}}
{{< tab "Debian / Ubuntu" >}}
```bash
# Import GPG key
curl -fsSL https://packages.vouch.sh/gpg/vouch.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/vouch-archive-keyring.gpg > /dev/null

# Add repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/vouch-archive-keyring.gpg] https://packages.vouch.sh/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/vouch.list > /dev/null

# Install
sudo apt-get update && sudo apt-get install -y vouch
```
{{< /tab >}}
{{< tab "Fedora / RHEL" >}}
```bash
sudo tee /etc/yum.repos.d/vouch.repo << 'EOF'
[vouch]
name=Vouch
baseurl=https://packages.vouch.sh/rpm/$basearch/
gpgcheck=1
gpgkey=https://packages.vouch.sh/gpg/vouch.asc
enabled=1
EOF

sudo dnf install -y vouch
```
{{< /tab >}}
{{< tab "Windows" >}}
Install with [winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/):

```
winget install SmokeTurner.Vouch
```

> **Note:** Windows support is limited. The SSH agent and SSH integration are not available on Windows. Only basic authentication and credential exchange commands are supported: `enroll`, `login`, `credential aws`, and `credential github`.
{{< /tab >}}
{{< /tabs >}}

Then enrolls with the same server:

```bash
vouch enroll --server https://{{< instance-url >}}
```

The first person to enroll from your Google Workspace domain becomes the organization owner. Everyone else on the same domain joins that organization. Initial enrollment needs no invite code or admin approval. (Enrolling an additional key later asks for a tap from a key that is already registered.)

---

## Step 3 -- Deploy AWS federation

Follow the AWS guide to [register the OIDC provider](/docs/aws/#step-1----register-the-vouch-oidc-provider) and [deploy a role](/docs/aws/#step-2----deploy-a-role) with a `*@yourcompany.com` email-domain trust condition. For permissions, **start with `ReadOnlyAccess`** and broaden to exactly what your team needs -- don't default to `PowerUserAccess`. You can tighten access further later (see [Tips for restricting access](/docs/aws/#tips-for-restricting-access) for single-user restrictions, ABAC session tags, and similar patterns).

When the role exists, copy its ARN -- you'll need it in Step 4.

---

## Step 4 -- Configure each developer's CLI

Each developer runs one command to configure their AWS profile:

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

Use the role ARN from Step 3.

---

## Step 5 -- Log in and verify

```bash
vouch login
aws sts get-caller-identity --profile vouch
```

You should see your email address in the assumed role ARN:

```json
{
  "UserId": "AROA...:alice@yourcompany.com",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@yourcompany.com"
}
```

From here, `aws s3 ls --profile vouch`, `cdk deploy`, `terraform apply`, and any other AWS tool works with no additional setup.

---

## Step 6 -- Add more integrations

With the same `vouch login` session, configure the rest of your toolchain:

| Integration | Setup | Docs |
|---|---|---|
| SSH certificates | Automatic after login | [SSH](/docs/ssh/) |
| GitHub | `vouch setup github` | [GitHub](/docs/github/) |
| Docker (ECR) | `vouch setup docker` | [Docker](/docs/docker/) |
| CodeCommit | `vouch setup codecommit` | [CodeCommit](/docs/codecommit/) |
| CodeArtifact | `vouch setup codeartifact` | [CodeArtifact](/docs/codeartifact/) |
| EKS | `vouch setup eks` | [EKS](/docs/eks/) |

Each integration is configured with one command. After setup, all integrations share the same `vouch login` session.

---

## What happens when someone leaves

Deactivate their Google Workspace account. Without [SCIM](/docs/scim/), also select **Deactivate** and **Revoke credentials** in the [admin dashboard](/docs/admin/). With SCIM, the IdP deactivation revokes their Vouch sessions automatically. Cached AWS credentials expire within 1 hour. There are no access keys, `authorized_keys` entries, or PATs to revoke. Expiry details: [When someone leaves](/docs/rollout/#when-someone-leaves).

---

## Scaling up

As your team grows, switch to the [Team Rollout playbook](/docs/rollout/) -- it covers service enablement, onboarding, and offboarding as an ongoing process. The usual thresholds:

- **5--15 people:** Manual user management works fine. SCIM is optional.
- **15--50 people:** Set up [SCIM provisioning](/docs/scim/) to automate onboarding and offboarding with Google Workspace.
- **Multiple AWS accounts:** See [Multi-Account AWS Strategy](/docs/aws-multi-account/) for chaining into dev/staging/prod accounts through a single hub.
- **CI/CD gates:** Add [human approval gates](/docs/cicd/) to production deployments.
- **Compliance requirements:** See [Security](/docs/security/) and the [Threat Model](/docs/threat-model/) for details on how Vouch protects credentials.

---

## Why not IAM Identity Center?

[AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) (formerly AWS SSO) is AWS's own solution for federated access. It is designed for organizations managing permission sets across multiple AWS accounts. The differences for a small team:

| Consideration | IAM Identity Center | Vouch |
|---|---|---|
| **Setup complexity** | Requires an AWS Organizations management account, an Identity Center instance, permission sets, and user/group sync | Deploy one CloudFormation template and run `vouch setup aws` |
| **Scope** | AWS only | AWS + SSH + GitHub + Docker + CodeCommit + CodeArtifact + databases + more |
| **Authentication** | Browser-based SSO (MFA depends on IdP config) | FIDO2 hardware key (phishing-resistant by design) |
| **Typical team size** | 20+ people across multiple accounts | 2--50 people |
| **Credential type** | Session credentials via `aws sso login` | Session credentials via `vouch login` |

For 20+ people with per-account permission sets across multiple AWS accounts, use IAM Identity Center (Vouch can [federate into it](/docs/aws-multi-account/#aws-iam-identity-center)). For a small team, Vouch requires less setup: one CloudFormation template and one command per developer.

## Why not AWS Builder ID?

[AWS Builder ID](https://docs.aws.amazon.com/signin/latest/userguide/sign-in-aws_builder_id.html) provides individual developer identity for AWS services. The key difference: Builder ID is **individual** identity, not **organizational** identity. It does not know about your Google Workspace domain, your team structure, or your offboarding process. You cannot restrict AWS access to "people who work at my company" using Builder ID alone.

Vouch federates your organization's identity (Google Workspace domain) into AWS, so access is tied to employment by design.
