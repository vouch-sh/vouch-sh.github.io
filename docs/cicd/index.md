# Add Human Approval Gates to CI/CD Pipelines

> Require a YubiKey tap before production deployments — hardware-verified identity embedded in every CI/CD credential.

Source: https://vouch.sh/docs/cicd/
Last updated: 2026-07-13

---
Vouch&#39;s OIDC attests *human presence*, so production deployments can require an explicit YubiKey tap from an authorized deployer, with the deployer&#39;s identity embedded in the resulting AWS credentials via STS session tags.

&gt; **Fully automated pipelines:** If your pipeline does not need a human approval gate and should run unattended, use the [client credentials grant](/docs/applications/#client-credentials-machine-to-machine) instead -- CI/CD systems authenticate with a client ID and secret, no YubiKey tap.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once (this whole page):** create the deployment role and add the token-exchange step to the workflow.
- **Each deploy:** the pipeline waits for a JWT an authorized deployer mints locally with `vouch credential aws --role &lt;ROLE_ARN&gt;`.
{{&lt; /tldr &gt;}}

## How it works

1. A deployer runs `vouch login` locally, then mints a short-lived JWT with `vouch credential aws --role &lt;ROLE_ARN&gt;`.
2. The JWT is passed to the pipeline as a workflow input, secret, or environment variable.
3. The pipeline exchanges it for AWS credentials via `AssumeRoleWithWebIdentity` and deploys with credentials tied to the deployer&#39;s hardware-verified identity.

---

## Step 1 -- Register Vouch as an IAM OIDC Provider

{{&lt; role admin &gt;}}

Create the OIDC provider in your AWS account if you have not already -- see the [AWS setup guide](/docs/aws/).

---

## Step 2 -- Create a Deployment Role

{{&lt; role admin &gt;}}

This is [the shared trust policy](/docs/aws/#shared-trust-policy) from the AWS guide, plus a `sub` condition restricting which Vouch users can assume the role:

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Principal&#34;: {
        &#34;Federated&#34;: &#34;arn:aws:iam::ACCOUNT_ID:oidc-provider/us.vouch.sh&#34;
      },
      &#34;Action&#34;: [
        &#34;sts:AssumeRoleWithWebIdentity&#34;,
        &#34;sts:SetSourceIdentity&#34;,
        &#34;sts:TagSession&#34;
      ],
      &#34;Condition&#34;: {
        &#34;StringEquals&#34;: {
          &#34;us.vouch.sh:aud&#34;: &#34;https://us.vouch.sh&#34;,
          &#34;us.vouch.sh:sub&#34;: [
            &#34;deployer@example.com&#34;,
            &#34;release-lead@example.com&#34;
          ]
        },
        &#34;Bool&#34;: {
          &#34;sts:RoleAuthorizedByIdp&#34;: &#34;true&#34;
        }
      }
    }
  ]
}
```

---

## Step 3 -- GitHub Actions Workflow

{{&lt; role admin &gt;}}

The workflow takes the deployer&#39;s JWT as an input:

```yaml
name: Deploy to Production
on:
  workflow_dispatch:
    inputs:
      vouch_token:
        description: &#39;Vouch JWT (from: vouch credential aws --role &lt;ROLE_ARN&gt;)&#39;
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Exchange Vouch JWT for AWS credentials
        run: |
          CREDS=$(aws sts assume-role-with-web-identity \
            --role-arn arn:aws:iam::ACCOUNT_ID:role/ProductionDeployRole \
            --role-session-name &#34;deploy-${{ github.run_id }}&#34; \
            --web-identity-token &#34;${{ inputs.vouch_token }}&#34; \
            --output json)

          echo &#34;AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r &#39;.Credentials.AccessKeyId&#39;)&#34; &gt;&gt; $GITHUB_ENV
          echo &#34;AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r &#39;.Credentials.SecretAccessKey&#39;)&#34; &gt;&gt; $GITHUB_ENV
          echo &#34;AWS_SESSION_TOKEN=$(echo $CREDS | jq -r &#39;.Credentials.SessionToken&#39;)&#34; &gt;&gt; $GITHUB_ENV

      - name: Deploy
        run: cdk deploy --require-approval never
```

---

## Audit trail

The deployer&#39;s `email` and `domain` are embedded as STS session tags and appear in CloudTrail under `userIdentity.sessionContext.webIdFederationData`, providing a clear chain from YubiKey tap to deployment action:

```json
{
  &#34;userIdentity&#34;: {
    &#34;type&#34;: &#34;AssumedRole&#34;,
    &#34;principalId&#34;: &#34;AROA...:deploy-12345&#34;,
    &#34;sessionContext&#34;: {
      &#34;webIdFederationData&#34;: {
        &#34;federatedProvider&#34;: &#34;arn:aws:iam::ACCOUNT:oidc-provider/us.vouch.sh&#34;,
        &#34;attributes&#34;: {
          &#34;email&#34;: &#34;deployer@example.com&#34;,
          &#34;domain&#34;: &#34;example.com&#34;
        }
      }
    }
  }
}
```

---

## Troubleshooting

### Token expired in pipeline

Vouch JWTs have a limited lifetime -- generate the token shortly before triggering the workflow. If it expires during deployment, the deployer needs to re-authenticate and re-trigger.

### Access denied on AssumeRoleWithWebIdentity

Check that the trust policy&#39;s `sub` condition includes the deployer&#39;s email and the `aud` matches `https://us.vouch.sh`.
