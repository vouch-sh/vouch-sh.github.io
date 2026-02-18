---
title: "Connect to EC2 Instances without SSH Port 22"
linkTitle: "AWS Systems Manager"
description: "Use AWS Systems Manager Session Manager with Vouch credentials to reach EC2 instances without opening SSH ports."
weight: 12
subtitle: "Connect to EC2 instances through AWS Systems Manager"
params:
  docsGroup: infra
---

Managing SSH access to EC2 instances typically means opening port 22, distributing keys, and maintaining security groups. Every open port is an attack surface, and every key is a secret to manage.

AWS Systems Manager Session Manager eliminates all of this: connections go through the Systems Manager service, every session is logged in CloudTrail, and no inbound ports are required. With Vouch, the underlying AWS credentials are hardware-verified and short-lived -- after `vouch login`, you can start sessions to any SSM-managed instance.

## How it works

1. **`vouch login`** -- The developer authenticates with their YubiKey and receives an OIDC ID token.
2. **`credential_process`** -- The AWS CLI calls Vouch to exchange the OIDC token for temporary STS credentials.
3. **`aws ssm start-session`** -- The AWS CLI uses the STS credentials to start an AWS SSM Session Manager session with the target instance.
4. **CloudTrail** -- Every session start is recorded with the Vouch user's identity via STS session tags.

```
vouch login → credential_process → STS → AWS SSM start-session → CloudTrail
```

---

## Prerequisites

Before using AWS Systems Manager Session Manager with Vouch, ensure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- The **[AWS integration](/docs/aws/)** configured (OIDC provider and IAM role)
- The **[Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)** installed for the AWS CLI
- EC2 instances with the **AWS SSM Agent** installed and an instance profile that allows AWS SSM connections

---

## Step 1 -- Start a session

With an active Vouch session, connect to an instance:

```bash
aws ssm start-session \
  --target i-0abc123def456 \
  --profile vouch
```

This opens an interactive shell session on the target instance without SSH.

---

## Step 2 -- SSH over AWS SSM

You can also use AWS SSM as a transport for standard SSH connections. This lets you use familiar SSH tooling (scp, rsync, port forwarding) while routing traffic through AWS SSM instead of direct TCP connections.

### Automated setup (recommended)

The `vouch setup ssm` command configures your SSH client automatically:

```bash
vouch setup ssm
```

| Flag | Description |
|---|---|
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--region` | AWS region to use in the ProxyCommand |
| `--hosts` | Host patterns to match (default: `i-* mi-*`) |
| `--force` | Overwrite any existing SSM configuration in `~/.ssh/config` |

To specify a profile and region explicitly:

```bash
vouch setup ssm --profile vouch --region us-east-1
```

This adds the following to your `~/.ssh/config`:

```
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --profile vouch --region us-east-1"
```

### Manual setup

Alternatively, add the following to your `~/.ssh/config` manually:

```
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --profile vouch --region <YOUR_REGION>"
```

Replace `<YOUR_REGION>` with your AWS region (e.g., `us-east-1`).

### Connect

With either setup method, connect with SSH as usual:

```bash
ssh i-0abc123def456
```

Because Vouch configures the SSH agent with a certificate, the SSH session is authenticated with both your Vouch SSH certificate and your hardware-backed AWS credentials.

---

## Port forwarding

AWS SSM supports port forwarding to access services on private instances. This is useful for reaching databases (such as RDS) through a bastion instance without exposing them to the internet:

```bash
# Forward local port 5432 to an RDS instance through an EC2 bastion
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.cluster-abc123.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}' \
  --profile vouch
```

Then connect to `localhost:5432` with your database client.

---

## IAM permissions

The IAM role assumed by Vouch needs AWS SSM session permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession",
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": [
        "arn:aws:ec2:us-east-1:123456789012:instance/*",
        "arn:aws:ssm:us-east-1::document/AWS-StartSSHSession",
        "arn:aws:ssm:us-east-1::document/AWS-StartPortForwardingSessionToRemoteHost"
      ]
    }
  ]
}
```

You can restrict access to specific instances using resource ARNs or tag-based conditions.

---

## Session identity and audit

When Vouch exchanges an OIDC token for STS credentials, the user's email and domain are embedded as session tags. These appear in CloudTrail under `userIdentity.sessionContext.webIdFederationData`, providing a clear chain from YubiKey tap to AWS SSM session.

AWS SSM also records session activity. With [Session Manager logging](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html) enabled, you get a complete record of commands executed during each session.

---

## Troubleshooting

### "SessionManagerPlugin is not found"

The Session Manager plugin is not installed or not in your PATH. Install it from the [AWS documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

### "TargetNotConnected"

The target instance does not have a running AWS SSM agent or cannot reach the AWS Systems Manager endpoint. Verify:

- The instance has an IAM instance profile with `AmazonSSMManagedInstanceCore` permissions.
- The AWS SSM agent is running: `sudo systemctl status amazon-ssm-agent`.
- The instance can reach the AWS SSM endpoint (either through a NAT gateway or VPC endpoint).

### "Access denied" when starting a session

- Verify your IAM role has `ssm:StartSession` permission for the target instance.
- Check that the instance ARN matches the resource constraints in your IAM policy.
- Ensure you have an active Vouch session: `vouch login`.

### `vouch setup ssm` reports existing SSM configuration

If the command detects an existing SSM block in your `~/.ssh/config`, it will not overwrite it by default. Use the `--force` flag to replace the existing configuration:

```bash
vouch setup ssm --force
```

### `vouch doctor` reports SSM issues

Run `vouch doctor` to diagnose SSM configuration problems. If issues are found, run `vouch setup ssm` to reconfigure your SSH client automatically.
