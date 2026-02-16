---
title: "Authenticate to Private Cargo Registries"
linkTitle: "Cargo"
description: "Use Vouch as a Cargo credential provider for private registries — no tokens in .cargo/config.toml."
weight: 9
subtitle: "Authenticate to private Cargo registries using Vouch"
params:
  docsGroup: code
---

Cargo's default credential mechanism stores a plaintext token in `~/.cargo/credentials.toml` with no expiration. If that file is committed to a dotfiles repo, included in a backup, or read by malware, the token works forever.

Vouch's credential provider replaces this with tokens derived from your hardware-backed session -- short-lived, never written to disk, and revoked the moment your session ends. After a single `vouch login`, commands like `cargo build`, `cargo publish`, and `cargo add` work seamlessly against private registries.

## How it works

Vouch implements the [Cargo credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html) (RFC 2730 / RFC 3139). When Cargo needs to authenticate to a private registry, it delegates to the Vouch credential provider:

1. **Cargo requests a token** -- Cargo calls the Vouch credential provider binary when it needs to authenticate to a configured private registry.
2. **Vouch exchanges your session** -- The credential provider contacts the Vouch server and exchanges your active hardware-backed session for a registry-scoped Bearer token.
3. **Token derived from your session** -- The token is derived from your Vouch session and is scoped to the specific registry. It is short-lived and never written to disk.
4. **Cargo authenticates** -- The Bearer token is sent to the registry as part of the HTTP request, and the operation proceeds.

Key characteristics:

- **Short-lived tokens** -- Tokens are derived from your Vouch session and expire when the session ends.
- **No stored secrets** -- There are no tokens in `~/.cargo/credentials.toml` or environment variables to manage.
- **Standard protocol** -- Vouch uses Cargo's official credential provider protocol, so it works with any registry that supports Bearer token authentication.

---

## Prerequisites

Before configuring the Cargo integration, make sure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- **Cargo** installed via [rustup](https://rustup.rs/) (version **1.74** or later, which includes credential provider protocol support)
- A **private Cargo registry** that supports Bearer token authentication

---

## Step 1 -- Configure Cargo Credential Provider

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
global-credential-providers = ["vouch"]

[registries.my-private-registry]
index = "sparse+https://cargo.example.com/index/"
credential-provider = ["vouch"]
```

The `global-credential-providers` setting registers Vouch as the default credential provider for all registries. You can also configure it per-registry using the `credential-provider` key under a specific `[registries.*]` section.

---

## Step 2 -- Authenticate

If you have not already logged in today, authenticate with your YubiKey:

```
vouch login
```

Your session lasts for 8 hours. All Cargo operations during that window use the session automatically.

---

## Step 3 -- Use Cargo normally

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

To use a private Cargo registry with Vouch, you need to define the registry in your Cargo configuration. Add the following to your project's `.cargo/config.toml` or your global `~/.cargo/config.toml`:

```toml
[registries.my-private-registry]
index = "sparse+https://cargo.example.com/index/"
credential-provider = ["vouch"]
```

If your `Cargo.toml` references dependencies from the private registry:

```toml
[dependencies]
my-crate = { version = "1.0", registry = "my-private-registry" }
```

Cargo will automatically call the Vouch credential provider when it needs to fetch or publish crates from this registry.

### Registry server requirements

The private registry must support:

- **Sparse index protocol** (recommended) or Git index protocol
- **Bearer token authentication** via the `Authorization` HTTP header
- The token format issued by Vouch (a signed JWT)

Consult your registry server's documentation to confirm Bearer token support.

---

## Troubleshooting

### Not authenticated

```
error: failed to get token for registry `my-private-registry`
caused by: not logged in to registry `my-private-registry`
```

- Verify you have an active Vouch session: `vouch login`
- Check that the Vouch agent is running: `vouch status`
- Confirm the credential provider is configured for the registry: inspect `~/.cargo/config.toml` and verify the `credential-provider` key is set to `["vouch"]`.

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
   global-credential-providers = ["vouch", "cargo:token"]
   ```
   Cargo tries providers in order. Place `vouch` first to ensure it is used before any fallback providers.

2. To use Vouch for only specific registries, remove it from `global-credential-providers` and set it per-registry instead:
   ```toml
   [registries.my-private-registry]
   credential-provider = ["vouch"]
   ```
