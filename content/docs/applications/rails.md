---
title: "Rails (OmniAuth)"
description: "Integrate Vouch OIDC authentication into a Ruby on Rails application using OmniAuth."
weight: 1
params:
  category: "server"
  language: "Ruby"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

[OmniAuth OpenID Connect](https://github.com/omniauth/omniauth_openid_connect) provides a standard OIDC strategy for Rails applications.

Add the `omniauth_openid_connect` gem to your `Gemfile`:

```ruby
gem 'omniauth_openid_connect'
```

Configure the provider in `config/initializers/omniauth.rb`:

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :openid_connect, {
    name: :vouch,
    scope: [:openid, :email],
    response_type: :code,
    pkce: true,
    issuer: "https://{{< instance-url >}}",
    discovery: true,
    client_options: {
      identifier: ENV["VOUCH_CLIENT_ID"],
      secret: ENV["VOUCH_CLIENT_SECRET"],
      redirect_uri: "https://your-app.example.com/auth/vouch/callback"
    }
  }
end
```

Add the callback route in `config/routes.rb`:

```ruby
get "/auth/vouch/callback", to: "sessions#create"
post "/auth/vouch/callback", to: "sessions#create"
```

Handle the callback in `app/controllers/sessions_controller.rb`:

```ruby
class SessionsController < ApplicationController
  def create
    auth = request.env["omniauth.auth"]
    user = User.find_or_create_by(vouch_id: auth.uid) do |u|
      u.email = auth.info.email
      u.name = auth.info.name
    end
    session[:user_id] = user.id
    redirect_to root_path, notice: "Signed in successfully."
  end
end
```

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` as an extra authorization parameter in the OmniAuth configuration:

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :openid_connect, {
    name: :vouch,
    scope: [:openid, :email],
    extra_authorize_params: {
      authorization_details: [
        { type: "account_access", actions: ["read", "transfer"] }
      ].to_json
    },
    # ... other configuration
  }
end
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
