# Vouch CLI Reference

> Complete command reference for the Vouch CLI — login, credentials, setup, and configuration.

Source: https://vouch.sh/docs/cli-reference/
Last updated: 2026-07-04

---
This page documents all available Vouch CLI commands. For installation instructions, see [Getting Started](/docs/getting-started/).

## Quick links

- [Global flags](#global-flags)
- [Authentication](#authentication)
- [AWS](#aws)
- [Setup](#setup)
- [Credentials](#credentials)
- [Key management](#key-management)
- [Environment](#environment)
- [Device posture](#device-posture)
- [Diagnostics](#diagnostics)
- [Exit codes](#exit-codes)
- [Binary download verification](#binary-download-verification)

---

## Global flags

These flags are available on all commands.

| Flag | Description |
|---|---|
| `--server &lt;URL&gt;` | Vouch server URL (also settable via `VOUCH_SERVER` environment variable). Saved locally after enrollment. |
| `-v`, `--verbose` | Enable debug logging |
| `--color &lt;MODE&gt;` | Control color output: `auto` (default), `always`, or `never` |

### Configuration file

The Vouch CLI follows the [XDG Base Directory specification](https://specifications.freedesktop.org/basedir-spec/latest/) on all platforms, including macOS: configuration lives at `~/.config/vouch/config.json` (`$XDG_CONFIG_HOME`), session state (cookie, audit log) under `~/.local/state/vouch/` (`$XDG_STATE_HOME`), and the agent sockets under `$XDG_RUNTIME_DIR/vouch/` (falling back to `~/.cache/vouch/` where `XDG_RUNTIME_DIR` is unset). The configuration file is created automatically during enrollment and contains the server URL and session state.

| Field | Description |
|---|---|
| `server_url` | Vouch server URL |
| `token` | Current session token (set by `vouch login`) |

**Precedence:** CLI flags (`--server`) override the `VOUCH_SERVER` environment variable, which overrides the config file value.

On Unix, the config file must have restrictive permissions (`0600`). The CLI rejects files that are group- or world-readable.

---

## Authentication

### `vouch enroll`

Register your YubiKey with a Vouch server and link it to your identity.

```
vouch enroll --server &lt;SERVER_URL&gt;
```

You only need to enroll once per YubiKey.

### `vouch login`

Authenticate with your YubiKey and start an 8-hour session.

```
vouch login [--timeout &lt;SECONDS&gt;]
```

| Flag | Description |
|---|---|
| `--timeout` | Timeout in seconds for YubiKey detection (default: `60`, use `0` for no timeout) |

During login, the CLI automatically collects [device posture signals](/docs/device-posture/) and sends them to the server for policy evaluation. After login, all credential helpers use the session automatically. Run this once at the start of each workday.

### `vouch logout`

End the current session and clear all cached credentials.

```
vouch logout
```

### `vouch status`

Display the current session status, including remaining session time and active integrations (SSH, AWS, SSM, Git, Docker, Cargo, Claude, OpenAI).

```
vouch status [--format &lt;FORMAT&gt;]
```

| Flag | Description |
|---|---|
| `--format` | Output format: `human` (default), `json`, or `shell`. The `shell` format outputs key=value pairs suitable for `eval`. |

---

## AWS

Commands for authenticating with AWS IAM Identity Center and discovering available accounts and roles.

### `vouch aws login`

Authenticate with AWS IAM Identity Center SSO.

```
vouch aws login [--sso-session &lt;NAME&gt;]
```

| Flag | Description |
|---|---|
| `--sso-session` | Named SSO session from `~/.aws/config` (optional; uses default if not specified) |

### `vouch aws accounts`

List AWS accounts available through IAM Identity Center.

```
vouch aws accounts [--sso-session &lt;NAME&gt;] [--json]
```

| Flag | Description |
|---|---|
| `--sso-session` | Named SSO session (optional) |
| `--json` | Output as JSON |

### `vouch aws roles`

List IAM roles available in an AWS account through IAM Identity Center.

```
vouch aws roles [--sso-session &lt;NAME&gt;] [--account &lt;ACCOUNT_ID&gt;] [--json]
```

| Flag | Description |
|---|---|
| `--sso-session` | Named SSO session (optional) |
| `--account` | AWS account ID to query (optional; lists roles across all accounts if not specified) |
| `--json` | Output as JSON |

See [Multi-Account AWS Strategy](/docs/aws-multi-account/) for full details.

### `vouch aws console`

Open the AWS Management Console in your browser.

```
vouch aws console [--role &lt;ROLE_ARN&gt;]
```

| Flag | Description |
|---|---|
| `--role` | AWS IAM role ARN to assume (auto-detected from `~/.aws/config` if not specified) |

This uses your active Vouch session to obtain temporary STS credentials, exchanges them for a federation sign-in token, and opens the console in your default browser.

---

## Setup

Setup commands configure credential helpers for each integration. Run these once per machine.

### `vouch setup aws`

Configure the AWS credential process for an IAM role, or auto-discover accounts and roles from IAM Identity Center.

```
vouch setup aws (--role &lt;ROLE_ARN&gt; | --discover) [--profile &lt;PROFILE&gt;] [--prefix &lt;PREFIX&gt;] [--region &lt;REGION&gt;]
```

| Flag | Description |
|---|---|
| `--role` | The IAM role ARN to assume (required unless `--discover` is used) |
| `--discover` | Auto-discover accounts and roles from IAM Identity Center SSO (alternative to `--role`) |
| `--profile` | AWS profile name to configure (default: `vouch`; additional profiles auto-name as `vouch-2`, `vouch-3`, etc.) |
| `--prefix` | Prefix for auto-generated profile names when using `--discover` |
| `--region` | AWS region to set in the profile |

See [AWS Integration](/docs/aws/) for full details.

### `vouch setup ssh`

Configure the SSH client to use the Vouch agent for certificate authentication.

```
vouch setup ssh [--hosts &lt;PATTERN&gt;]
```

| Flag | Description |
|---|---|
| `--hosts` | Host patterns to trust with this CA (e.g., `*.example.com`). If specified, adds an entry to `~/.ssh/known_hosts`. |

See [SSH Certificates](/docs/ssh/) for full details.

### `vouch setup github`

Configure Git to use Vouch as the credential helper for GitHub.

```
vouch setup github [--host &lt;HOST&gt;] [--configure]
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
vouch setup cargo [--registry &lt;NAME&gt;] [--configure]
```

| Flag | Description |
|---|---|
| `--registry` | Name of the Cargo registry to configure |
| `--configure` | Apply the configuration automatically |

See [Cargo Integration](/docs/cargo/) for full details.

### `vouch setup codeartifact`

Configure a package manager for an AWS CodeArtifact repository.

```
vouch setup codeartifact --tool &lt;TOOL&gt; --repository &lt;REPO&gt; [--domain &lt;DOMAIN&gt;] [--domain-owner &lt;ACCOUNT_ID&gt;] [--region &lt;REGION&gt;] [--profile &lt;PROFILE&gt;]
```

| Flag | Description |
|---|---|
| `--tool` | Package manager to configure: `cargo`, `pip`, `npm`, `pnpm`, or `uv` (required) |
| `--repository` | The AWS CodeArtifact repository name (required) |
| `--domain` | The AWS CodeArtifact domain name (optional if a profile is configured) |
| `--domain-owner` | AWS account ID that owns the domain (optional if a profile is configured) |
| `--region` | AWS region (optional if a profile is configured) |
| `--profile` | Named AWS CodeArtifact profile to use or create (stores domain/owner/region for reuse) |

See [AWS CodeArtifact](/docs/codeartifact/) for full details.

### `vouch setup codecommit`

Configure Git to use Vouch as the credential helper for AWS CodeCommit.

```
vouch setup codecommit [--region &lt;REGION&gt;] [--profile &lt;PROFILE&gt;] [--configure]
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
vouch setup eks --cluster &lt;CLUSTER_NAME&gt; [--region &lt;REGION&gt;] [--profile &lt;PROFILE&gt;] [--kubeconfig &lt;PATH&gt;]
```

| Flag | Description |
|---|---|
| `--cluster` | The EKS cluster name (required) |
| `--region` | AWS region (auto-detected from AWS profile or environment if not specified) |
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--kubeconfig` | Path to kubeconfig file (defaults to `~/.kube/config`) |

See [Amazon EKS](/docs/eks/) for full details.

### `vouch setup k8s`

Configure kubectl to use Vouch for Kubernetes OIDC authentication. This works with any Kubernetes distribution that supports OIDC (self-hosted, GKE, AKS, k3s, etc.).

```
vouch setup k8s --cluster &lt;NAME&gt; --server &lt;URL&gt; [--certificate-authority &lt;PATH&gt;] [--audience &lt;AUDIENCE&gt;] [--kubeconfig &lt;PATH&gt;]
```

| Flag | Description |
|---|---|
| `--cluster` | Kubernetes cluster name (required) |
| `--server` | Kubernetes API server URL, e.g., `https://k8s.example.com:6443` (required) |
| `--certificate-authority` | Path to the cluster&#39;s CA certificate file (PEM format) |
| `--audience` | OIDC audience — must match `--oidc-client-id` on the API server (default: `kubernetes`) |
| `--kubeconfig` | Path to kubeconfig file (defaults to `~/.kube/config`) |

See [Kubernetes](/docs/kubernetes/) for full details.

### `vouch setup anthropic`

Configure Anthropic (Claude) Workload Identity Federation. Persists federation parameters to `~/.config/vouch/config.json` and auto-merges `~/.claude/settings.json` to set `apiKeyHelper` → `vouch credential anthropic` plus `env.CLAUDE_CODE_API_KEY_HELPER_TTL_MS` so Claude Code re-runs the helper before the token expires.

```
vouch setup anthropic --federation-rule-id &lt;ID&gt; --organization-id &lt;UUID&gt; --service-account-id &lt;ID&gt; --workspace-id &lt;ID&gt; [--audience &lt;AUD&gt;] [--token-endpoint &lt;URL&gt;] [--force]
```

| Flag | Description |
|---|---|
| `--federation-rule-id` | Anthropic federation rule ID (`fdrl_...`) (required) |
| `--organization-id` | Anthropic organization ID (UUID) (required) |
| `--service-account-id` | Anthropic service account ID (`svac_...`) (required) |
| `--workspace-id` | Anthropic workspace ID (`wrkspc_...`) (required) |
| `--audience` | `aud` claim to request on the assertion (optional; most federation rules match on `sub` alone) |
| `--token-endpoint` | Override the Anthropic token endpoint (defaults to Anthropic&#39;s public endpoint) |
| `--force` | Overwrite an existing Claude Code `apiKeyHelper` configuration |

See [Claude &amp; OpenAI APIs](/docs/ai-api-keys/) for full details.

### `vouch setup openai`

Configure OpenAI Workload Identity Federation. Persists federation parameters to `~/.config/vouch/config.json` and auto-merges `~/.codex/config.toml` to add a `[model_providers.vouch]` block (with a refreshing `auth` command) and set the top-level `model_provider = &#34;vouch&#34;`.

```
vouch setup openai --identity-provider-id &lt;ID&gt; --service-account-id &lt;ID&gt; [--audience &lt;AUD&gt;] [--token-endpoint &lt;URL&gt;] [--force]
```

| Flag | Description |
|---|---|
| `--identity-provider-id` | OpenAI Workload Identity Provider ID for the Vouch issuer (required) |
| `--service-account-id` | OpenAI service account ID (required) |
| `--audience` | `aud` claim to request on the assertion (matches the audience OpenAI configured for the Vouch issuer) |
| `--token-endpoint` | Override the OpenAI token endpoint (defaults to OpenAI&#39;s public endpoint) |
| `--force` | Switch Codex&#39;s top-level `model_provider` away from another provider already in place |

OpenAI must onboard the Vouch issuer as a workload identity provider before this works — custom OIDC issuers are not self-service on OpenAI&#39;s side. See [Claude &amp; OpenAI APIs](/docs/ai-api-keys/) for full details.

### `vouch setup ssm`

Configure SSH to use AWS Systems Manager Session Manager as a proxy for connections to EC2 and managed instances.

```
vouch setup ssm [--profile &lt;PROFILE&gt;] [--region &lt;REGION&gt;] [--hosts &lt;HOSTS&gt;] [--force]
```

| Flag | Description |
|---|---|
| `--profile` | AWS profile to use (defaults to auto-detected vouch profile) |
| `--region` | AWS region to use in the ProxyCommand |
| `--hosts` | Host patterns to match (default: `i-* mi-*`) |
| `--force` | Overwrite any existing SSM configuration in `~/.ssh/config` |

See [AWS Systems Manager](/docs/ssm/) for full details.

---

## Credentials

Credential commands obtain service-specific credentials from your active session. These are typically called automatically by credential helpers, but can be run manually for debugging.

### `vouch credential aws`

Obtain temporary AWS STS credentials.

```
vouch credential aws --role &lt;ROLE_ARN&gt;
```

| Flag | Description |
|---|---|
| `--role` | The IAM role ARN to assume (required) |

### `vouch credential ssh`

Obtain an SSH certificate from the Vouch server.

```
vouch credential ssh [--key &lt;PATH&gt;]
```

| Flag | Description |
|---|---|
| `--key` | Path to SSH private key (default: `~/.ssh/id_ed25519_vouch`) |

### `vouch credential codeartifact`

Obtain an AWS CodeArtifact authorization token.

```
vouch credential codeartifact [--domain &lt;DOMAIN&gt;] [--domain-owner &lt;ACCOUNT_ID&gt;] [--region &lt;REGION&gt;] [--profile &lt;PROFILE&gt;]
```

| Flag | Description |
|---|---|
| `--domain` | The AWS CodeArtifact domain name (optional if a profile is configured) |
| `--domain-owner` | AWS account ID that owns the domain (optional if a profile is configured) |
| `--region` | AWS region (optional if a profile is configured) |
| `--profile` | Named AWS CodeArtifact profile to use |

### `vouch credential rds`

Generate an RDS IAM authentication token for database connections. The token is valid for 15 minutes.

```
vouch credential rds --hostname &lt;HOSTNAME&gt; --username &lt;USERNAME&gt; [--port &lt;PORT&gt;] [--region &lt;REGION&gt;] [--role &lt;ROLE&gt;]
```

| Flag | Description |
|---|---|
| `--hostname` | RDS instance hostname (required) |
| `--username` | Database username (required) |
| `--port` | Database port (default: `5432`) |
| `--region` | AWS region (auto-detected if not specified) |
| `--role` | AWS IAM role ARN (auto-detected from vouch profile if not specified) |

Example:

```bash
TOKEN=$(vouch credential rds \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --username mydbuser)

PGPASSWORD=&#34;$TOKEN&#34; psql -h mydb.cluster-abc123.us-east-1.rds.amazonaws.com -U mydbuser -d mydb &#34;sslmode=require&#34;
```

### `vouch credential redshift`

Generate temporary credentials for Amazon Redshift. Supports both provisioned clusters and Redshift Serverless workgroups.

```
vouch credential redshift (--cluster-id &lt;ID&gt; | --workgroup &lt;NAME&gt;) [--db-name &lt;NAME&gt;] [--region &lt;REGION&gt;] [--role &lt;ROLE&gt;] [--duration &lt;SECONDS&gt;]
```

| Flag | Description |
|---|---|
| `--cluster-id` | Redshift provisioned cluster identifier (mutually exclusive with `--workgroup`) |
| `--workgroup` | Redshift Serverless workgroup name (mutually exclusive with `--cluster-id`) |
| `--db-name` | Database name (optional) |
| `--region` | AWS region (auto-detected if not specified) |
| `--role` | AWS IAM role ARN (auto-detected from vouch profile if not specified) |
| `--duration` | Credential duration in seconds, 900--3600 (provisioned clusters only, default: `900`) |

Examples:

```bash
# Provisioned cluster
vouch credential redshift --cluster-id my-cluster --db-name mydb

# Serverless workgroup
vouch credential redshift --workgroup my-workgroup --db-name mydb
```

### `vouch credential k8s`

Obtain an OIDC token for Kubernetes authentication. Outputs an [`ExecCredential`](https://kubernetes.io/docs/reference/config-api/client-authentication.v1/) JSON object for use as a kubectl exec-based credential plugin.

```
vouch credential k8s --cluster &lt;NAME&gt; [--audience &lt;AUDIENCE&gt;]
```

| Flag | Description |
|---|---|
| `--cluster` | Kubernetes cluster name — used as cache key (required) |
| `--audience` | OIDC audience — must match `--oidc-client-id` on the API server (default: `kubernetes`) |

### `vouch credential anthropic`

Obtain a short-lived Anthropic (Claude) API token via Workload Identity Federation ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523) `jwt-bearer` grant). Requires `vouch setup anthropic` and an active session.

```
vouch credential anthropic
```

Prints a bare `sk-ant-oat01-...` token to stdout with no trailing newline — designed to be invoked by Claude Code&#39;s `apiKeyHelper`. The token is cached until just before its expiry; subsequent invocations within that window return the cached value.

See [Claude &amp; OpenAI APIs](/docs/ai-api-keys/) for full details.

### `vouch credential openai`

Obtain a short-lived OpenAI API token via Workload Identity Federation ([RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) token-exchange grant). Requires `vouch setup openai`, an active session, and that OpenAI has onboarded the Vouch issuer.

```
vouch credential openai
```

Prints a bare OpenAI access token to stdout with no trailing newline — designed to be invoked by the OpenAI Codex CLI as a `[model_providers.*.auth]` command with `refresh_interval_ms`. The token is cached until just before its expiry.

See [Claude &amp; OpenAI APIs](/docs/ai-api-keys/) for full details.

### `vouch credential token`

Print the raw session access token to stdout for use with curl or other tools.

```
vouch credential token
```

Example:

```bash
curl -H &#34;Authorization: Bearer $(vouch credential token)&#34; https://api.example.com/endpoint
```

---

## Key management

### `vouch register`

Register an additional YubiKey with your account.

```
vouch register [--name &lt;NAME&gt;] [--timeout &lt;SECONDS&gt;]
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
vouch keys remove &lt;KEY_ID&gt; [--force]
```

| Flag | Description |
|---|---|
| `-f`, `--force` | Skip the confirmation prompt |

### `vouch keys rename`

Rename a registered security key.

```
vouch keys rename &lt;KEY_ID&gt; &lt;NEW_NAME&gt;
```

---

## Environment

Both `vouch exec` and `vouch env` inject credentials as environment variables. They accept the same type-specific flags and set the same variables -- `exec` runs a command with the variables injected, while `env` outputs shell export statements for use with `eval`.

### Shared flags

| Flag | Description |
|---|---|
| `--type` | Credential type: `aws`, `github`, `codeartifact`, `rds`, or `redshift` (required) |
| `--role` | AWS IAM role ARN (required when `--type aws`) |
| `--codeartifact-domain` | AWS CodeArtifact domain name (when `--type codeartifact`; optional if a profile is configured) |
| `--codeartifact-domain-owner` | AWS account ID that owns the domain (when `--type codeartifact`; optional if a profile is configured) |
| `--codeartifact-region` | AWS region (when `--type codeartifact`; optional if a profile is configured) |
| `--codeartifact-profile` | Named AWS CodeArtifact profile to use (when `--type codeartifact`) |
| `--rds-hostname` | RDS instance hostname (required when `--type rds`) |
| `--rds-username` | Database username (required when `--type rds`) |
| `--rds-port` | Database port (when `--type rds`, default: `5432`) |
| `--rds-region` | AWS region (when `--type rds`; auto-detected if not specified) |
| `--redshift-cluster-id` | Redshift provisioned cluster identifier (when `--type redshift`; mutually exclusive with `--redshift-workgroup`) |
| `--redshift-workgroup` | Redshift Serverless workgroup name (when `--type redshift`; mutually exclusive with `--redshift-cluster-id`) |
| `--redshift-db-name` | Database name (when `--type redshift`) |
| `--redshift-duration` | Credential duration in seconds, 900--3600 (when `--type redshift`, provisioned clusters only, default: `900`) |
| `--redshift-region` | AWS region (when `--type redshift`; auto-detected if not specified) |

### Environment variables by type

| Type | Variables |
|---|---|
| `aws` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` |
| `github` | `GITHUB_TOKEN`, `GH_TOKEN` |
| `codeartifact` | `CODEARTIFACT_AUTH_TOKEN` |
| `rds` | `PGPASSWORD`, `PGHOST`, `PGPORT`, `PGUSER`, `PGSSLMODE` |
| `redshift` | `PGPASSWORD`, `PGUSER`, `PGSSLMODE` |

### `vouch exec`

Run a command with Vouch credentials injected as environment variables.

```
vouch exec --type &lt;TYPE&gt; [FLAGS...] -- &lt;COMMAND&gt; [ARGS...]
```

Examples:

```bash
# AWS credentials
vouch exec --type aws --role arn:aws:iam::123456789012:role/VouchDeveloper -- terraform plan

# AWS CodeArtifact token
vouch exec --type codeartifact -- mvn deploy -s settings.xml

# RDS PostgreSQL — connect with psql, no manual token handling
vouch exec --type rds \
  --rds-hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --rds-username mydbuser \
  -- psql -d mydb

# Redshift provisioned cluster
vouch exec --type redshift \
  --redshift-cluster-id my-cluster \
  --redshift-db-name mydb \
  -- psql -h my-cluster.abc123.us-east-1.redshift.amazonaws.com -p 5439

# Redshift Serverless
vouch exec --type redshift \
  --redshift-workgroup my-workgroup \
  -- psql -h my-workgroup.123456789012.us-east-1.redshift-serverless.amazonaws.com -p 5439
```

### `vouch env`

Output credential environment variables for use with `eval`.

```
eval &#34;$(vouch env --type &lt;TYPE&gt; [--shell &lt;SHELL&gt;] [FLAGS...])&#34;
```

| Flag | Description |
|---|---|
| `--shell` | Shell syntax: `bash` or `fish` (default: `bash`). The `bash` syntax also works for zsh. |

Examples:

```bash
# RDS PostgreSQL
eval &#34;$(vouch env --type rds \
  --rds-hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --rds-username mydbuser)&#34;
psql -d mydb

# Redshift
eval &#34;$(vouch env --type redshift \
  --redshift-cluster-id my-cluster \
  --redshift-db-name mydb)&#34;
psql -h my-cluster.abc123.us-east-1.redshift.amazonaws.com -p 5439
```

### `vouch init`

Output a shell hook that sets `VOUCH_AUTHENTICATED`, `VOUCH_EMAIL`, and `VOUCH_EXPIRES_IN` on each prompt. Add to your shell profile for ambient session awareness.

```
eval &#34;$(vouch init &lt;SHELL&gt;)&#34;
```

Supported shells: `bash`, `zsh`, `fish`.

---

## Device posture

### `vouch posture`

Display the security posture signals detected on the current machine. This shows the same data that is sent to the server during `vouch login` for policy evaluation.

```
vouch posture [--format &lt;FORMAT&gt;]
```

| Flag | Description |
|---|---|
| `--format` | Output format: `text` (default) or `json`. The `json` format outputs the exact `authorization_details` payload sent during login. |

This command does not require an active session — use it to verify device posture at any time.

See [Device Posture Policies](/docs/device-posture/) for full details.

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
- Integration configurations (SSH, AWS, SSM, EKS, Git, Docker, Cargo)
- Claude / OpenAI federation (cross-checks `~/.config/vouch/config.json` against Claude Code&#39;s `apiKeyHelper` and Codex&#39;s `model_provider`)

### `vouch completions`

Generate shell completion scripts.

```
vouch completions &lt;SHELL&gt;
```

Supported shells: `bash`, `zsh`, `fish`, `powershell`, `elvish`.

Example:

```bash
# Add to your ~/.zshrc
eval &#34;$(vouch completions zsh)&#34;
```

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General error |
| `2` | Not authenticated (session expired or missing) |
| `3` | Hardware key not detected |
| `4` | Network or server unreachable |
| `5` | Permission denied |
| `6` | Configuration error |

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
