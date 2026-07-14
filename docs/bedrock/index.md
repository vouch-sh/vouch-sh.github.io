# Access Amazon Bedrock with Hardware-Verified Credentials

> Connect to Amazon Bedrock foundation models using short-lived credentials with full audit trails.

Source: https://vouch.sh/docs/bedrock/
Last updated: 2026-07-04

---
[Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) uses standard AWS SigV4 authentication. Vouch&#39;s `credential_process` provides STS credentials backed by FIDO2 verification, so every Bedrock API call is tied to a hardware-verified human identity, with no shared API keys.

```
YubiKey tap → FIDO2 → Vouch JWT → STS → Amazon Bedrock InvokeModel → CloudTrail
```

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once:** grant `bedrock:InvokeModel` (scoped to approved model ARNs) on the Vouch IAM role.
- **Each developer:** nothing new -- `aws bedrock-runtime invoke-model ... --profile vouch` works with the existing profile.
{{&lt; /tldr &gt;}}

---

## Step 1 -- Use Amazon Bedrock with Vouch

{{&lt; role developer &gt;}}

Any tool that uses the AWS SDK for Amazon Bedrock will pick up Vouch credentials automatically:

```bash
# AWS CLI
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-sonnet-4-20250514 \
  --body &#39;{&#34;prompt&#34;: &#34;Hello&#34;}&#39; \
  --profile vouch \
  output.json
```

```python
# Python (boto3)
import boto3

session = boto3.Session(profile_name=&#39;vouch&#39;)
bedrock = session.client(&#39;bedrock-runtime&#39;)
response = bedrock.invoke_model(
    modelId=&#39;anthropic.claude-sonnet-4-20250514&#39;,
    body=&#39;{&#34;prompt&#34;: &#34;Hello&#34;}&#39;
)
```

---

## Step 2 -- Restrict model access by IAM policy

{{&lt; role admin &gt;}}

Use IAM policies to control which foundation models each role can invoke:

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: [
        &#34;bedrock:InvokeModel&#34;,
        &#34;bedrock:InvokeModelWithResponseStream&#34;
      ],
      &#34;Resource&#34;: [
        &#34;arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514&#34;,
        &#34;arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-haiku-20241022&#34;
      ]
    }
  ]
}
```

Restricting to specific model ARNs prevents access to more expensive or higher-capability models without authorization.

---

## The audit chain

CloudTrail records every Amazon Bedrock API call with the full identity chain. The `webIdFederationData` field includes the Vouch OIDC issuer and the user&#39;s email (from the `sub` claim).

With [Amazon Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) enabled, you get token counts and costs attributed to hardware-verified identities.

---

## Agent delegation

For automated agents that call Amazon Bedrock on behalf of users, use scoped JWTs with agent-specific `sub` claims and session policies that limit model access -- the human identity chain is preserved while the agent is restricted to only the models it needs.

---

## Related guides

- [Claude &amp; OpenAI APIs](/docs/ai-api-keys/) -- Access the Claude and OpenAI APIs directly (not through AWS) with short-lived tokens via Workload Identity Federation.
- [AWS](/docs/aws/) -- The OIDC federation pattern Vouch uses for AWS STS credentials.
