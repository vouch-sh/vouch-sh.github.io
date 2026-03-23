---
title: "Laravel (Socialite)"
description: "Integrate Vouch OIDC authentication into a Laravel application using Socialite."
weight: 5
params:
  category: "server"
  language: "PHP"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Install the Socialite OIDC driver and provider manager:

```bash
composer require laravel/socialite kovah/laravel-socialite-oidc socialiteproviders/manager
```

Register the service provider in `config/app.php`:

```php
'providers' => [
    // ...
    SocialiteProviders\Manager\ServiceProvider::class,
],
```

Register the event listener in `app/Providers/EventServiceProvider.php`:

```php
protected $listen = [
    \SocialiteProviders\Manager\SocialiteWasCalled::class => [
        \SocialiteProviders\OIDC\OIDCExtendSocialite::class . '@handle',
    ],
];
```

Add the provider configuration to `config/services.php`:

```php
'oidc' => [
    'base_url' => env('VOUCH_ISSUER', 'https://{{< instance-url >}}'),
    'client_id' => env('VOUCH_CLIENT_ID'),
    'client_secret' => env('VOUCH_CLIENT_SECRET'),
    'redirect' => env('VOUCH_REDIRECT_URI', 'http://localhost:3000/auth/callback'),
],
```

Add routes in `routes/web.php`:

```php
use App\Http\Controllers\AuthController;

Route::get('/', [AuthController::class, 'home']);
Route::get('/auth/redirect', [AuthController::class, 'redirect']);
Route::get('/auth/callback', [AuthController::class, 'callback']);
Route::post('/logout', [AuthController::class, 'logout']);
```

Create the controller in `app/Http/Controllers/AuthController.php`:

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Laravel\Socialite\Facades\Socialite;
use SocialiteProviders\Manager\Config;

class AuthController extends \Illuminate\Routing\Controller
{
    private function oidcConfig(): Config
    {
        return new Config(
            config('services.oidc.client_id'),
            config('services.oidc.client_secret'),
            config('services.oidc.redirect'),
            ['base_url' => config('services.oidc.base_url')]
        );
    }

    public function redirect()
    {
        return Socialite::driver('oidc')
            ->setConfig($this->oidcConfig())
            ->scopes(['openid', 'email'])
            ->enablePKCE()
            ->redirect();
    }

    public function callback(Request $request)
    {
        $vouchUser = Socialite::driver('oidc')
            ->setConfig($this->oidcConfig())
            ->enablePKCE()
            ->user();

        $request->session()->put('user', [
            'email' => $vouchUser->email,
            'hardware_verified' => $vouchUser->user['hardware_verified'] ?? false,
        ]);

        return redirect('/');
    }

    public function logout(Request $request)
    {
        $request->session()->forget('user');
        return redirect('/');
    }
}
```

The driver name is `'oidc'`. PKCE is enabled with `->enablePKCE()`. Hardware attestation claims are available via `$vouchUser->user['hardware_verified']`.

### Rich Authorization Requests

To request structured permissions beyond scopes, pass `authorization_details` as an extra parameter using the `with()` method:

```php
return Socialite::driver('oidc')
    ->setConfig($this->oidcConfig())
    ->scopes(['openid', 'email'])
    ->enablePKCE()
    ->with([
        'authorization_details' => json_encode([
            ['type' => 'account_access', 'actions' => ['read', 'transfer']]
        ])
    ])
    ->redirect();
```

See the [Rich Authorization Requests]({{< ref "/docs/applications#rich-authorization-requests" >}}) section for the full `authorization_details` format.
