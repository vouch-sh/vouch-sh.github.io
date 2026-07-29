# Vouch for Startups

> Skip IAM users, access keys, and IAM Identity Center. Go from Google Workspace to AWS in minutes with OIDC federation.

Source: https://vouch.sh/docs/startups/
Last updated: 2026-07-04

---
You just created an AWS account. Every tutorial says &#34;create an IAM user.&#34; Don&#39;t.

IAM users come with long-lived access keys that never expire, get committed to Git, leaked in logs, and compromised by malware. Rotating them is a manual chore. When someone leaves, you have to hunt down every key they ever created. And none of this is necessary -- AWS supports OIDC federation, which means you can authenticate with the identity system your team already uses.

If your team uses **Google Workspace**, Vouch bridges it directly into AWS. One `vouch login` gives every developer short-lived credentials for AWS, SSH, GitHub, Docker registries, databases, and more -- all tied to their Google Workspace identity, all backed by a hardware key.

&gt; **Already have AWS accounts and a growing team?** This page is for day-one setups. For rolling Vouch out across an existing organization -- service enablement checklists, onboarding blocks, offboarding -- use the [Team Rollout playbook](/docs/rollout/). Wondering how Vouch compares to [IAM Identity Center or Builder ID](#why-not-iam-identity-center)? That&#39;s at the end.

---

## What you get

After following this guide, your team will have:

- **No IAM users** -- Every developer authenticates with their Google Workspace account &#43; YubiKey.
- **No access keys** -- AWS credentials are temporary (1 hour) and never written to disk.
- **No credential files** -- No `~/.aws/credentials`, no SSH keys to distribute, no GitHub PATs.
- **Instant offboarding** -- When someone&#39;s Google Workspace account is deactivated, their AWS access ends immediately.
- **Full audit trail** -- Every AWS API call in CloudTrail shows which developer made it.

---

## Step 1 -- Get YubiKeys for the team

Order [YubiKey 5 series](https://www.yubico.com/products/yubikey-5-overview/) keys for each team member. Any FIDO2-compatible security key works, but YubiKey 5 series is recommended.

---

## Step 2 -- Enroll the team

The first person to log into Vouch from your Google Workspace domain becomes the organization owner.

**Owner enrollment:**

```bash
# Install the CLI
brew install vouch-sh/tap/vouch
brew services start vouch

# Enroll (first person becomes the org owner)
vouch enroll --server https://us.vouch.sh
```

**Team member enrollment:**

Each team member installs the CLI and enrolls with the same server:

```bash
brew install vouch-sh/tap/vouch
brew services start vouch
vouch enroll --server https://us.vouch.sh
```

As long as they authenticate with the same Google Workspace domain, they join the same organization. No invite codes or admin approval needed for initial enrollment.

---

## Step 3 -- Deploy AWS federation

Follow the [AWS integration guide](/docs/aws/) to register the OIDC provider and deploy a role with a `*@yourcompany.com` email-domain trust condition. For permissions, **start with `ReadOnlyAccess`** and broaden to exactly what your team needs -- don&#39;t default to `PowerUserAccess`. You can tighten access further later (see [Tips for restricting access](/docs/aws/#tips-for-restricting-access) for single-user restrictions, ABAC session tags, and similar patterns).

When the role exists, copy its ARN -- you&#39;ll need it in Step 4.

---

## Step 4 -- Configure each developer&#39;s CLI

Each developer runs one command to configure their AWS profile:

```bash
vouch setup aws --role arn:aws:iam::123456789012:role/VouchDeveloper
```

Replace the account ID with your own (printed in the CloudFormation stack outputs).

---

## Step 5 -- Log in and verify

```bash
vouch login
aws sts get-caller-identity --profile vouch
```

You should see your email address in the assumed role ARN:

```json
{
  &#34;UserId&#34;: &#34;AROA...:alice@yourcompany.com&#34;,
  &#34;Account&#34;: &#34;123456789012&#34;,
  &#34;Arn&#34;: &#34;arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@yourcompany.com&#34;
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

Each integration takes one command. After setup, every tool uses the same session -- one YubiKey tap covers the entire developer toolchain.

---

## What happens when someone leaves

This is where the investment pays off: deactivate their Google Workspace account (which you were going to do anyway) and their access ends -- no access keys to hunt down, no SSH keys to remove from servers, no PATs to revoke. The exact sequence, expiry timeline, and the optional AWS-side deny statement are in [When someone leaves](/docs/rollout/#when-someone-leaves) on the rollout playbook.

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

[AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) (formerly AWS SSO) is AWS&#39;s own solution for federated access. It&#39;s a good product, but it&#39;s designed for enterprises with dozens of accounts and hundreds of users. For a startup:

| Consideration | IAM Identity Center | Vouch |
|---|---|---|
| **Setup complexity** | Requires an AWS Organizations management account, an Identity Center instance, permission sets, and user/group sync | Deploy one CloudFormation template and run `vouch setup aws` |
| **Scope** | AWS only | AWS &#43; SSH &#43; GitHub &#43; Docker &#43; CodeCommit &#43; CodeArtifact &#43; databases &#43; more |
| **Authentication** | Browser-based SSO (MFA depends on IdP config) | FIDO2 hardware key (phishing-resistant by design) |
| **Team size sweet spot** | 20&#43; people across multiple accounts | 2--50 people |
| **Credential type** | Session credentials via `aws sso login` | Session credentials via `vouch login` |

If you have a large organization with complex permission requirements across many AWS accounts, IAM Identity Center is the right choice (and Vouch can [federate into it](/docs/aws-multi-account/#aws-iam-identity-center)). If you are a startup that wants secure AWS access without the overhead, Vouch gets you there faster.

## Why not AWS Builder ID?

[AWS Builder ID](https://docs.aws.amazon.com/signin/latest/userguide/sign-in-aws_builder_id.html) provides individual developer identity for AWS services. The key difference: Builder ID is **individual** identity, not **organizational** identity. It does not know about your Google Workspace domain, your team structure, or your offboarding process. You cannot restrict AWS access to &#34;people who work at my company&#34; using Builder ID alone.

Vouch federates your organization&#39;s identity (Google Workspace domain) into AWS, so access is tied to employment by design.
