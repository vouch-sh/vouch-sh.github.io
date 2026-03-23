---
title: "Next.js (NextAuth.js)"
description: "Integrate Vouch OIDC authentication into a Next.js application using NextAuth.js."
weight: 4
params:
  category: "server"
  language: "JavaScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install [NextAuth.js](https://next-auth.js.org/):

```bash
npm install next-auth
```

Create the auth configuration in `pages/api/auth/[...nextauth].js`:

```javascript
import NextAuth from 'next-auth';

export default NextAuth({
  providers: [{
    id: 'vouch',
    name: 'Vouch',
    type: 'oauth',
    wellKnown: `${process.env.VOUCH_ISSUER || 'https://{{< instance-url >}}'}/.well-known/openid-configuration`,
    clientId: process.env.VOUCH_CLIENT_ID,
    clientSecret: process.env.VOUCH_CLIENT_SECRET,
    authorization: { params: { scope: 'openid email' } },
    checks: ['pkce', 'state'],
    idToken: true,
    client: { id_token_signed_response_alg: 'ES256' },
    profile(profile) {
      return {
        id: profile.sub,
        email: profile.email,
        hardwareVerified: profile.hardware_verified,
        hardwareAaguid: profile.hardware_aaguid,
      };
    },
  }],
  callbacks: {
    async jwt({ token, profile }) {
      if (profile) {
        token.hardwareVerified = profile.hardware_verified;
        token.hardwareAaguid = profile.hardware_aaguid;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.hardwareVerified = token.hardwareVerified;
      session.user.hardwareAaguid = token.hardwareAaguid;
      return session;
    },
  },
  secret: process.env.NEXTAUTH_SECRET || 'dev-secret-change-in-production',
});
```

The provider uses `type: 'oauth'` with `wellKnown` for OIDC auto-discovery. The `client` option sets `id_token_signed_response_alg` to `ES256` to match Vouch's signing algorithm. Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are extracted from the profile and passed through to the session via JWT callbacks.

Wrap your app with the session provider in `pages/_app.js`:

```javascript
import { SessionProvider } from 'next-auth/react';

export default function App({ Component, pageProps: { session, ...pageProps } }) {
  return (
    <SessionProvider session={session}>
      <Component {...pageProps} />
    </SessionProvider>
  );
}
```

Use the session in any page:

```javascript
import { signIn, signOut, useSession } from 'next-auth/react';

export default function Home() {
  const { data: session } = useSession();

  if (session) {
    return (
      <div>
        <p>Signed in as {session.user.email}</p>
        {session.user.hardwareVerified && <p><strong>Hardware Verified</strong></p>}
        <button onClick={() => signOut()}>Sign out</button>
      </div>
    );
  }

  return <button onClick={() => signIn('vouch')}>Sign in with Vouch</button>;
}
```

Add environment variables to `.env.local`:

```
VOUCH_ISSUER=https://{{< instance-url >}}
VOUCH_CLIENT_ID=your-client-id
VOUCH_CLIENT_SECRET=your-client-secret
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-session-secret
```

**Callback URL:** Register `http://localhost:3000/api/auth/callback/vouch` as a redirect URI for your application.
