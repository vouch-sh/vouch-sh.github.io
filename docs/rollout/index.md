# Roll Out Vouch to Your Team

> The playbook for giving a growing team access to AWS, EKS, CodeCommit, CodeArtifact, and more — without becoming the bottleneck.

Source: https://vouch.sh/docs/rollout/
Last updated: 2026-07-04

---
Your job: get everyone on the team into AWS, Kubernetes, and the package and repo services around them — fast, without handing out static keys, and without you becoming the person everyone waits on. This page is the whole playbook. Each step links to a deep-dive guide, but you should rarely need one.

{{&lt; tldr &gt;}}
1. **Once (admin):** register the OIDC provider, deploy one IAM role, note its ARN — [30 minutes](#day-0----the-foundation).
2. **Per service (admin):** add a few IAM actions to that role — [checklist below](#enable-services).
3. **Per developer:** they run `vouch enroll`, then `vouch setup aws --role &lt;ARN&gt;` — [send them this](#onboard-developers).
4. **Ongoing:** offboarding is your IdP &#43; [one deny statement](#when-someone-leaves); nothing to hunt down.
{{&lt; /tldr &gt;}}

## Day 0 -- The foundation

{{&lt; role admin &gt;}}

Everything below hangs off one OIDC provider and one IAM role. This is the only part with real decisions in it.

1. **Enroll yourself.** Install the CLI and enroll your YubiKey -- [Getting Started](/docs/getting-started/) (5 minutes). The first person to enroll from your Google Workspace domain becomes the organization owner.

2. **Register the Vouch OIDC provider** in AWS -- one CLI command or a few lines of Terraform/CloudFormation. Exactly one per organization, in your management account if you have an AWS Organization. [AWS guide, Step 1](/docs/aws/#step-1----register-the-vouch-oidc-provider).

3. **Pick your account layout.** This decides what roles you deploy:

   | Your AWS layout | What to deploy | Guide |
   |---|---|---|
   | Single account | One `VouchDeveloper` role | [AWS guide](/docs/aws/#step-2----deploy-a-role) |
   | Multiple accounts (Organizations) | A hub role in the management account, a spoke role per account (StackSets or a Terraform module) | [Multi-account](/docs/aws-multi-account/) |
   | IAM Identity Center already in place | Register Vouch as a trusted token issuer | [Identity Center](/docs/aws-multi-account/#aws-iam-identity-center) |

4. **Deploy the role(s)** with the standard trust policy scoped to `*@yourdomain.com`. Start the permissions policy at `ReadOnlyAccess` and broaden deliberately. **Write down the role ARN** -- it is the only thing developers need from you.

5. **Plan user lifecycle.** Under ~15 people, manual member management in the [admin dashboard](/docs/admin/) is fine. At 15&#43;, connect [SCIM](/docs/scim/) so your IdP creates and deactivates Vouch users automatically.

&lt;div class=&#34;checkpoint&#34;&gt;
&lt;p&gt;&lt;strong&gt;Foundation is done when...&lt;/strong&gt;&lt;/p&gt;
&lt;ul&gt;
  &lt;li&gt;&lt;code&gt;aws sts get-caller-identity --profile vouch&lt;/code&gt; returns an assumed-role ARN with &lt;em&gt;your&lt;/em&gt; email in the session name.&lt;/li&gt;
  &lt;li&gt;You have the role ARN (and hub-role ARN, if multi-account) saved somewhere you can paste from.&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;

---

## Enable services

{{&lt; role admin &gt;}}

Each additional service is **IAM permissions on the role you already deployed, plus at most one admin action**. Developers then enable it with a single command. Full guides are linked for when something goes sideways.

| Service | Add to the role&#39;s permissions | One-time admin action | Each developer runs |
|---|---|---|---|
| [AWS CLI / SDKs](/docs/aws/) | Your chosen policy (start with `ReadOnlyAccess`) | -- | `vouch setup aws --role &lt;ARN&gt;` |
| [EKS](/docs/eks/) | `eks:DescribeCluster` | Create an [Access Entry](/docs/eks/#creating-eks-access-entries) mapping the role to cluster permissions | `vouch setup eks --cluster &lt;NAME&gt;` |
| [CodeCommit](/docs/codecommit/) | `codecommit:GitPull`, `codecommit:GitPush` | -- | `vouch setup codecommit --configure` |
| [CodeArtifact](/docs/codeartifact/) | `codeartifact:GetAuthorizationToken`, `codeartifact:GetRepositoryEndpoint`, `codeartifact:ReadFromRepository`, `sts:GetServiceBearerToken` | -- | `vouch setup codeartifact --tool &lt;npm\|pip\|cargo\|pnpm\|uv&gt; --repository &lt;REPO&gt;` |
| [Docker / ECR](/docs/docker/) | `ecr:GetAuthorizationToken` &#43; pull/push actions | -- | `vouch setup docker` |
| [SSM Session Manager](/docs/ssm/) | `ssm:StartSession` (scope by tag) | Instances need the SSM agent &#43; instance profile | `aws ssm start-session --target &lt;ID&gt; --profile vouch` |
| [RDS / Aurora](/docs/databases/) | `rds-db:connect` | Create IAM-auth database users (`GRANT rds_iam`) | `vouch exec --type rds -- psql ...` |
| [Bedrock](/docs/bedrock/) | `bedrock:InvokeModel` (scope by model) | -- | works via the `vouch` AWS profile |
| [GitHub](/docs/github/) | -- (not AWS) | Install the Vouch GitHub App on your org | `vouch setup github` |
| [SSH](/docs/ssh/) | -- (not AWS) | Trust the Vouch CA in `sshd_config` | automatic after `vouch login` |

Sequence tip: ship **AWS first** (everything else chains off it), then EKS and CodeCommit/CodeArtifact, then the rest as teams ask for them.

---

## Onboard developers

{{&lt; role developer &gt;}}

Developers never touch IAM. Paste the block below into Slack or your onboarding wiki, fill in the two placeholders, and each person is productive in about five minutes -- no action from you.

````markdown
**Set up Vouch (one time, ~5 min, YubiKey required)**

1. Install the CLI and background agent:

       brew install vouch-sh/tap/vouch
       brew services start vouch

   (Linux/Windows: https://vouch.sh/docs/getting-started/)

2. Enroll your YubiKey -- opens the browser for SSO, then asks for a tap:

       vouch enroll --server https://us.vouch.sh

3. Connect AWS (paste the role ARN from your admin):

       vouch setup aws --role &lt;ROLE_ARN&gt;

4. Connect the cluster (if you use Kubernetes):

       vouch setup eks --cluster &lt;CLUSTER_NAME&gt;

5. Start each workday with one tap:

       vouch login

Check it worked: `aws sts get-caller-identity --profile vouch`
shows your email in the role ARN. Questions -&gt; #devops-help
````

Enrollment needs no invite codes or approval -- anyone authenticating through your Google Workspace domain lands in your organization automatically.

---

## Pilot, then roll out

Vouch installs alongside existing credentials -- the `vouch` AWS profile, SSH certificates, and stacked Git credential helpers don&#39;t disturb anything your team uses today. So de-risk the rollout the boring way:

1. **Pilot on yourself plus one volunteer** for a week, with static credentials still in place.
2. **Migrate one integration at a time**, AWS first. The [migration guide](/docs/migration/) has the recommended order, per-integration checklists, and a rollback plan for each integration.
3. **Onboard the team** with the block above once the pilot is clean.
4. **Revoke static credentials last** -- deactivate old AWS access keys, remove stale `authorized_keys` entries and PATs only after everyone has run on Vouch for a week. Commands in [migration, Phase 3](/docs/migration/#phase-3----revoke-old-credentials).

CI/CD pipelines are a separate track: they have no YubiKeys and should use their platform&#39;s own OIDC federation or instance roles, not Vouch. See [CI/CD considerations](/docs/migration/#cicd-considerations) -- Vouch&#39;s [CI/CD integration](/docs/cicd/) adds *human approval gates* on top, it does not replace machine credentials.

---

## When someone leaves

This is the payoff for never distributing static keys: offboarding is your identity provider plus, at most, one IAM statement.

1. **Deactivate their account in your IdP** (you were doing this anyway).
   - **With [SCIM](/docs/scim/):** their Vouch sessions are revoked automatically the moment the IdP deactivates them. Nothing else to do on the Vouch side.
   - **Without SCIM:** use **Deactivate** and **Revoke credentials** in the [admin dashboard](/docs/admin/).
2. **Cut off cached AWS credentials** (optional, for immediate effect): STS credentials already on their laptop live up to 1 hour. To kill them instantly, add the `vouch:Email` deny statement to your roles or as an SCP -- copy it from [Revoke access for a specific user](/docs/aws/#revoke-access-for-a-specific-user).

What expires on its own:

| Credential | Gone after |
|---|---|
| Vouch session (new credentials) | immediately on deactivation |
| Cached AWS STS credentials | up to 1 hour |
| SSH certificate, GitHub tokens | up to 8 hours (end of session) |
| ECR / CodeArtifact authorization tokens | up to 12 hours |

There are no access keys to hunt down, no `authorized_keys` to scrub, no PATs to revoke. Full failure-mode detail (including **break-glass access** if the Vouch server is ever unreachable) is in [Availability](/docs/availability/).

---

## Related guides

- [AWS Integration](/docs/aws/) -- the OIDC provider and role this whole page builds on.
- [Multi-Account AWS](/docs/aws-multi-account/) -- hub/spoke roles, StackSets, SCP guardrails.
- [SCIM Provisioning](/docs/scim/) -- automatic user lifecycle from your IdP.
- [Migration Guide](/docs/migration/) -- integration-by-integration checklists and rollback.
- [Availability and Failure Modes](/docs/availability/) -- offline behavior and break-glass planning.
