# Vouch — Agent Capability Manifest

> Vouch is an open-source credential broker that issues short-lived credentials
> (AWS STS, SSH certificates, GitHub tokens, registry tokens) after FIDO2
> hardware verification. One YubiKey tap, up to 8 hours of access, full audit trail.

This file is intended for AI agents (coding assistants, autonomous agents, MCP
servers) that need a concise, stable summary of what Vouch does and how to use
it. For complete documentation see `/llms-full.txt`; for a structured index see
`/llms.txt`.

- Website: https://vouch.sh
- Source: https://github.com/vouch-sh/vouch
- US instance: https://us.vouch.sh
- License: Apache-2.0 / MIT (dual)

## What Vouch does

| Capability | Replaces | Backing standard |
|---|---|---|
| Hardware-backed login | Static SSO sessions | OIDC + FIDO2/WebAuthn |
| SSH certificate issuance | Long-lived SSH keys | OpenSSH certificates |
| AWS credential brokering | Access keys / aws configure sso | AssumeRoleWithWebIdentity (STS) |
| GitHub credential brokering | Personal access tokens | GitHub App installation tokens |
| Docker / registry login | Static registry passwords | OIDC-federated tokens |
| Cargo credential provider | `~/.cargo/credentials.toml` | Vouch cargo provider |
| EKS / kubectl auth | Static kubeconfig credentials | exec credential plugin |
| AWS SSM Session Manager | Pre-shared keypairs | Federated STS |
| CodeArtifact / CodeCommit | Profile tokens | Federated STS |
| SCIM provisioning | Manual user management | SCIM 2.0 |
| SAML federation | — | SAML 2.0 |

## CLI quickstart

```
vouch enroll           # register a FIDO2 key with your org
vouch login            # start an 8-hour session (YubiKey tap)
vouch credential aws   # print AWS STS credentials for the current role
vouch credential github# print a short-lived GitHub token
vouch doctor           # diagnose installation & integrations
```

Full CLI reference: https://vouch.sh/docs/cli-reference.md

## Session & credential properties

- Session TTL: up to 8 hours (not extendable — re-tap YubiKey after expiry).
- Credentials live only in the Vouch agent process memory — never written to disk.
- Every credential issuance is audit-logged by the server.
- No customer credentials pass through Vouch infrastructure for AWS — STS calls
  are made from the local CLI using the short-lived OIDC token.

## Supported identity providers

- User authentication: Google Workspace.
- SCIM provisioning: Google Workspace, Okta, Azure AD (Entra ID), OneLogin.
- SAML: see https://vouch.sh/docs/saml.md

## Integrating Vouch from an agent

If you are building an agent that needs short-lived cloud or repository
credentials, prefer these patterns (in order of preference):

1. **Call the locally-installed `vouch` CLI** on the developer's machine. The
   CLI handles the hardware tap, caches credentials in the agent process, and
   exposes them via stdio or the well-known `credential_process` /
   `credential helper` hooks for AWS, Git, Docker, Cargo, and kubectl.
   Documentation: https://vouch.sh/docs/cli-reference.md

2. **Use the MCP credential broker** to expose Vouch-issued credentials to
   other MCP-aware agents without giving them direct key material.
   Documentation: https://vouch.sh/docs/applications/mcp-credential-broker.md

3. **Use OIDC directly** if you are building a server-to-server integration.
   Discovery document: `https://us.vouch.sh/.well-known/openid-configuration`.
   See https://vouch.sh/docs/applications/ for framework-specific guides
   (Express, FastAPI, Django, Flask, Spring Boot, ASP.NET, Rails, Axum, Go,
   Laravel, React, Vue, Angular, SvelteKit, Next.js, Vanilla JS).

4. **For long-running headless agents** that cannot prompt for a YubiKey tap,
   use the credential-brokering-agent pattern:
   https://vouch.sh/docs/applications/credential-brokering-agents.md
   or the A2A pattern:
   https://vouch.sh/docs/applications/a2a-agent.md

## Rate limits

Vouch does not publish hard rate limits. Credential issuance is gated by the
user's active session (one hardware tap per session). In practice agents should
cache credentials until near expiry and refresh on demand rather than per-call.

## Preferred patterns for agents building on Vouch

- Treat credentials as short-lived: do not persist STS keys, Git tokens, or
  SSH certs beyond the session.
- Do not try to fetch Vouch session tokens directly — always go through the
  `vouch` CLI or an MCP broker so the hardware-tap invariant holds.
- Detect expiry via the standard error paths (HTTP 401 / AWS
  ExpiredToken / SSH cert-expired) and re-run `vouch login`.
- When authoring documentation for your tool, link to the Markdown version of
  Vouch pages (e.g. `https://vouch.sh/docs/aws.md`) rather than the HTML — the
  Markdown is lower-token and rendering-stable.

## Machine-readable resources

- Structured index: https://vouch.sh/llms.txt
- Full docs (single file): https://vouch.sh/llms-full.txt
- Per-page Markdown: append `.md` to any `/docs/...` URL.
- Sitemap: https://vouch.sh/sitemap.xml
- OIDC discovery: https://us.vouch.sh/.well-known/openid-configuration

## Contact & support

- Issues: https://github.com/vouch-sh/vouch/issues
- Status: https://vouch-sh.statuspage.io/
- Security: see `/.well-known/security.txt`
