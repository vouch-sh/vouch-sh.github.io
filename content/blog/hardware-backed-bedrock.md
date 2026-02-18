---
title: "Hardware-Backed Credentials for Amazon Bedrock"
description: "Every Bedrock API call should trace to a verified human. Hardware-backed credentials make per-user cost attribution and audit trails automatic."
date: 2026-02-10
---

Amazon Bedrock gives developers access to foundation models from Anthropic, Meta, Mistral, and others through a standard AWS API. Unlike hosted model APIs that use API keys, Bedrock uses AWS SigV4 authentication -- the same credential mechanism as S3, DynamoDB, and every other AWS service.

This is a significant security advantage. But most teams undermine it by sharing AWS credentials across developers, making it impossible to attribute model usage to individual people.

## The attribution problem

When a team shares an IAM user or uses a common set of access keys for Bedrock development:

- **CloudTrail shows the IAM user, not the human.** If five developers share `bedrock-dev-user` credentials, you cannot tell who invoked which model.
- **Cost attribution is impossible.** Bedrock charges per token (input and output). Without per-user identity, you cannot determine which developer or experiment consumed the most tokens.
- **Model invocation logging is less useful.** Bedrock supports [model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) that captures prompts and responses. Without per-user identity, these logs cannot be attributed to specific people.
- **Access revocation is coarse-grained.** Revoking one person's access means rotating the shared credentials, which affects everyone.

## Hardware-verified Bedrock access

With Vouch, every Bedrock API call traces back to a hardware-verified human identity:

```
YubiKey tap → FIDO2 assertion → Vouch session → OIDC token → AWS STS → Bedrock InvokeModel → CloudTrail
```

The chain of trust starts at a physical hardware key and ends in CloudTrail with the developer's email address:

```
arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@example.com
```

### What this gives you

**Per-user CloudTrail entries.** Every `InvokeModel`, `InvokeModelWithResponseStream`, and `Converse` call shows which developer made it.

**Per-user cost attribution.** Combined with Bedrock's model invocation logging, you can calculate per-user token consumption and costs. This is valuable for:

- Understanding which experiments or projects drive model costs.
- Setting per-user or per-team budgets.
- Identifying inefficient prompt patterns.

**Phishing-resistant access.** FIDO2 authentication is bound to the Vouch server's origin. An attacker cannot trick a developer into authenticating to a malicious service to steal Bedrock credentials.

**Instant revocation.** When a developer leaves, their Google Workspace account is deactivated, their Vouch session is revoked, and they can no longer obtain STS credentials for Bedrock. No shared keys to rotate.

## Setup

If you already have the [AWS integration](https://vouch.sh/docs/aws/) configured, Bedrock works with no additional setup. Any tool that uses the AWS SDK picks up Vouch credentials automatically:

```python
import boto3

session = boto3.Session(profile_name='vouch')
bedrock = session.client('bedrock-runtime')

response = bedrock.invoke_model(
    modelId='anthropic.claude-sonnet-4-20250514',
    contentType='application/json',
    body='{"prompt": "Hello", "max_tokens": 100}'
)
```

The SDK calls `credential_process`, which calls `vouch credential aws`, which returns STS credentials derived from the developer's hardware-backed session.

### IAM permissions

The IAM role assumed through Vouch needs Bedrock permissions:

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
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/*"
    }
  ]
}
```

Restrict the `Resource` to specific models if you want to control which foundation models developers can access.

### Cost controls with session tags

Vouch sets session tags (`email` and `domain`) when assuming IAM roles. You can use these tags in IAM policies to restrict access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/domain": "example.com"
        }
      }
    }
  ]
}
```

This ensures only users from your domain can invoke models, even if an STS credential were somehow obtained by an outsider.

## The AI safety angle

As AI capabilities grow, so does the importance of knowing who is using them and how. Shared API keys make AI model access anonymous. Hardware-backed credentials make it attributable.

For teams building with foundation models, per-user identity is not just a security feature -- it's an audit and compliance requirement that will only become more important.

## Getting started

See the [Amazon Bedrock integration guide](https://vouch.sh/docs/bedrock/) for the full setup, including IAM permissions and SDK configuration.
