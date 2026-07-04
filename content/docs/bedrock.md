---
title: "Access Amazon Bedrock with Hardware-Verified Credentials"
linkTitle: "Amazon Bedrock"
description: "Connect to Amazon Bedrock foundation models using short-lived credentials with full audit trails."
weight: 8
subtitle: "Hardware-verified access to Amazon Bedrock"
params:
  docsGroup: aws
---

[Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) uses standard AWS SigV4 authentication. Vouch's `credential_process` provides STS credentials backed by FIDO2 verification, so every Bedrock API call is tied to a hardware-verified human identity, with no shared API keys.

```
YubiKey tap → FIDO2 → Vouch JWT → STS → Amazon Bedrock InvokeModel → CloudTrail
```

{{< tldr >}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once:** grant `bedrock:InvokeModel` (scoped to approved model ARNs) on the Vouch IAM role.
- **Each developer:** nothing new -- `aws bedrock-runtime invoke-model ... --profile vouch` works with the existing profile.
{{< /tldr >}}

---

## Step 1 -- Use Amazon Bedrock with Vouch

{{< role developer >}}

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

{{< role admin >}}

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
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-haiku-20241022"
      ]
    }
  ]
}
```

Restricting to specific model ARNs prevents access to more expensive or higher-capability models without authorization.

---

## The audit chain

CloudTrail records every Amazon Bedrock API call with the full identity chain. The `webIdFederationData` field includes the Vouch OIDC issuer and the user's email (from the `sub` claim).

With [Amazon Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) enabled, you get token counts and costs attributed to hardware-verified identities.

---

## Agent delegation

For automated agents that call Amazon Bedrock on behalf of users, use scoped JWTs with agent-specific `sub` claims and session policies that limit model access -- the human identity chain is preserved while the agent is restricted to only the models it needs.

---

## Related guides

- [Claude & OpenAI APIs](/docs/ai-api-keys/) -- Access the Claude and OpenAI APIs directly (not through AWS) with short-lived tokens via Workload Identity Federation.
- [AWS](/docs/aws/) -- The OIDC federation pattern Vouch uses for AWS STS credentials.
