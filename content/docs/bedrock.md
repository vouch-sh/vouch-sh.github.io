---
title: "Access Amazon Bedrock with Hardware-Verified Credentials"
linkTitle: "Amazon Bedrock"
description: "Connect to Amazon Bedrock foundation models using short-lived credentials with full audit trails."
weight: 16
subtitle: "Hardware-verified access to Amazon Bedrock"
params:
  docsGroup: infra
---

AI model access creates unique audit and cost challenges. Shared API keys make it impossible to attribute usage to individual developers, and a leaked key means unlimited access to expensive model invocations with no way to trace who's responsible.

[Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) uses standard AWS SigV4 authentication. Vouch's `credential_process` provides STS credentials backed by FIDO2 verification, which means every Amazon Bedrock API call is tied to a hardware-verified human identity.

```
YubiKey tap → FIDO2 → Vouch JWT → STS → Amazon Bedrock InvokeModel → CloudTrail
```

## Why this matters

- **Every API call has a verified identity** -- no shared API keys or service accounts for human users.
- **CloudTrail captures the full chain** -- from YubiKey tap to model invocation.
- **Cost attribution per user** -- with [Amazon Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) enabled, token counts and costs are tied to hardware-verified identities.

---

## Step 1 -- Use Amazon Bedrock with Vouch

Any tool that uses the AWS SDK for Amazon Bedrock will pick up Vouch credentials automatically:

```bash
# AWS CLI
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-sonnet-4-20250514 \
  --body '{"prompt": "Hello"}' \
  --profile vouch \
  output.json
```

```python
# Python (boto3)
import boto3

session = boto3.Session(profile_name='vouch')
bedrock = session.client('bedrock-runtime')
response = bedrock.invoke_model(
    modelId='anthropic.claude-sonnet-4-20250514',
    body='{"prompt": "Hello"}'
)
```

---

## Step 2 -- Restrict model access by IAM policy

Use IAM policies to control which foundation models each role can invoke:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-haiku-4-20250514"
      ]
    }
  ]
}
```

Restricting to specific model ARNs prevents users from accessing more expensive or higher-capability models without authorization.

---

## The audit chain

CloudTrail records every Amazon Bedrock API call with the full identity chain. The `webIdFederationData` field includes the Vouch OIDC issuer and the user's email (from the `sub` claim).

With [Amazon Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) enabled, you get token counts and costs attributed to hardware-verified identities.

---

## Agent delegation

For automated agents that call Amazon Bedrock on behalf of users, you can use scoped JWTs with agent-specific `sub` claims and session policies that limit model access. This preserves the human identity chain while restricting the agent's capabilities to only the models it needs.
