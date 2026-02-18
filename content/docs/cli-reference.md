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

The `--server` flag is a global option available on all commands (e.g., `--server https://us.vouch.sh`). The CLI saves the server configuration locally after enrollment, so subsequent commands use it automatically.

You only need to enroll once per YubiKey.

### `vouch login`

Authenticate with your YubiKey and start an 8-hour session.

```
vouch login [--timeout <SECONDS>]
```

| Flag | Description |
|---|---|
| `--timeout` | Timeout in seconds for YubiKey detection (default: `60`, use `0` for no timeout) |

After login, all credential helpers use the session automatically. Run this once at the start of each workday.

### `vouch logout`

End the current session and clear all cached credentials.

```
vouch logout
```

### `vouch status`

Display the current session status, including remaining session time and active integrations.

```
vouch status [--json]
```

| Flag | Description |
|---|---|
| `--json` | Output as JSON |

---

## Setup

Setup commands configure credential helpers for each integration. Run these once per machine.

### `vouch setup aws`

Configure the AWS credential process for an IAM role.

```
vouch setup aws --role <ROLE_ARN> [--profile <PROFILE>]
```

| Flag | Description |
|---|---|
| `--role` | The IAM role ARN to assume (required) |
| `--profile` | AWS profile name to configure (default: `vouch`; additional profiles auto-name as `vouch-2`, `vouch-3`, etc.) |

See [AWS Integration](/docs/aws/) for full details.

### `vouch setup ssh`

Configure the SSH client to use the Vouch agent for certificate authentication.

```
vouch setup ssh [--hosts <PATTERN>]
```

| Flag | Description |
|---|---|
| `--hosts` | Host patterns to trust with this CA (e.g., `*.example.com`). If specified, adds an entry to `~/.ssh/known_hosts`. |

See [SSH Certificates](/docs/ssh/) for full details.

### `vouch setup github`

Configure Git to use Vouch as the credential helper for GitHub.

```
vouch setup github [--host <HOST>] [--configure]
```

| Flag | Description |
|---|---|
| `--host` | GitHub host to configure (default: `github.com`) |
| `--configure` | Apply the configuration automatically (without this flag, the command only prints the configuration) |

See [GitHub Integration](/docs/github/) for full details.

### `vouch setup docker`

Configure Docker to use Vouch as the credential helper for container registries.

```
vouch setup docker [--configure] [REGISTRIES...]
```

| Flag | Description |
|---|---|
| `--configure` | Apply the configuration automatically (without this flag, the command only prints the configuration) |
| `REGISTRIES` | Container registry URLs to configure (e.g., `ghcr.io`) |

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

Configure a package manager for an AWS CodeArtifact repository.

```
vouch setup codeartifact --tool <TOOL> --repository <REPO> [--domain <DOMAIN>] [--domain-owner <ACCOUNT_ID>] [--region <REGION>] [--profile <PROFILE>]
```

| Flag | Description |
|---|---|
| `--tool` | Package manager to configure: `cargo`, `pip`, or `npm` (required) |
| `--repository` | The AWS CodeArtifact repository name (required) |
| `--domain` | The AWS CodeArtifact domain name (optional if a profile is configured) |
| `--domain-owner` | AWS account ID that owns the domain (optional if a profile is configured) |
| `--region` | AWS region (default: `us-east-1`; optional if a profile is configured) |
| `--profile` | Named AWS CodeArtifact profile to use or create (stores domain/owner/region for reuse) |

See [AWS CodeArtifact](/docs/codeartifact/) for full details.

### `vouch setup codecommit`

Configure Git to use Vouch as the credential helper for AWS CodeCommit.

```
vouch setup codecommit [--region <REGION>] [--profile <PROFILE>] [--configure]
```

| Flag | Description |
|---|---|
| `--region` | AWS region (default: wildcard matching all regions) |
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--configure` | Apply the configuration automatically (without this flag, the command only prints the configuration) |

See [AWS CodeCommit](/docs/codecommit/) for full details.

### `vouch setup eks`

Configure kubectl to use Vouch for EKS cluster authentication.

```
vouch setup eks --cluster <CLUSTER_NAME> [--region <REGION>] [--profile <PROFILE>] [--kubeconfig <PATH>]
```

| Flag | Description |
|---|---|
| `--cluster` | The EKS cluster name (required) |
| `--region` | AWS region (auto-detected from AWS profile or environment if not specified) |
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--kubeconfig` | Path to kubeconfig file (defaults to `~/.kube/config`) |

See [Amazon EKS](/docs/eks/) for full details.

---

## Credentials

Credential commands obtain service-specific credentials from your active session. These are typically called automatically by credential helpers, but can be run manually for debugging.

### `vouch credential aws`

Obtain temporary AWS STS credentials.

```
vouch credential aws --role <ROLE_ARN> [--session-name <NAME>]
```

| Flag | Description |
|---|---|
| `--role` | The IAM role ARN to assume (required) |
| `--session-name` | Session name for the assumed role |

### `vouch credential ssh`

Obtain an SSH certificate from the Vouch server.

```
vouch credential ssh [--key <PATH>]
```

| Flag | Description |
|---|---|
| `--key` | Path to SSH private key (default: `~/.ssh/id_ed25519_vouch`) |

