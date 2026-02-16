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

Create the auth configuration in `app/api/auth/[...nextauth]/route.js`:

```javascript
import NextAuth from "next-auth";

const handler = NextAuth({
  providers: [
    {
      id: "vouch",
      name: "Vouch",
      type: "oidc",
      issuer: "https://{{< instance-url >}}",
      clientId: process.env.VOUCH_CLIENT_ID,
      clientSecret: process.env.VOUCH_CLIENT_SECRET,
      authorization: { params: { scope: "openid email" } },
      profile(profile) {
        return {
          id: profile.sub,
          name: profile.name,
          email: profile.email,
        };
      },
    },
  ],
  callbacks: {
    async jwt({ token, account, profile }) {
      if (account) {
        token.accessToken = account.access_token;
        token.vouchId = profile.sub;
      }
      return token;
    },
    async session({ session, token }) {
      session.accessToken = token.accessToken;
      session.user.vouchId = token.vouchId;
      return session;
    },
  },
});

export { handler as GET, handler as POST };
```

Add environment variables to `.env.local`:

```
NEXTAUTH_URL=https://your-app.example.com
NEXTAUTH_SECRET=your-session-secret
VOUCH_CLIENT_ID=your-client-id
VOUCH_CLIENT_SECRET=your-client-secret
```
