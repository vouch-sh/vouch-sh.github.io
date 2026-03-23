---
title: "ASP.NET Core (OpenID Connect)"
description: "Integrate Vouch OIDC authentication into an ASP.NET Core application using the built-in OpenID Connect middleware."
weight: 11
params:
  category: "server"
  language: "C#"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

ASP.NET Core includes built-in OpenID Connect middleware via the [Microsoft.AspNetCore.Authentication.OpenIdConnect](https://www.nuget.org/packages/Microsoft.AspNetCore.Authentication.OpenIdConnect) package. Requires .NET 10.0 or later.

Add the package:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```

### Authentication configuration

Configure cookie and OpenID Connect authentication in `Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;

var builder = WebApplication.CreateBuilder(args);

var vouchIssuer = Environment.GetEnvironmentVariable("VOUCH_ISSUER") ?? "https://{{< instance-url >}}";
var clientId = Environment.GetEnvironmentVariable("VOUCH_CLIENT_ID")
    ?? throw new InvalidOperationException("VOUCH_CLIENT_ID is required");
var clientSecret = Environment.GetEnvironmentVariable("VOUCH_CLIENT_SECRET")
    ?? throw new InvalidOperationException("VOUCH_CLIENT_SECRET is required");
var redirectUri = Environment.GetEnvironmentVariable("VOUCH_REDIRECT_URI") ?? "http://localhost:3000/callback";

builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = vouchIssuer;
    options.ClientId = clientId;
    options.ClientSecret = clientSecret;
    options.ResponseType = OpenIdConnectResponseType.Code;
    options.UsePkce = true;
    options.Scope.Clear();
    options.Scope.Add("openid");
    options.Scope.Add("email");
    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;
    options.CallbackPath = new PathString(new Uri(redirectUri).AbsolutePath);
    options.Events = new OpenIdConnectEvents
    {
        OnRedirectToIdentityProvider = context =>
        {
            context.ProtocolMessage.RedirectUri = redirectUri;
            return Task.CompletedTask;
        },
    };

    options.ClaimActions.MapJsonKey("hardware_verified", "hardware_verified");
    options.ClaimActions.MapJsonKey("hardware_aaguid", "hardware_aaguid");
});

builder.Services.AddAuthorization();
```

The `ClaimActions.MapJsonKey` calls map the Vouch-specific `hardware_verified` and `hardware_aaguid` claims from the UserInfo response into the user's `ClaimsPrincipal`.

### Login and logout endpoints

```csharp
var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/login", () =>
    Results.Challenge(
        new AuthenticationProperties { RedirectUri = "/" },
        [OpenIdConnectDefaults.AuthenticationScheme]));

app.MapPost("/logout", async (HttpContext context) =>
{
    await context.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
    return Results.Redirect("/");
});
```

### Accessing claims

After authentication, read claims from the `ClaimsPrincipal`:

```csharp
app.MapGet("/", (HttpContext context) =>
{
    if (context.User.Identity?.IsAuthenticated == true)
    {
        var email = context.User.FindFirst("email")?.Value ?? "unknown";
        var hwVerified = context.User.FindFirst("hardware_verified")?.Value == "true";
        // Use email and hwVerified as needed
    }
});
```

### Rich Authorization Requests

To request structured permissions beyond scopes, add `authorization_details` as an extra parameter in the redirect event:

```csharp
options.Events = new OpenIdConnectEvents
{
    OnRedirectToIdentityProvider = context =>
    {
        context.ProtocolMessage.SetParameter(
            "authorization_details",
            """[{"type":"account_access","actions":["read","transfer"]}]""");
        return Task.CompletedTask;
    },
};
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