### `vouch credential github`

Git credential helper for GitHub. This command is invoked automatically by Git when configured via `vouch setup github`. Users should not call it directly.

### `vouch credential docker`

Docker credential helper for container registries. This command is invoked automatically by Docker when configured via `vouch setup docker`. Users should not call it directly.

### `vouch credential cargo`

Cargo credential provider for private registries. This command is invoked automatically by Cargo when configured via `vouch setup cargo`. Users should not call it directly.

### `vouch credential codeartifact`

Obtain an AWS CodeArtifact authorization token.

```
vouch credential codeartifact [--domain <DOMAIN>] [--domain-owner <ACCOUNT_ID>] [--region <REGION>] [--profile <PROFILE>]
```

| Flag | Description |
|---|---|
| `--domain` | The AWS CodeArtifact domain name (optional if a profile is configured) |
| `--domain-owner` | AWS account ID that owns the domain (optional if a profile is configured) |
| `--region` | AWS region (default: `us-east-1`; optional if a profile is configured) |
| `--profile` | Named AWS CodeArtifact profile to use |

---

## Key management

### `vouch register`

Register an additional YubiKey with your account.

```
vouch register [--name <NAME>] [--timeout <SECONDS>]
```

| Flag | Description |
|---|---|
| `--name` | Human-readable name for this YubiKey (default: `YubiKey`) |
| `--timeout` | Timeout in seconds for YubiKey detection (default: `60`, use `0` for no timeout) |

This allows you to use multiple hardware keys (e.g., a primary and a backup) with the same Vouch identity.

### `vouch keys list`

List all registered security keys for your account.

```
vouch keys list [--json]
```

| Flag | Description |
|---|---|
| `--json` | Output as JSON |

### `vouch keys remove`

Remove a registered security key from your account.

```
vouch keys remove <KEY_ID> [--force]
```

| Flag | Description |
|---|---|
| `-f`, `--force` | Skip the confirmation prompt |

### `vouch keys rename`

Rename a registered security key.

```
vouch keys rename <KEY_ID> <NEW_NAME>
```

---

## Environment

### `vouch exec`

Run a command with Vouch credentials injected as environment variables.

```
vouch exec --type <TYPE> [FLAGS...] -- <COMMAND> [ARGS...]
```

| Flag | Description |
|---|---|
| `--type` | Credential type to inject: `aws`, `github`, or `codeartifact` (required) |
| `--role` | AWS IAM role ARN (required when `--type aws`) |
| `--session-name` | Session name for the assumed role (when `--type aws`) |
| `--ca-domain` | AWS CodeArtifact domain name (when `--type codeartifact`; optional if a profile is configured) |
| `--ca-domain-owner` | AWS account ID that owns the domain (when `--type codeartifact`; optional if a profile is configured) |
| `--ca-region` | AWS region (when `--type codeartifact`; optional if a profile is configured) |
| `--ca-profile` | Named AWS CodeArtifact profile to use (when `--type codeartifact`) |

When `--type codeartifact`, injects `CODEARTIFACT_AUTH_TOKEN` into the subprocess environment.

Examples:

```bash
# AWS credentials
vouch exec --type aws --role arn:aws:iam::123456789012:role/VouchDeveloper -- terraform plan

# AWS CodeArtifact token
vouch exec --type codeartifact -- mvn deploy -s settings.xml
```

### `vouch env`

Output credential environment variables for use with `eval`. This sets variables like `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` (for AWS), `GITHUB_TOKEN` (for GitHub), or `CODEARTIFACT_AUTH_TOKEN` (for AWS CodeArtifact) in your current shell.

```
eval "$(vouch env --type <TYPE> [--shell <SHELL>] [FLAGS...])"
```

| Flag | Description |
|---|---|
| `--type` | Credential type: `aws`, `github`, or `codeartifact` (required) |
| `--shell` | Shell syntax: `bash` or `fish` (default: `bash`). The `bash` syntax also works for zsh. |
| `--role` | AWS IAM role ARN (required when `--type aws`) |
| `--session-name` | Session name for the assumed role (when `--type aws`) |
| `--ca-domain` | AWS CodeArtifact domain name (when `--type codeartifact`; optional if a profile is configured) |
| `--ca-domain-owner` | AWS account ID that owns the domain (when `--type codeartifact`; optional if a profile is configured) |
| `--ca-region` | AWS region (when `--type codeartifact`; optional if a profile is configured) |
| `--ca-profile` | Named AWS CodeArtifact profile to use (when `--type codeartifact`) |

### `vouch init`

Output a shell hook that sets `VOUCH_AUTHENTICATED`, `VOUCH_EMAIL`, and `VOUCH_EXPIRES_IN` on each prompt. Add to your shell profile for ambient session awareness.

```
eval "$(vouch init <SHELL>)"
```

Supported shells: `bash`, `zsh`, `fish`.

---

## Diagnostics

### `vouch doctor`

Run diagnostic checks to verify your Vouch installation and configuration.

```
vouch doctor [--quiet] [--json]
```

| Flag | Description |
|---|---|
| `-q`, `--quiet` | Suppress all output (exit code only) |
| `--json` | Output results as JSON |

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

Supported shells: `bash`, `zsh`, `fish`, `powershell`, `elvish`.

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
