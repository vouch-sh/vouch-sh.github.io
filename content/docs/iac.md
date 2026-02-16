---
title: "Use Terraform and CDK with Hardware-Verified Credentials"
linkTitle: "Infrastructure as Code"
description: "Run CDK, Terraform, SAM, and other IaC tools using short-lived AWS credentials from Vouch."
weight: 14
subtitle: "Use CDK, Terraform, SAM, and other IaC tools with Vouch"
params:
  docsGroup: manage
---

IaC tools like Terraform, CDK, and SAM need AWS credentials to provision infrastructure. If those credentials are long-lived access keys, a compromised dev machine could modify production infrastructure. If they're shared across the team, there's no audit trail showing who deployed what.

If a tool reads `~/.aws/config`, it already works with Vouch. The `credential_process` setting in your Vouch AWS profile is picked up by the AWS SDK, which means every IaC tool that uses the SDK gets hardware-verified credentials automatically. No plugins or wrappers needed.

## AWS CDK

```bash
cdk deploy --profile vouch
```

CDK has known issues with SSO credential discovery ([#23520](https://github.com/aws/aws-cdk/issues/23520), [#21328](https://github.com/aws/aws-cdk/issues/21328)) that `credential_process` avoids entirely.

---

## AWS SAM

```bash
sam deploy --profile vouch
```

---

## Terraform

```bash
# Set the AWS profile for the session
export AWS_PROFILE=vouch
terraform plan
terraform apply
```

This works for the AWS provider's authentication. Terraform Cloud registry auth is separate and not handled by Vouch.

---

## AWS Copilot

```bash
export AWS_PROFILE=vouch
copilot deploy
```

---

## AWS Amplify

```bash
export AWS_PROFILE=vouch
amplify push
```

With Vouch, you can skip `amplify configure` entirely -- there is no need to generate long-lived IAM access keys for local development. The `credential_process` in your Vouch profile provides credentials on demand.

---

## Pulumi

```bash
export AWS_PROFILE=vouch
pulumi up
```

---

## Tips

### Setting `AWS_PROFILE` vs `--profile`

Some tools accept `--profile vouch` as a flag, while others only read the `AWS_PROFILE` environment variable. Setting the environment variable works universally:

```bash
export AWS_PROFILE=vouch
```

Add this to your shell profile (`.bashrc`, `.zshrc`) to make it the default for all sessions.

### Multiple accounts

If you deploy to multiple AWS accounts, set up separate Vouch profiles for each:

```bash
vouch setup aws --role arn:aws:iam::111111111111:role/VouchDeveloper --profile vouch-dev
vouch setup aws --role arn:aws:iam::222222222222:role/VouchDeveloper --profile vouch-prod
```

Then specify the profile per command:

```bash
cdk deploy --profile vouch-dev
cdk deploy --profile vouch-prod
```
