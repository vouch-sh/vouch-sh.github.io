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

This guide creates a new AWS account and connects it to your **Google Workspace** domain with Vouch. Nobody on the team gets an IAM user or access key. Access follows the email domain, every sign-in requires a hardware key, and one `vouch login` also covers SSH, GitHub, Docker registries, and databases.

> **Already have AWS accounts?** Follow [Team Rollout](/docs/rollout/).

[Compare Vouch with IAM user access keys, `aws login`, IAM Identity Center, and AWS projects](/docs/aws/#compare-aws-access-options).

---

## Step 1 -- Get YubiKeys for the team

Order [YubiKey 5 series](https://www.yubico.com/products/yubikey-5-overview/) keys for each team member. Any FIDO2-compatible security key works, but YubiKey 5 series is recommended.

---

## Step 2 -- Create the AWS account

Use AWS's [new sign-up](https://docs.aws.amazon.com/accounts/latest/reference/sign-in-new.html):

1. Go to [aws.amazon.com](https://aws.amazon.com/) and choose **Create account**.
2. Choose email, enter your company email address, verify it, and set a password. This creates your AWS Builder ID, which owns the account.
3. AWS creates your first project in one of three Regions. Note the Region in [AWS Settings](https://settings.aws.com) -- developers need it in Step 5.

Don't invite team members to the project. Invited members get admin access to every service. Vouch gives them access through the IAM role in Step 4, with the permissions you choose.

---

## Step 3 -- Enroll the team

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

## Step 4 -- Deploy AWS federation

Open [AWS CloudShell](https://console.aws.amazon.com/cloudshell/) from the console; it is already signed in as you. Follow the AWS guide to [register the OIDC provider](/docs/aws/#step-1--register-the-vouch-oidc-provider) and [deploy a role](/docs/aws/#step-2--deploy-a-role) with a `*@yourcompany.com` email-domain trust condition. For permissions, **start with `ReadOnlyAccess`** and broaden to exactly what your team needs -- don't default to `PowerUserAccess`. You can tighten access further later (see [Tips for restricting access](/docs/aws/#tips-for-restricting-access) for single-user restrictions, ABAC session tags, and similar patterns).

When the role exists, copy its ARN -- developers need it in Step 5.

---

## Step 5 -- Configure each developer's CLI

Each developer runs one command to configure their AWS profile:

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper --region <PROJECT_REGION>
```

Use the role ARN from Step 4 and the project Region from Step 2. Project accounts deny requests to other Regions, except for global services.

---

## Step 6 -- Log in and verify

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

## Step 7 -- Add more integrations

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
