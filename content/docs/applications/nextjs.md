---
title: "Next.js (NextAuth.js)"
description: "Integrate Vouch OIDC authentication into a Next.js application using NextAuth.js."
weight: 4
params:
  category: "server"
  language: "JavaScript"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

NextAuth.js provides drop-in authentication for Next.js with OIDC auto-discovery. Configure the provider with `id_token_signed_response_alg: 'ES256'` to match Vouch's signing algorithm.

## Example

**[web/nextjs-nextauth](https://github.com/vouch-sh/examples/tree/main/web/nextjs-nextauth)** -- Complete working example with NextAuth.js provider, PKCE, and hardware attestation claims passed through JWT and session callbacks.
