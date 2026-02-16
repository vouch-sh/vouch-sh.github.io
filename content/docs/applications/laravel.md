---
title: "Laravel (Socialite)"
description: "Integrate Vouch OIDC authentication into a Laravel application using Socialite."
weight: 5
params:
  category: "server"
  language: "PHP"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install the [Socialite](https://laravel.com/docs/socialite) OpenID Connect driver:

```bash
composer require socialiteproviders/openid-connect
```

Add the provider configuration to `config/services.php`:

```php
'vouch' => [
    'client_id' => env('VOUCH_CLIENT_ID'),
    'client_secret' => env('VOUCH_CLIENT_SECRET'),
    'redirect' => env('VOUCH_REDIRECT_URI', 'https://your-app.example.com/auth/vouch/callback'),
    'discovery_url' => '{{< instance-url >}}/.well-known/openid-configuration',
],
```

Register the event listener in `app/Providers/EventServiceProvider.php`:

```php
protected $listen = [
    \SocialiteProviders\Manager\SocialiteWasCalled::class => [
        \SocialiteProviders\OpenIDConnect\OpenIDConnectExtendSocialite::class . '@handle',
    ],
];
```

Add routes in `routes/web.php`:

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/vouch', function () {
    return Socialite::driver('openid-connect')
        ->setConfig(config('services.vouch'))
        ->scopes(['openid', 'email'])
        ->redirect();
});

Route::get('/auth/vouch/callback', function () {
    $user = Socialite::driver('openid-connect')
        ->setConfig(config('services.vouch'))
        ->user();

    $localUser = User::updateOrCreate(
        ['vouch_id' => $user->getId()],
        [
            'name' => $user->getName(),
            'email' => $user->getEmail(),
        ]
    );

    Auth::login($localUser);
    return redirect('/dashboard');
});
```
