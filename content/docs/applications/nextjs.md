---
title: "Next.js (NextAuth.js)"
description: "Integrate Vouch OIDC authentication into a Next.js application using NextAuth.js."
weight: 4
params:
  category: "server"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[NextAuth.js](https://next-auth.js.org/) provides drop-in authentication for Next.js with OIDC auto-discovery. Key configuration:

- Configure a custom OAuth provider with `wellKnown` discovery URL
- Set `id_token_signed_response_alg: 'ES256'` to match Vouch's signing algorithm
- Enable PKCE with `checks: ['pkce', 'state']` and set `idToken: true`
- The hardware attestation claim (`hardware_verified`) is in the access token JWT — decode in the `jwt` callback and propagate through the session callback
- Requires `NEXTAUTH_SECRET` environment variable (generate with `openssl rand -base64 32`)

## Example

**[web/nextjs-nextauth](https://github.com/vouch-sh/examples/tree/main/web/nextjs-nextauth)** — Complete working example with NextAuth.js provider, PKCE, and hardware claim propagation through JWT and session callbacks.
