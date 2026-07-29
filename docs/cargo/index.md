# Authenticate to Private Cargo Registries

> Use Vouch as a Cargo credential provider for private registries — no tokens in .cargo/config.toml.

Source: https://vouch.sh/docs/cargo/
Last updated: 2026-07-04

---
Vouch replaces the plaintext token in `~/.cargo/credentials.toml` with tokens derived from your hardware-backed session -- short-lived, never written to disk, and revoked when your session ends. Vouch implements the [Cargo credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html), so it works with any registry that supports Bearer token authentication.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → this page.
- **Admin, once:** confirm your private registry meets the [registry server requirements](#registry-server-requirements) (Bearer token authentication).
- **Each developer:** `vouch setup cargo --configure`, then `cargo build` and `cargo publish` just work.
{{&lt; /tldr &gt;}}

## Prerequisites

Before configuring the Cargo integration, make sure you have:

- **Cargo** installed via [rustup](https://rustup.rs/) (version **1.74** or later, which includes credential provider protocol support)
- A **private Cargo registry** that supports Bearer token authentication

---

## Step 1 -- Configure Cargo Credential Provider

{{&lt; role developer &gt;}}

Run the setup command to install the Vouch credential provider for Cargo:

```
vouch setup cargo
```

This prints the Cargo configuration that will be added. To apply it automatically:

```
vouch setup cargo --configure
```

To configure a specific named registry:

```
vouch setup cargo --registry my-private-registry --configure
```

The command adds the following to your `~/.cargo/config.toml`:

```toml
[registry]
global-credential-providers = [&#34;vouch&#34;]

[registries.my-private-registry]
index = &#34;sparse&#43;https://cargo.example.com/index/&#34;
credential-provider = [&#34;vouch&#34;]
```

The `global-credential-providers` setting registers Vouch as the default credential provider for all registries. You can also configure it per-registry using the `credential-provider` key under a specific `[registries.*]` section.

---

## Step 2 -- Use Cargo normally

{{&lt; role developer &gt;}}

{{&lt; session-note &gt;}}

With the credential provider configured and an active session, Cargo commands work without any extra flags or manual token management:

```bash
# Build a project that depends on private crates
cargo build

# Publish a crate to your private registry
cargo publish --registry my-private-registry

# Add a dependency from a private registry
cargo add my-crate --registry my-private-registry

# Update dependencies including private ones
cargo update
```

Vouch handles authentication transparently. You do not need to run `cargo login` or set any environment variables.

---

## Private Registry Configuration

To use a private Cargo registry with Vouch, you need to define the registry in your Cargo configuration. Add the following to your project&#39;s `.cargo/config.toml` or your global `~/.cargo/config.toml`:

```toml
[registries.my-private-registry]
index = &#34;sparse&#43;https://cargo.example.com/index/&#34;
credential-provider = [&#34;vouch&#34;]
```

If your `Cargo.toml` references dependencies from the private registry:

```toml
[dependencies]
my-crate = { version = &#34;1.0&#34;, registry = &#34;my-private-registry&#34; }
```

Cargo will automatically call the Vouch credential provider when it needs to fetch or publish crates from this registry.

### Registry server requirements

{{&lt; role admin &gt;}}

The private registry must support:

- **Sparse index protocol** (recommended) or Git index protocol
- **Bearer token authentication** via the `Authorization` HTTP header
- The token format issued by Vouch (a signed JWT)

Consult your registry server&#39;s documentation to confirm Bearer token support.

---

## How it works

When Cargo needs to authenticate to a private registry, it delegates to the Vouch credential provider:

1. **Cargo requests a token** -- Cargo calls the Vouch credential provider binary when it needs to authenticate to a configured private registry.
2. **Vouch exchanges your session** -- The credential provider contacts the Vouch server and exchanges your active hardware-backed session for a registry-scoped Bearer token.
3. **Token derived from your session** -- The token (a signed JWT) is scoped to the specific registry, expires when your session ends, and is never written to disk.
4. **Cargo authenticates** -- The Bearer token is sent to the registry as part of the HTTP request, and the operation proceeds.

---

## Troubleshooting

### Not authenticated

```
error: failed to get token for registry `my-private-registry`
caused by: not logged in to registry `my-private-registry`
```

- Verify you have an active Vouch session: `vouch login`
- Check that the Vouch agent is running: `vouch status`
- Confirm the credential provider is configured for the registry: inspect `~/.cargo/config.toml` and verify the `credential-provider` key is set to `[&#34;vouch&#34;]`.

### Cargo not using Vouch

If Cargo prompts you for a token or uses a different credential provider:

1. Check your Cargo configuration for conflicting credential provider settings:
   ```
   cargo config get registry.global-credential-providers
   ```
2. Ensure no `CARGO_REGISTRY_TOKEN` or `CARGO_REGISTRIES_*_TOKEN` environment variables are set, as these override credential providers:
   ```bash
   env | grep CARGO_REGISTR
   ```
3. Re-run `vouch setup cargo --configure` to ensure the configuration is correct.

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

### Token not cached

If Vouch appears to request a new token for every Cargo operation (causing delays):

- This is expected behavior. Vouch derives tokens from your active session on each request rather than caching them to disk. The overhead is minimal (typically under 100ms).
- If latency is a concern, ensure the Vouch agent is running (`vouch status`), as it keeps your session in memory for fast token derivation.

### Multiple credential providers

If you have multiple credential providers configured and they conflict:

1. Check the provider order in `~/.cargo/config.toml`:
   ```toml
   [registry]
   global-credential-providers = [&#34;vouch&#34;, &#34;cargo:token&#34;]
   ```
   Cargo tries providers in order. Place `vouch` first to ensure it is used before any fallback providers.

2. To use Vouch for only specific registries, remove it from `global-credential-providers` and set it per-registry instead:
   ```toml
   [registries.my-private-registry]
   credential-provider = [&#34;vouch&#34;]
   ```
