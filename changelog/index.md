# Changelog

> Major features and improvements in recent Vouch releases: OIDC role pinning, per-organization issuers, post-quantum TLS, AWS setup redesign, internationalization, mandatory HTTP message signatures, and more.

Source: https://vouch.sh/changelog/
Last updated: 2026-07-14

---
Highlights from recent Vouch releases. For the complete list of changes in every
release — including bug fixes, dependency updates, and internal refactoring —
see the [GitHub releases page](https://github.com/vouch-sh/vouch/releases).

## [v2026.7.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.7.2) — July 14, 2026

- **OIDC role pinning for AWS** — each token now carries the target role ARN in
  the `https://aws.amazon.com/roles` claim, and AWS STS enforces it: a leaked
  token cannot be exchanged for any role other than the one it was minted for.
  Trust policies can make pinning mandatory with the `sts:RoleAuthorizedByIdp`
  condition key.
- **Per-organization OIDC issuers** — each organization gets its own subdomain
  issuer for AWS federation, with per-org signing keys and key rotation.
- **Post-quantum TLS** — the CLI and agent now prefer hybrid post-quantum key
  exchange when connecting to the Vouch server.
- **IAM Identity Center context in AssumeRole** — the CLI forwards IdC identity
  context into role sessions for trusted identity propagation.
- **Access-token audience enforcement** — the server now enforces the token
  audience at every resource endpoint.

## [v2026.7.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.7.1) — July 3, 2026

- **AWS setup redesigned around organization anchors** — `vouch credential aws`
  and `vouch setup aws` were rebuilt around a single org-level anchor, so
  multi-account access is configured once and roles resolve consistently
  everywhere.
- **Interactive AWS setup wizard** — `vouch setup aws` now walks you through
  account and role configuration step by step instead of requiring flags.
- **Device code pre-fill** — `vouch login` pre-fills the device code in the
  browser, removing the copy-and-type step from the login flow.

## [v2026.6.3](https://github.com/vouch-sh/vouch/releases/tag/v2026.6.3) — June 24, 2026

- **Internationalization** — the server UI, CLI, and agent are now fully
  translation-ready, with all user-facing strings extracted into per-binary
  [Fluent](https://projectfluent.org/) catalogs and locale negotiated
  automatically. Timestamps in the applications and admin UI render in the
  viewer&#39;s locale.
- **XDG Base Directory compliance** — the CLI and agent now follow the XDG
  spec on every platform for config, state, data, and cache files. A legacy
  flat `~/.vouch/` directory is migrated automatically on first run.
- **Mandatory HTTP message signatures** — all `/v1/*` API requests now require
  [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) HTTP message signatures
  with Content-Digest enforcement, and the server advertises its policy via
  `Accept-Signature`.
- **OpenID Connect RP-Initiated Logout 1.0** — applications can now end a
  Vouch session through the standard RP-initiated logout flow.
- **STIG-aligned kernel hardening** — the server AMI ships with STIG-aligned
  kernel settings and a published STIG mapping document.

## [v2026.6.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.6.2) — June 13, 2026

- **Security evaluation remediation** — a prioritized internal security
  evaluation was completed and all P1 and P2 findings were remediated,
  including an SSRF egress guard for client-controlled JWKS and `request_uri`
  fetches.
- **Server-rendered security-keys page** — key management moved to full
  server-side rendering with form POSTs, reducing the JavaScript surface.
- **FIPS crypto-policy in the attestable AMI** — the hardened server image now
  enables the FIPS cryptographic policy.

## [v2026.5.4](https://github.com/vouch-sh/vouch/releases/tag/v2026.5.4) — May 30, 2026

- **Workload identity federation for AI APIs** — exchange a Vouch session for
  short-lived Anthropic (Claude) and OpenAI API credentials, so AI tooling
  runs without long-lived API keys on disk.

## [v2026.5.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.5.2) — May 17, 2026

- **Multiple upstream IdPs** — a single Vouch server can now federate with
  more than one upstream identity provider.
- **Multi-domain organizations** — organizations can span multiple email
  domains.
- **Cross-account KMS keys** — the server can use KMS signing keys that live
  in a different AWS account.

## [v2026.5.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.5.1) — May 11, 2026

- **DNS-over-HTTPS** — the CLI and agent resolve the Vouch server over
  encrypted DNS, hardening credential flows on untrusted networks.

## [v2026.4.8](https://github.com/vouch-sh/vouch/releases/tag/v2026.4.8) — May 1, 2026

- **Windows support matured** — Vouch is published to winget
  (`winget install SmokeTurner.Vouch`) and Windows binaries are code-signed.
- **Clock-skew detection** — the CLI warns when local clock drift would cause
  token validation failures.

## [v2026.4.6](https://github.com/vouch-sh/vouch/releases/tag/v2026.4.6) — April 27, 2026

- **SSH certificate caching** — issued SSH certificates are cached for their
  lifetime instead of being re-requested on every connection.
- **AI agent attribution** — when an AI agent invokes AWS credential helpers,
  the `AI_AGENT` environment variable is forwarded into session tags for
  CloudTrail attribution, with a session policy limiting role chaining.
- **IAM role paths** — roles with paths (e.g. `/engineering/deploy`) are fully
  supported.

## [v2026.4.5](https://github.com/vouch-sh/vouch/releases/tag/v2026.4.5) — April 21, 2026

- **AWS multi-account SSO** — automatic discovery of accounts and roles across
  an AWS organization, with role chaining.
- **`vouch aws console`** — open the AWS web console from the terminal using
  your short-lived credentials.
- **RFC 9728 Protected Resource Metadata** — the server publishes OAuth 2.0
  protected resource metadata for standards-based client discovery.
