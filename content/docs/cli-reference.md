---
title: "Vouch CLI Reference"
linkTitle: "CLI Reference"
description: "Complete command reference for the Vouch CLI — login, credentials, setup, and configuration."
weight: 17
subtitle: "Complete command reference for the Vouch CLI"
params:
  docsGroup: manage
---

This page documents all available Vouch CLI commands. For installation instructions, see [Getting Started](/docs/getting-started/).

---

## Authentication

### `vouch enroll`

Register your YubiKey with a Vouch server and link it to your identity.

```
vouch enroll --server <SERVER_URL>
```

| Flag | Description |
|---|---|
| `--server` | The Vouch server URL to enroll with (e.g., `https://us.vouch.sh`) |

You only need to enroll once per YubiKey. The CLI saves the server configuration locally for future commands.

### `vouch login`

Authenticate with your YubiKey and start an 8-hour session.

```
vouch login
```

After login, all credential helpers use the session automatically. Run this once at the start of each workday.

### `vouch logout`

End the current session and clear all cached credentials.

```
vouch logout
```

### `vouch status`

Display the current session status, including remaining session time and active integrations.

```
vouch status
```

---

## Setup

Setup commands configure credential helpers for each integration. Run these once per machine.

### `vouch setup aws`

Configure the AWS credential process for an IAM role.

```
vouch setup aws --role <ROLE_ARN>
```

| Flag | Description |
|---|---|
| `--role` | The IAM role ARN to assume |
| `--profile` | AWS profile name to configure (default: `vouch`) |
| `--region` | Default AWS region for the profile |

See [AWS Integration](/docs/aws/) for full details.

### `vouch setup ssh`

Configure the SSH client to use the Vouch agent for certificate authentication.

```
vouch setup ssh
```

See [SSH Certificates](/docs/ssh/) for full details.

### `vouch setup github`

Configure Git to use Vouch as the credential helper for GitHub.

```
vouch setup github [--configure]
```

| Flag | Description |
|---|---|
| `--configure` | Apply the configuration automatically (without this flag, the command only prints the configuration) |

See [GitHub Integration](/docs/github/) for full details.

### `vouch setup docker`

Configure Docker to use Vouch as the credential helper for container registries.

```
vouch setup docker
```

See [Docker Registries](/docs/docker/) for full details.

### `vouch setup cargo`

Configure Cargo to use Vouch as the credential provider for private registries.

```
vouch setup cargo [--registry <NAME>] [--configure]
```

| Flag | Description |
|---|---|
| `--registry` | Name of the Cargo registry to configure |
| `--configure` | Apply the configuration automatically |

See [Cargo Integration](/docs/cargo/) for full details.

### `vouch setup codeartifact`

Configure authentication for an AWS CodeArtifact repository.

```
vouch setup codeartifact --domain <DOMAIN> --repository <REPO> [--region <REGION>]
```

| Flag | Description |
|---|---|
| `--domain` | The CodeArtifact domain name |
| `--repository` | The CodeArtifact repository name |
| `--region` | AWS region for the CodeArtifact domain |

See [AWS CodeArtifact](/docs/codeartifact/) for full details.

### `vouch setup codecommit`

Configure Git to use Vouch as the credential helper for AWS CodeCommit.

```
vouch setup codecommit
```

See [AWS CodeCommit](/docs/codecommit/) for full details.

### `vouch setup eks`

Configure kubectl to use Vouch for EKS cluster authentication.

```
vouch setup eks --cluster <CLUSTER_NAME> [--role <ROLE_ARN>] [--region <REGION>]
```

| Flag | Description |
|---|---|
| `--cluster` | The EKS cluster name |
| `--role` | IAM role ARN to assume for the cluster |
| `--region` | AWS region where the cluster is located |

See [Amazon EKS](/docs/eks/) for full details.

---

## Credentials

Credential commands obtain service-specific credentials from your active session. These are typically called automatically by credential helpers, but can be run manually for debugging.

### `vouch credential aws`

Obtain temporary AWS STS credentials.

```
vouch credential aws --role <ROLE_ARN>
```

| Flag | Description |
|---|---|
| `--role` | The IAM role ARN to assume |

### `vouch credential ssh`

Display information about the current SSH certificate.

```
vouch credential ssh
```

### `vouch credential github`

Obtain a short-lived GitHub access token.

```
vouch credential github
```

### `vouch credential docker`

Obtain credentials for a container registry.

```
vouch credential docker
```

### `vouch credential cargo`

Obtain a token for a private Cargo registry.

```
vouch credential cargo
```

### `vouch credential codeartifact`

Obtain an AWS CodeArtifact authorization token.

```
vouch credential codeartifact
```

---

## Key management

### `vouch register`

Register an additional YubiKey with your account.

```
vouch register
```

This allows you to use multiple hardware keys (e.g., a primary and a backup) with the same Vouch identity.

### `vouch keys list`

List all registered security keys for your account.

```
vouch keys list
```

### `vouch keys remove`

Remove a registered security key from your account.

```
vouch keys remove <KEY_ID>
```

### `vouch keys rename`

Rename a registered security key.

```
vouch keys rename <KEY_ID> --name <NEW_NAME>
```

---

## Environment

### `vouch exec`

Run a command with Vouch credentials injected as environment variables.

```
vouch exec -- <COMMAND> [ARGS...]
```

Example:

```bash
vouch exec -- terraform plan
```

### `vouch env`

Print environment variables for the current session.

```
vouch env
```

### `vouch init`

Initialize the Vouch agent and background services.

```
vouch init
```

---

## Diagnostics

### `vouch doctor`

Run diagnostic checks to verify your Vouch installation and configuration.

```
vouch doctor
```

This checks:

- CLI version and updates
- Agent connectivity
- Server reachability
- Integration configurations (SSH, AWS, Git, Docker, Cargo)

### `vouch completions`

Generate shell completion scripts.

```
vouch completions <SHELL>
```

Supported shells: `bash`, `zsh`, `fish`, `powershell`.

Example:

```bash
# Add to your ~/.zshrc
eval "$(vouch completions zsh)"
```

---

## Binary download verification

If you downloaded the Vouch CLI binary directly from the [GitHub releases](https://github.com/vouch-sh/vouch/releases) page, you can verify its integrity using the SHA256 checksums and SLSA provenance attestation published alongside each release.

### SHA256 checksum

Each release includes a `checksums.txt` file. Verify the downloaded binary:

```bash
sha256sum --check checksums.txt
```

### SLSA provenance

Vouch release binaries are built with SLSA Level 3 provenance. You can verify the provenance attestation using the [slsa-verifier](https://github.com/slsa-framework/slsa-verifier) tool:

```bash
slsa-verifier verify-artifact vouch-linux-amd64 \
  --provenance-path vouch-linux-amd64.intoto.jsonl \
  --source-uri github.com/vouch-sh/vouch
```
