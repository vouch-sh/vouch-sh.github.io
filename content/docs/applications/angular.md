---
title: "Angular (angular-auth-oidc-client)"
description: "Integrate Vouch OIDC authentication into an Angular single-page application using angular-auth-oidc-client."
weight: 23
params:
  category: "spa"
  language: "TypeScript"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

For client-side applications running in the browser, use the authorization code flow with PKCE. These examples do not use a client secret since the code runs in the user's browser.

> **PKCE is required** for browser-based clients. `angular-auth-oidc-client` enables PKCE by default -- no additional configuration is needed.

[angular-auth-oidc-client](https://github.com/damienbod/angular-auth-oidc-client) is a certified OpenID Connect client library for Angular.

Install the library:

```bash
npm install angular-auth-oidc-client
```

Configure the OIDC provider in `src/main.ts`:

```typescript
import { bootstrapApplication } from "@angular/platform-browser";
import { provideRouter } from "@angular/router";
import { provideAuth, LogLevel } from "angular-auth-oidc-client";
import { AppComponent } from "./app/app.component";
import { routes } from "./app/app.routes";

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideAuth({
      config: {
        authority: "https://{{< instance-url >}}",
        clientId: "your-client-id",
        redirectUrl: "https://your-app.example.com/callback",
        postLogoutRedirectUri: window.location.origin,
        scope: "openid email",
        responseType: "code",
        autoUserInfo: true,
        logLevel: LogLevel.Warn,
      },
    }),
  ],
});
```

Use the auth service in `src/app/app.component.ts`:

```typescript
import { Component, OnInit, inject } from "@angular/core";
import { CommonModule } from "@angular/common";
import { RouterOutlet } from "@angular/router";
import { OidcSecurityService } from "angular-auth-oidc-client";

@Component({
  selector: "app-root",
  standalone: true,
  imports: [CommonModule, RouterOutlet],
  template: `
    <div *ngIf="isAuthenticated; else loginBlock">
      <p>Signed in as {{ email }}</p>
      <p *ngIf="hardwareVerified"><strong>Hardware Verified</strong></p>
      <button (click)="logout()">Sign out</button>
    </div>

    <ng-template #loginBlock>
      <button (click)="login()">Sign in with Vouch</button>
    </ng-template>

    <router-outlet></router-outlet>
  `,
})
export class AppComponent implements OnInit {
  private oidc = inject(OidcSecurityService);

  isAuthenticated = false;
  email = "";
  hardwareVerified = false;

  ngOnInit() {
    this.oidc.checkAuth().subscribe(({ isAuthenticated, userData }) => {
      this.isAuthenticated = isAuthenticated;
      if (userData) {
        this.email = userData.email || "";
        this.hardwareVerified = userData.hardware_verified || false;
      }
      if (window.location.pathname === "/callback") {
        window.location.href = "/";
      }
    });
  }

  login() {
    this.oidc.authorize();
  }

  logout() {
    this.oidc.logoffLocal();
    this.isAuthenticated = false;
    this.email = "";
    this.hardwareVerified = false;
  }
}
```

Create a callback component (`src/app/callback.component.ts`):

```typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-callback",
  standalone: true,
  template: "<p>Processing login...</p>",
})
export class CallbackComponent {}
```

Register the callback route in your routes configuration:

```typescript
import { CallbackComponent } from "./callback.component";

export const routes = [
  { path: "callback", component: CallbackComponent },
];
```

### Rich Authorization Requests

To request structured permissions beyond scopes, add `customParamsAuthRequest` to the OIDC configuration:

```typescript
provideAuth({
  config: {
    // ... other config
    customParamsAuthRequest: {
      authorization_details: JSON.stringify([
        { type: "account_access", actions: ["read", "transfer"] },
      ]),
    },
  },
});
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
