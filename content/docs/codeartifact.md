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
vouch setup codeartifact --domain my-domain --repository my-repo
```

To specify a region:

```bash
vouch setup codeartifact --domain my-domain --repository my-repo --region us-east-1
```

This configures the appropriate credential helper for your package manager and writes the necessary configuration files.

---

## Supported package managers

| Package Manager | Protocol | Authentication Method |
|---|---|---|
| **Cargo** | `sparse+https` | Bearer token via credential provider |
| **pip** | HTTPS | Token embedded in index URL |
| **npm** | HTTPS | Bearer token via `.npmrc` |

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

### pip

```bash
# Install a package from your CodeArtifact repository
pip install my-package --index-url https://my-domain-123456789012.d.codeartifact.us-east-1.amazonaws.com/pypi/my-repo/simple/

# Install from requirements.txt
pip install -r requirements.txt
```

### npm

```bash
# Install packages
npm install

# Publish a package
npm publish
```

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

- Run `vouch login` to refresh your session. CodeArtifact tokens are derived from your Vouch session and expire when the session ends.

### Wrong domain or repository

- Run `vouch setup codeartifact` again with the correct `--domain` and `--repository` flags.
- Check your package manager's configuration files for conflicting settings.

### Package manager not using Vouch

- Ensure no environment variables (e.g., `CODEARTIFACT_AUTH_TOKEN`) are overriding the credential helper.
- Verify the package manager configuration points to the correct CodeArtifact endpoint.

---

## Maven

For Maven projects, use `aws codeartifact get-authorization-token` to generate a token and reference it in your `settings.xml`:

```bash
# Get an auth token for Maven
export CODEARTIFACT_AUTH_TOKEN=$(aws codeartifact \
  get-authorization-token \
  --domain my-domain \
  --domain-owner 123456789012 \
  --query authorizationToken \
  --output text \
  --profile vouch)

# Use the token in your settings.xml or pass via CLI
mvn deploy -s settings.xml
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

If your CodeArtifact domain is in a different AWS account, configure a separate Vouch profile with a role in that account:

```bash
# Set up a second profile for the artifacts account
vouch setup aws \
  --role arn:aws:iam::ARTIFACTS_ACCOUNT:role/CodeArtifactReader \
  --profile vouch-artifacts

# Use it when logging in to CodeArtifact
aws codeartifact login \
  --tool npm \
  --domain shared-packages \
  --domain-owner ARTIFACTS_ACCOUNT \
  --repository npm-store \
  --profile vouch-artifacts
```

---

## Token Lifetime

CodeArtifact authorization tokens are valid for up to **12 hours** by default. When the token expires, re-run `aws codeartifact login` to get a fresh token. You can set a longer duration with `--duration-seconds 43200` (12 hours max). If your Vouch session (8 hours) has also expired, run `vouch login` first.
