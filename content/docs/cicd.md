---
title: "CI/CD Integration"
description: "Add human authorization gates to deployment pipelines using Vouch OIDC tokens."
weight: 15
subtitle: "Add human authorization gates to deployment pipelines"
---

Vouch's OIDC provider can be used as an identity source for CI/CD pipelines. Unlike GitHub Actions' built-in OIDC (which attests *code location* -- "this workflow runs in repo X"), Vouch's OIDC attests *human presence* -- "a verified human authorized this action with their YubiKey."

This enables a pattern where production deployments require an explicit YubiKey tap from an authorized deployer, with the deployer's identity embedded in the resulting AWS credentials via STS session tags.

## How it works

1. A deployer authenticates with Vouch on their local machine (`vouch login`).
2. The deployer generates a short-lived JWT (`vouch credential aws --format token`).
3. The JWT is passed to the CI/CD pipeline (as a workflow input, secret, or environment variable).
4. The pipeline exchanges the JWT for AWS credentials using `AssumeRoleWithWebIdentity`.
5. The deployment proceeds with credentials tied to the deployer's hardware-verified identity.

---

## Step 1 -- Register Vouch as an IAM OIDC Provider

If you have not already, create the OIDC provider in your AWS account. See the [AWS setup guide](/docs/aws/) for full details.

---

## Step 2 -- Create a Deployment Role

Create an IAM role that trusts Vouch tokens and restricts access to authorized deployers:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/{{< instance-url >}}"
      },
      "Action": ["sts:AssumeRoleWithWebIdentity", "sts:TagSession"],
      "Condition": {
        "StringEquals": {
          "{{< instance-url >}}:aud": "{{< instance-url >}}",
          "{{< instance-url >}}:sub": [
            "deployer@example.com",
            "release-lead@example.com"
          ]
        }
      }
    }
  ]
}
```

The `sub` condition restricts which Vouch users can assume this role.

---

## Step 3 -- GitHub Actions Workflow

In this pattern, a deployer authenticates with Vouch on their local machine, and the resulting JWT is passed to the GitHub Actions workflow as an input, which exchanges it for AWS credentials.

```yaml
name: Deploy to Production
on:
  workflow_dispatch:
    inputs:
      vouch_token:
        description: 'Vouch JWT (from: vouch credential aws --format token)'
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

When Vouch tokens are exchanged for STS credentials, the deployer's `email` and `domain` are embedded as STS session tags. These appear in CloudTrail under `userIdentity.sessionContext.webIdFederationData`, providing a clear chain from YubiKey tap to deployment action:

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

Vouch JWTs have a limited lifetime. The deployer must generate the token shortly before triggering the workflow. If the token expires during deployment, the deployer needs to re-authenticate and re-trigger.

### Access denied on AssumeRoleWithWebIdentity

Check that the trust policy's `sub` condition includes the deployer's email and the `aud` matches {{< instance-url >}}.
