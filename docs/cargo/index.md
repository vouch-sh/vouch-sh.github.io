# Authenticate Cargo to AWS CodeArtifact

> Use Vouch as a Cargo credential provider for AWS CodeArtifact registries — no tokens in .cargo/credentials.toml.

Source: https://vouch.sh/docs/cargo/
Last updated: 2026-10-19

---
Vouch acts as a [Cargo credential provider](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html) for [AWS CodeArtifact](/docs/codeartifact/) Cargo registries. Instead of a plaintext token in `~/.cargo/credentials.toml`, Cargo asks Vouch for a token, and Vouch exchanges your hardware-backed session for a short-lived AWS CodeArtifact authorization token through AWS STS. AWS CodeArtifact is the only Cargo registry that Vouch supports.

&gt; **Known issue:** The Cargo credential provider in current Vouch CLI releases fails the Cargo handshake with `failed to deserialize hello`, so Cargo cannot obtain a token from Vouch for any registry. The fix is tracked in [vouch-sh/vouch#1841](https://github.com/vouch-sh/vouch/issues/1841). This page describes the intended setup.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once:** add `codeartifact:GetAuthorizationToken`, `codeartifact:GetRepositoryEndpoint`, `codeartifact:ReadFromRepository`, and `sts:GetServiceBearerToken` to the Vouch IAM role.
- **Each developer:** `vouch setup codeartifact --tool cargo --repository &lt;REPO&gt;`, then use Cargo with `--registry codeartifact-&lt;REPO&gt;`.
{{&lt; /tldr &gt;}}

## Prerequisites

{{&lt; role admin &gt;}}

Before developers can configure Cargo:

- The **[AWS integration](/docs/aws/)** must be configured (OIDC provider and IAM role)
- An **AWS CodeArtifact domain and Cargo repository** must exist in your AWS account
- The IAM role must have `codeartifact:GetAuthorizationToken`, `codeartifact:GetRepositoryEndpoint`, `codeartifact:ReadFromRepository`, and `sts:GetServiceBearerToken` permissions

Each developer needs **Cargo** installed via [rustup](https://rustup.rs/) (version **1.74** or later, which includes credential provider protocol support).

---

## Step 1 -- Configure the Cargo registry

{{&lt; role developer &gt;}}

Run the AWS CodeArtifact setup command with `--tool cargo`:

```bash
vouch setup codeartifact --tool cargo --repository my-repo [--domain my-domain] [--domain-owner 123456789012] [--region us-east-1] [--domain-profile my-profile]
```

The flags are the same as for every other package manager; see [AWS CodeArtifact](/docs/codeartifact/#step-1--configure-the-vouch-cli) for the full table, including domain profiles and `--profile`.

The command adds a registry named `codeartifact-&lt;repository&gt;` to `~/.cargo/config.toml`:

```toml
[registries.codeartifact-my-repo]
index = &#34;sparse&#43;https://my-domain-123456789012.d.codeartifact.us-east-1.amazonaws.com/cargo/my-repo/&#34;
credential-provider = [&#34;/path/to/vouch&#34;, &#34;credential&#34;, &#34;cargo&#34;, &#34;--&#34;]
```

Vouch is configured only for this registry. Other registries, including crates.io, keep their existing credential providers.

---

## Step 2 -- Use Cargo normally

{{&lt; role developer &gt;}}

{{&lt; session-note &gt;}}

With the registry configured and an active session, Cargo commands work without `cargo login` or token environment variables:

```bash
# Publish a crate to the AWS CodeArtifact registry
cargo publish --registry codeartifact-my-repo

# Add a dependency from the AWS CodeArtifact registry
cargo add my-crate --registry codeartifact-my-repo
```

To depend on crates from the registry, reference it in `Cargo.toml`:

```toml
[dependencies]
my-crate = { version = &#34;1.0&#34;, registry = &#34;codeartifact-my-repo&#34; }
```

---

## How it works

1. **Cargo requests a token** -- Cargo calls the Vouch credential provider when it needs to authenticate to the `codeartifact-&lt;repository&gt;` registry.
2. **OIDC to STS** -- Vouch exchanges your active hardware-backed session for temporary AWS STS credentials via `AssumeRoleWithWebIdentity`.
3. **STS to AWS CodeArtifact** -- Vouch calls `codeartifact:GetAuthorizationToken` with the STS credentials to obtain an AWS CodeArtifact authorization token.
4. **Cargo authenticates** -- Vouch returns the token to Cargo along with its expiry. Cargo sends it to AWS CodeArtifact in the `Authorization` header. The token is never written to disk.

---

## Troubleshooting

### Not authenticated

```
error: failed to get token for registry `codeartifact-my-repo`
```

- Verify you have an active Vouch session: `vouch login`
- Check that the Vouch agent is running: `vouch status`
- Confirm the registry is configured: inspect `~/.cargo/config.toml` and verify the `[registries.codeartifact-my-repo]` section has a `credential-provider` entry pointing to Vouch.
- Confirm the IAM role has the permissions listed under [Prerequisites](#prerequisites).

### Cargo not using Vouch

If Cargo prompts you for a token or uses a different credential provider:

1. Ensure no `CARGO_REGISTRY_TOKEN` or `CARGO_REGISTRIES_*_TOKEN` environment variables are set, as these override credential providers:
   ```bash
   env | grep CARGO_REGISTR
   ```
2. Re-run `vouch setup codeartifact --tool cargo --repository &lt;REPO&gt;` to ensure the configuration is correct.

### Unsupported protocol version

```
error: credential provider `vouch` failed: unsupported credential provider protocol version
```

The Cargo credential provider protocol requires **Cargo 1.74 or later**. Check your Cargo version:

```
cargo --version
```

If your version is older than 1.74, update via rustup:

```
rustup update stable
```

For AWS-side errors (access denied, wrong region, cross-account domains), see [AWS CodeArtifact troubleshooting](/docs/codeartifact/#troubleshooting).
