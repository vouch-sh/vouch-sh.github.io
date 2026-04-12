# Device Authorization (CLI)

> Integrate Vouch authentication into native desktop applications and CLI tools using the Device Authorization Grant.

Source: https://vouch.sh/docs/applications/device-authorization/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

For native desktop applications and CLI tools that cannot open a browser redirect, use the [Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628) (RFC 8628). This flow displays a URL and code that the user enters in a browser on any device.

## How it works

1. Your application requests a device code from `POST /oauth/device/code`.
2. The user opens the verification URL in a browser and enters the displayed code.
3. The user authenticates with their YubiKey in the browser.
4. Your application polls `POST /oauth/token` with the `device_code` until the user completes authentication.
5. The token response includes an access token with hardware attestation claims (`hardware_verified`, `hardware_aaguid`).

Key details:

- Handle `authorization_pending` (keep polling), `slow_down` (increase interval), and `expired_token` (request a new code) responses
- The default device code expiration is 10 minutes
- No client secret is needed (public client)
- [Rich Authorization Requests](/docs/applications/#rich-authorization-requests) are supported via `authorization_details` in the device code request

## Examples

Working examples are available in the examples repository:

- **[native/python](https://github.com/vouch-sh/examples/tree/main/native/python)** — Python device flow with polling and hardware claim extraction
- **[native/node](https://github.com/vouch-sh/examples/tree/main/native/node)** — Node.js device flow with polling and hardware claim extraction
- **[native/rust](https://github.com/vouch-sh/examples/tree/main/native/rust)** — Rust device flow with the [`openidconnect`](https://crates.io/crates/openidconnect) crate
