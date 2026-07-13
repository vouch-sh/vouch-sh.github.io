---
title: "Add Human Approval Gates to CI/CD Pipelines"
linkTitle: "CI/CD"
description: "Require a YubiKey tap before production deployments — hardware-verified identity embedded in every CI/CD credential."
weight: 10
subtitle: "Add human authorization gates to deployment pipelines"
params:
  docsGroup: aws
---

Vouch's OIDC attests *human presence*, so production deployments can require an explicit YubiKey tap from an authorized deployer, with the deployer's identity embedded in the resulting AWS credentials via STS session tags.

> **Fully automated pipelines:** If your pipeline does not need a human approval gate and should run unattended, use the [client credentials grant](/docs/applications/#client-credentials-machine-to-machine) instead -- CI/CD systems authenticate with a client ID and secret, no YubiKey tap.

{{< tldr >}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once (this whole page):** create the deployment role and add the token-exchange step to the workflow.
- **Each deploy:** the pipeline waits for a JWT an authorized deployer mints locally with `vouch credential aws --role <ROLE_ARN>`.
{{< /tldr >}}

## How it works

1. A deployer runs `vouch login` locally, then mints a short-lived JWT with `vouch credential aws --role <ROLE_ARN>`.
2. The JWT is passed to the pipeline as a workflow input, secret, or environment variable.
3. The pipeline exchanges it for AWS credentials via `AssumeRoleWithWebIdentity` and deploys with credentials tied to the deployer's hardware-verified identity.

---

## Step 1 -- Register Vouch as an IAM OIDC Provider

{{< role admin >}}

Create the OIDC provider in your AWS account if you have not already -- see the [AWS setup guide](/docs/aws/).

---

## Step 2 -- Create a Deployment Role

{{< role admin >}}

This is [the shared trust policy](/docs/aws/#shared-trust-policy) from the AWS guide, plus a `sub` condition restricting which Vouch users can assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/{{< instance-url >}}"
      },
      "Action": [
        "sts:AssumeRoleWithWebIdentity",
        "sts:SetSourceIdentity",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "{{< instance-url >}}:aud": "https://{{< instance-url >}}",
          "{{< instance-url >}}:sub": [
            "deployer@example.com",
            "release-lead@example.com"
          ]
        },
        "Bool": {
          "sts:RoleAuthorizedByIdp": "true"
        }
      }
    }
  ]
}
```

---

## Step 3 -- GitHub Actions Workflow

{{< role admin >}}

The workflow takes the deployer's JWT as an input:

```yaml
name: Deploy to Production
on:
  workflow_dispatch:
    inputs:
      vouch_token:
        description: 'Vouch JWT (from: vouch credential aws --role <ROLE_ARN>)'
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
            --role-session-name "deploy-${{ github.run_id }}" \
            --web-identity-token "${{ inputs.vouch_token }}" \
            --output json)

          echo "AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')" >> $GITHUB_ENV
          echo "AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')" >> $GITHUB_ENV
          echo "AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')" >> $GITHUB_ENV

      - name: Deploy
        run: cdk deploy --require-approval never
```

---

## Audit trail

The deployer's `email` and `domain` are embedded as STS session tags and appear in CloudTrail under `userIdentity.sessionContext.webIdFederationData`, providing a clear chain from YubiKey tap to deployment action:

```json
{
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROA...:deploy-12345",
    "sessionContext": {
      "webIdFederationData": {
        "federatedProvider": "arn:aws:iam::ACCOUNT:oidc-provider/{{< instance-url >}}",
        "attributes": {
          "email": "deployer@example.com",
          "domain": "example.com"
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

Check that the trust policy's `sub` condition includes the deployer's email and the `aud` matches `https://{{< instance-url >}}`.
