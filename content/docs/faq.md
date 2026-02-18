---
title: "Frequently Asked Questions"
linkTitle: "FAQ"
description: "Common questions about Vouch — supported hardware, session behavior, platform support, and cost."
weight: 22
subtitle: "Answers to the most common questions about using Vouch"
params:
  docsGroup: manage
---

## Hardware keys

### Which YubiKeys are supported?

Any **FIDO2-compatible** security key works with Vouch. Recommended models:

- **YubiKey 5 series** (5 NFC, 5C, 5C NFC, 5Ci, 5C Nano, 5 Nano) -- Recommended. Supports FIDO2, USB-A or USB-C.
- **YubiKey 5 FIPS series** -- Same as above, with FIPS 140-2 validation.
- **YubiKey Bio** -- Supports FIDO2 with fingerprint biometrics instead of PIN.
- **Security Key by Yubico** (NFC or C NFC) -- Budget option. Supports FIDO2 but lacks other YubiKey features (PIV, OpenPGP).

Other FIDO2-compliant keys (e.g., Google Titan, Feitian, SoloKeys) may work but have not been tested. The key must support the `hmac-secret` extension and resident credentials.

### What happens if I lose my YubiKey?

1. **Report it immediately** to your organization administrator so they can remove the key from your account.
2. Your active session (if any) continues until it expires (up to 8 hours), but no new sessions can be created with the lost key.
3. Enroll a new YubiKey by running `vouch enroll` again.
4. If you have a backup key already enrolled, use that key to log in while you replace the lost one.

### Can I enroll multiple YubiKeys?

Yes. Run `vouch enroll` with each key. This is recommended so you have a backup in case one key is lost or damaged. All enrolled keys can be used interchangeably for `vouch login`.

### Can I use the same YubiKey across multiple Vouch organizations?

Yes. A single YubiKey can hold multiple FIDO2 credentials. Enroll the key with each organization's Vouch server and it will work with all of them.

---

## Sessions and credentials

### How long does a session last?

Sessions last **8 hours** from the time you run `vouch login`. After 8 hours, the session expires and you need to log in again. There is no way to extend a session -- you must re-authenticate with your YubiKey.

### How long do AWS credentials last?

AWS STS credentials obtained through Vouch are valid for up to **1 hour**. The Vouch agent caches them, and when they expire, a new set is fetched automatically (as long as your session is active). You do not need to take any action.

### What happens when my session expires mid-task?

- **SSH connections** already established continue to work. New connections will fail.
- **AWS commands** fail with a credential error. Run `vouch login` and retry.
- **Git operations** fail if they require authentication. Run `vouch login` and retry.
- **Long-running processes** (e.g., `terraform apply`, `cdk deploy`) that started with valid credentials will continue until they need to refresh credentials. If a refresh fails mid-operation, the process may fail partially.

### Are credentials written to disk?

No. All credentials are held in the Vouch agent's process memory. The session token, SSH certificate, and cached STS credentials are never written to a file. If the agent process stops, all credentials are lost.

### Does Vouch work with `aws-vault`?

Vouch replaces the need for `aws-vault`. Both tools solve the same problem (avoiding static AWS credentials), but they work differently. You should use one or the other, not both. If you are currently using `aws-vault`, see the [Migration Guide](/docs/migration/) for switching to Vouch.

---

## Platform support

### Does Vouch work on Windows?

Windows support is limited. The following commands work on Windows:

- `vouch enroll`
- `vouch login`
- `vouch credential aws`
- `vouch credential github`

The SSH agent and SSH certificate integration are **not available** on Windows. The background agent service is also not available -- credentials are obtained directly by each command invocation.

### Does Vouch work on Linux?

Yes. Vouch supports Debian/Ubuntu (APT) and Fedora/RHEL (DNF). The agent runs as a systemd user service. See [Getting Started](/docs/getting-started/) for installation instructions.

### Does Vouch work in WSL?

WSL (Windows Subsystem for Linux) works the same as native Linux. Install the Linux version of the CLI and use USB passthrough for YubiKey access.

### Does Vouch work in containers?

Vouch is designed for developer workstations, not containers. For containers in CI/CD pipelines, use the pipeline's native credential mechanism (e.g., GitHub Actions OIDC, IAM roles for ECS tasks). See [CI/CD Integration](/docs/cicd/) for human approval gates.

---

## AWS

### Does Vouch use AWS STS? Is there a cost?

Yes, Vouch calls [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) to obtain temporary credentials. **AWS STS is free** -- there is no charge for STS API calls.

### What appears in CloudTrail?

Every API call made with Vouch credentials appears in CloudTrail with the assumed role session name set to the developer's email address:

```
arn:aws:sts::123456789012:assumed-role/VouchDeveloper/alice@example.com
```

This provides full per-user attribution for AWS API activity.

### Can I use Vouch with multiple AWS accounts?

Yes. See [Multi-Account AWS Strategy](/docs/aws-multi-account/) for deployment patterns using CloudFormation StackSets or Terraform modules.

### Does Vouch support AWS GovCloud?

Vouch supports standard and China partitions. GovCloud support depends on the integration -- check the specific integration documentation page for cross-partition details.

---

## Identity and access

### Which identity providers does Vouch support?

Vouch currently supports **Google Workspace** as the primary identity provider for user authentication. For automated user provisioning (SCIM), Vouch works with Google Workspace, Okta, Azure AD (Entra ID), and OneLogin.

### What happens when someone leaves the company?

If SCIM is configured: deactivating the user in your identity provider automatically revokes their Vouch sessions. See [SCIM Provisioning](/docs/scim/).

If SCIM is not configured: an administrator must manually remove the user from the Vouch server. Their active session is revoked immediately. Outstanding short-lived credentials expire on their own (within 8 hours at most).

### Can I restrict which team members can assume specific AWS roles?

Yes. Use IAM trust policy conditions to restrict role assumption by email address or domain. See [Restricting access](/docs/aws/#tips-for-restricting-access) in the AWS documentation.

---

## Security

### Is Vouch open source?

The Vouch CLI is open source. The server is operated as a managed service.

### What data does the Vouch server store?

The server stores enrolled FIDO2 public keys, user metadata (email, organization membership), and audit logs. It does not store AWS credentials, SSH private keys, GitHub tokens, or any other secrets. See [Security](/docs/security/) for details.

### What happens if the Vouch server is compromised?

An attacker with server access could issue sessions for any enrolled user, which could be used to broker credentials from external services (AWS, GitHub, etc.). However, the server does not store any credentials itself -- there is no credential vault to extract. See the [threat model](/docs/security/#threat-model) for the full analysis.

### Are credentials encrypted in transit?

Yes. All communication between the CLI and server uses TLS 1.2+. See [Security](/docs/security/#encryption) for details.

---

## Troubleshooting

### "Error: agent not running"

Start the Vouch agent:

- **macOS:** `brew services start vouch`
- **Linux:** `systemctl --user start vouch`

### "Error: no active session"

Run `vouch login` and authenticate with your YubiKey.

### "Error: credential_process returned error"

This typically means your Vouch session has expired or the agent is not running. Run `vouch login` to re-authenticate.

### Where can I get help?

- **Documentation:** [vouch.sh/docs](/docs/)
- **GitHub Issues:** [github.com/vouch-sh/vouch/issues](https://github.com/vouch-sh/vouch/issues)
- **Security issues:** Email security@vouch.sh
