---
title: "Authenticate to AWS CodeArtifact without Stored Tokens"
linkTitle: "AWS CodeArtifact"
description: "Pull and publish packages from CodeArtifact using hardware-backed credentials — no token files, no refresh scripts."
weight: 7
subtitle: "Authenticate to AWS CodeArtifact repositories using Vouch"
params:
  docsGroup: code
---

Every package manager has its own credential mechanism -- pip uses `~/.pip/pip.conf` or `PIP_INDEX_URL`, npm uses `.npmrc`, Cargo uses `~/.cargo/credentials.toml`, and Maven uses `settings.xml`. Each requires a different token format and rotation process. Keeping them all current across a growing team is a constant chore.

With [AWS CodeArtifact](https://docs.aws.amazon.com/codeartifact/latest/ug/welcome.html), you can unify these behind IAM, and with Vouch, the IAM credentials are hardware-backed and automatic. After a single `vouch login`, package managers like Cargo, pip, and npm can pull and publish packages from your CodeArtifact repositories without manual token management.

## How it works

1. **Package manager requests a token** -- When a package manager needs to authenticate to a CodeArtifact repository, the Vouch credential helper intercepts the request.
2. **OIDC to STS** -- Vouch exchanges your active hardware-backed session for temporary AWS STS credentials via `AssumeRoleWithWebIdentity`.
3. **STS to CodeArtifact** -- Vouch calls `codeartifact:GetAuthorizationToken` with the STS credentials to obtain a CodeArtifact authorization token.
4. **Package manager authenticates** -- The token is returned to the package manager and used for the current operation. Tokens are short-lived and never written to disk.

---

## Prerequisites

Before configuring the CodeArtifact integration, make sure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- The **[AWS integration](/docs/aws/)** configured (OIDC provider and IAM role)
- An **AWS CodeArtifact domain and repository** in your AWS account
- The IAM role must have `codeartifact:GetAuthorizationToken` and `sts:GetServiceBearerToken` permissions

---

## Step 1 -- Configure the Vouch CLI

Run the setup command to configure Vouch for your CodeArtifact repository:

```bash
vouch setup codeartifact --tool cargo --repository my-repo [--domain my-domain] [--domain-owner 123456789012] [--region us-east-1] [--profile my-profile]
```

| Flag | Description |
|---|---|
| `--tool` | Package manager to configure: `cargo`, `pip`, or `npm` (required) |
| `--repository` | The CodeArtifact repository name (required) |
| `--domain` | The CodeArtifact domain name (optional if a profile is configured) |
| `--domain-owner` | AWS account ID that owns the domain (optional if a profile is configured) |
| `--region` | AWS region (default: `us-east-1`; optional if a profile is configured) |
| `--profile` | Named profile to use or create (see [Profiles](#profiles) below) |

This configures the appropriate credential helper for your package manager and writes the necessary configuration files.

---

## Profiles

Vouch supports named profiles for CodeArtifact, allowing you to store domain, domain owner, and region settings and reuse them across commands. Profiles are stored in `~/.vouch/config.json`.

### Default profile

When you run `vouch setup codeartifact` with `--domain`, `--domain-owner`, and `--region`, these values are saved to the default profile. Subsequent commands can omit these flags:

```bash
# First time: specify all values (saved to default profile)
vouch setup codeartifact --tool cargo --domain my-domain --domain-owner 123456789012 --repository my-repo --region us-east-1

# Later: only --tool and --repository are needed
vouch setup codeartifact --tool pip --repository my-pypi-repo
```

### Named profiles

Use `--profile` to create and manage separate configurations for different CodeArtifact domains or accounts:

```bash
# Create a profile for the shared artifacts account
vouch setup codeartifact --tool cargo --domain shared-packages --domain-owner 111111111111 --repository cargo-store --profile shared

# Create a profile for the team account
vouch setup codeartifact --tool cargo --domain team-packages --domain-owner 222222222222 --repository team-cargo --profile team
```

Named profiles are referenced by other commands using the `--profile` flag.

---

## Supported package managers

| Package Manager | Protocol | Authentication Method | Token Model |
|---|---|---|---|
| **Cargo** | `sparse+https` | Bearer token via credential provider | Dynamic (fetched on demand) |
| **pip** | HTTPS | Token embedded in index URL | Dynamic (fetched on demand) |
| **npm** | HTTPS | Bearer token via `.npmrc` | Static (embedded in `.npmrc`, ~12h expiry) |

**Dynamic tokens** (Cargo, pip) are fetched transparently on each operation and do not expire during normal use. **Static tokens** (npm) are written to `.npmrc` during setup and expire after approximately 12 hours. When an npm token expires, re-run `vouch setup codeartifact --tool npm --repository <REPO>` to refresh it.

---

## Step 2 -- Authenticate

If you have not already logged in today, authenticate with your YubiKey:

```
vouch login
```

Your session lasts for 8 hours. All CodeArtifact operations during that window use the session automatically.

---

## Step 3 -- Use your package manager normally

### Cargo

```bash
# Build a project that depends on private crates
cargo build

# Publish a crate to your CodeArtifact registry
cargo publish --registry my-codeartifact-registry
```

Cargo tokens are fetched dynamically on each operation via the credential provider. No token refresh is needed.

### pip

```bash
# Install a package from your CodeArtifact repository
pip install my-package --index-url https://my-domain-123456789012.d.codeartifact.us-east-1.amazonaws.com/pypi/my-repo/simple/

# Install from requirements.txt
pip install -r requirements.txt
```

pip tokens are fetched dynamically by embedding the credential helper in the index URL. No token refresh is needed.

### npm

```bash
# Install packages
npm install

# Publish a package
npm publish
```

npm uses a static token written to `.npmrc`. If you see authentication errors after ~12 hours, re-run `vouch setup codeartifact --tool npm --repository <REPO>` to refresh the token.

---

## Environment variables

You can inject a `CODEARTIFACT_AUTH_TOKEN` environment variable into your shell or a subprocess using `vouch env` or `vouch exec`. This is useful for tools that read the token from the environment (such as Maven or custom scripts).

### `vouch env`

Output the token as a shell export statement:

```bash
eval "$(vouch env --type codeartifact [--ca-domain <DOMAIN>] [--ca-domain-owner <ACCOUNT_ID>] [--ca-region <REGION>] [--ca-profile <PROFILE>] [--shell <SHELL>])"
```

This sets `CODEARTIFACT_AUTH_TOKEN` in your current shell.

### `vouch exec`

Run a command with the token injected:

```bash
vouch exec --type codeartifact [--ca-domain <DOMAIN>] [--ca-domain-owner <ACCOUNT_ID>] [--ca-region <REGION>] [--ca-profile <PROFILE>] -- mvn deploy
```

| Flag | Description |
|---|---|
| `--ca-domain` | CodeArtifact domain name (optional if a profile is configured) |
| `--ca-domain-owner` | AWS account ID that owns the domain (optional if a profile is configured) |
| `--ca-region` | AWS region (optional if a profile is configured) |
| `--ca-profile` | Named CodeArtifact profile to use |

---

## Cross-partition support

Vouch supports CodeArtifact across all AWS partitions:

| Partition | Region Examples |
|---|---|
| **Standard** (`aws`) | `us-east-1`, `eu-west-1`, `ap-southeast-1` |
| **China** (`aws-cn`) | `cn-north-1`, `cn-northwest-1` |
| **GovCloud** (`aws-us-gov`) | `us-gov-west-1`, `us-gov-east-1` |

Use the `--region` flag during setup to configure the appropriate partition.

---

## Troubleshooting

### "Access denied" when fetching packages

- Verify your IAM role has the following permissions:
  - `codeartifact:GetAuthorizationToken`
  - `codeartifact:GetRepositoryEndpoint`
  - `codeartifact:ReadFromRepository`
  - `sts:GetServiceBearerToken`
- Confirm the CodeArtifact domain and repository names are correct.
- Check that you have an active Vouch session: `vouch login`.

### "Token is expired"

- For **npm**: Re-run `vouch setup codeartifact --tool npm --repository <REPO>` to refresh the static token in `.npmrc`.
- For **Cargo/pip**: Run `vouch login` to refresh your session. Dynamic tokens are fetched on demand, so expiry usually indicates the Vouch session itself has ended.

### Wrong domain or repository

- Run `vouch setup codeartifact` again with the correct `--tool` and `--repository` flags.
- If using profiles, check `~/.vouch/config.json` for the stored domain and region values.
- Check your package manager's configuration files for conflicting settings.

### Package manager not using Vouch

- Ensure no environment variables (e.g., `CODEARTIFACT_AUTH_TOKEN`) are overriding the credential helper.
- Verify the package manager configuration points to the correct CodeArtifact endpoint.

---

## Maven

For Maven projects, use `vouch credential codeartifact` or `vouch exec` to obtain a token:

```bash
# Option 1: Set the token in your shell
export CODEARTIFACT_AUTH_TOKEN=$(vouch credential codeartifact)

# Option 2: Use vouch exec to inject the token into Maven
vouch exec --type codeartifact -- mvn deploy -s settings.xml
```

If you need to specify the domain explicitly:

```bash
export CODEARTIFACT_AUTH_TOKEN=$(vouch credential codeartifact --domain my-domain --domain-owner 123456789012)
```

In your `settings.xml`, reference the environment variable as the password:

```xml
<server>
  <id>codeartifact</id>
  <username>aws</username>
  <password>${env.CODEARTIFACT_AUTH_TOKEN}</password>
</server>
```

---

## Cross-Account Access

If your CodeArtifact domain is in a different AWS account, use named profiles to manage access:

```bash
# Set up a Vouch AWS profile for the artifacts account
vouch setup aws \
  --role arn:aws:iam::ARTIFACTS_ACCOUNT:role/CodeArtifactReader \
  --profile vouch-artifacts

# Create a CodeArtifact profile that uses the artifacts account
vouch setup codeartifact \
  --tool npm \
  --domain shared-packages \
  --domain-owner ARTIFACTS_ACCOUNT \
  --repository npm-store \
  --profile artifacts

# Use the profile when fetching credentials
vouch credential codeartifact --profile artifacts
```

---

## Token Lifetime

CodeArtifact authorization tokens are valid for up to **12 hours** by default. For Cargo and pip, Vouch fetches tokens dynamically on each operation, so expiry is transparent. For npm, the token is embedded in `.npmrc` and must be refreshed by re-running `vouch setup codeartifact --tool npm` when it expires. If your Vouch session (8 hours) has expired, run `vouch login` first.
