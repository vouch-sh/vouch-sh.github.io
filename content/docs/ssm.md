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

SSM Session Manager eliminates all of this: connections go through the Systems Manager service, every session is logged in CloudTrail, and no inbound ports are required. With Vouch, the underlying AWS credentials are hardware-verified and short-lived -- after `vouch login`, you can start sessions to any SSM-managed instance.

## How it works

1. **`vouch login`** -- The developer authenticates with their YubiKey and receives an OIDC ID token.
2. **`credential_process`** -- The AWS CLI calls Vouch to exchange the OIDC token for temporary STS credentials.
3. **`aws ssm start-session`** -- The AWS CLI uses the STS credentials to start a Session Manager session with the target instance.
4. **CloudTrail** -- Every session start is recorded with the Vouch user's identity via STS session tags.

```
vouch login → credential_process → STS → SSM start-session → CloudTrail
```

---

## Prerequisites

Before using SSM Session Manager with Vouch, ensure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- The **[AWS integration](/docs/aws/)** configured (OIDC provider and IAM role)
- The **[Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)** installed for the AWS CLI
- EC2 instances with the **SSM Agent** installed and an instance profile that allows SSM connections

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

## Step 2 -- SSH over SSM

You can also use SSM as a transport for standard SSH connections. This lets you use familiar SSH tooling (scp, rsync, port forwarding) while routing traffic through SSM instead of direct TCP connections.

Add the following to your `~/.ssh/config`:

```
Host i-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --profile vouch"
    User ec2-user
```

Then connect with SSH as usual:

```bash
ssh i-0abc123def456
```

Because Vouch configures the SSH agent with a certificate, the SSH session is authenticated with both your Vouch SSH certificate and your hardware-backed AWS credentials.

---

## Port forwarding

SSM supports port forwarding to access services on private instances. This is useful for reaching databases (such as RDS) through a bastion instance without exposing them to the internet:

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

The IAM role assumed by Vouch needs SSM session permissions:

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

When Vouch exchanges an OIDC token for STS credentials, the user's email and domain are embedded as session tags. These appear in CloudTrail under `userIdentity.sessionContext.webIdFederationData`, providing a clear chain from YubiKey tap to SSM session.

SSM also records session activity. With [Session Manager logging](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html) enabled, you get a complete record of commands executed during each session.

---

## Troubleshooting

### "SessionManagerPlugin is not found"

The Session Manager plugin is not installed or not in your PATH. Install it from the [AWS documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

### "TargetNotConnected"

The target instance does not have a running SSM agent or cannot reach the Systems Manager endpoint. Verify:

- The instance has an IAM instance profile with `AmazonSSMManagedInstanceCore` permissions.
- The SSM agent is running: `sudo systemctl status amazon-ssm-agent`.
- The instance can reach the SSM endpoint (either through a NAT gateway or VPC endpoint).

### "Access denied" when starting a session

- Verify your IAM role has `ssm:StartSession` permission for the target instance.
- Check that the instance ARN matches the resource constraints in your IAM policy.
- Ensure you have an active Vouch session: `vouch login`.
