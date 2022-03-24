# Two Factor
[![Latest Version on Packagist](https://img.shields.io/packagist/v/laragear/two-factor.svg)](https://packagist.org/packages/laragear/two-factor) [![Latest stable test run](https://github.com/Laragear/TwoFactor/workflows/Tests/badge.svg)](https://github.com/Laragear/TwoFactor/actions) [![Codecov coverage](https://codecov.io/gh/Laragear/TwoFactor/branch/1.x/graph/badge.svg?token=BJMBVZNPM8)](https://codecov.io/gh/Laragear/TwoFactor) [![Maintainability](https://api.codeclimate.com/v1/badges/64241e25adb0f55d7ba1/maintainability)](https://codeclimate.com/github/Laragear/TwoFactor/maintainability) [![Sonarcloud Status](https://sonarcloud.io/api/project_badges/measure?project=Laragear_TwoFactor&metric=alert_status)](https://sonarcloud.io/dashboard?id=Laragear_TwoFactor) [![Laravel Octane Compatibility](https://img.shields.io/badge/Laravel%20Octane-Compatible-success?style=flat&logo=laravel)](https://laravel.com/docs/9.x/octane#introduction)

On-premises Two-Factor Authentication for all your users out of the box.

```php
use Illuminate\Support\Facades\Auth;
use Laragear\TwoFactor\TwoFactor;

Auth::attemptWhen($request->only('email', 'password'), TwoFactor::hasCode());
```

This package enables TOTP authentication using 6 digits codes. No need for external APIs.

## Keep this package free

[![](.assets/patreon.png)](https://patreon.com/packagesforlaravel)[![](.assets/ko-fi.png)](https://ko-fi.com/DarkGhostHunter)[![](.assets/buymeacoffee.png)](https://www.buymeacoffee.com/darkghosthunter)[![](.assets/paypal.png)](https://www.paypal.com/paypalme/darkghosthunter)

Your support allows me to keep this package free, up-to-date and maintainable. Alternatively, you can **[spread the word!](http://twitter.com/share?text=I%20am%20using%20this%20cool%20PHP%20package&url=https://github.com%2FLaragear%2FTwoFactor&hashtags=PHP,Laravel)**

## Requirements

* Laravel 9.x or later
* PHP 8.0 or later

## Installation

Fire up Composer and require this package in your project.

    composer require laragear/two-factor

That's it.

### How this works

This package adds a **Contract** to detect if, after the credentials are deemed valid, should use Two-Factor Authentication as a second layer of authentication.

It includes a custom **view** and a **callback** to handle the Two-Factor authentication itself during login attempts.

Works without middleware or new guards, but you can go full manual if you want.

## Set up

1. First, publish the migration file into your application, and use `migrate` to create the table that handles the Two-Factor Authentication information for each model you want to attach to 2FA.

```shell
php artisan vendor:publish --provider="Laragear\TwoFactor\TwoFactorServiceProvider" --tag="migrations"
php artisan migrate
```

2. Add the `TwoFactorAuthenticatable` _contract_ and the `TwoFactorAuthentication` trait to the User model, or any other model you want to make Two-Factor Authentication available. 

```php
<?php

namespace App;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laragear\TwoFactor\TwoFactorAuthentication;
use Laragear\TwoFactor\Contracts\TwoFactorAuthenticatable;

class User extends Authenticatable implements TwoFactorAuthenticatable
{
    use TwoFactorAuthentication;
    
    // ...
}
```

> The contract is used to identify the model using Two-Factor Authentication, while the trait conveniently implements the methods required to handle it.

That's it. You're now ready to use 2FA in your application.

### Enabling Two-Factor Authentication

To enable Two-Factor Authentication for the User, he must sync the Shared Secret between its Authenticator app and the application. 

> Some free Authenticator Apps are [iOS Authenticator](https://www.apple.com/ios/ios-15-preview/features), [FreeOTP](https://freeotp.github.io/), [Authy](https://authy.com/), [andOTP](https://github.com/andOTP/andOTP), [Google](https://apps.apple.com/app/google-authenticator/id388497605) [Authenticator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=en), and [Microsoft Authenticator](https://www.microsoft.com/en-us/account/authenticator), to name a few.

To start, generate the needed data using the `createTwoFactorAuth()` method. Once you do, you can show the Shared Secret to the User as a string or QR Code (encoded as SVG) in your view.

```php
use Illuminate\Http\Request;

public function prepareTwoFactor(Request $request)
{
    $secret = $request->user()->createTwoFactorAuth();
    
    return view('user.2fa', [
        'qr_code' => $secret->toQr(),     // As QR Code
        'uri'     => $secret->toUri(),    // As "otpauth://" URI.
        'string'  => $secret->toString(), // As a string
    ]);
}
```

> When you use `createTwoFactorAuth()` on someone with Two-Factor Authentication already enabled, the previous data becomes permanently invalid. This ensures a User **never** has two Shared Secrets enabled at any given time.

Then, the User must confirm the Shared Secret with a Code generated by their Authenticator app. The `confirmTwoFactorAuth()` method will automatically enable it if the code is valid.

```php
use Illuminate\Http\Request;

public function confirmTwoFactor(Request $request)
{
    $request->validate([
        'code' => 'required|numeric'
    ]);
    
    $activated = $request->user()->confirmTwoFactorAuth($request->code);
    
    // ...
}
```

If the User doesn't issue the correct Code, the method will return `false`. You can tell the User to double-check its device's timezone, or create another Shared Secret with `createTwoFactorAuth()`.

### Recovery Codes

Recovery Codes are automatically generated each time the Two-Factor Authentication is enabled. By default, a Collection of ten one-use 8-characters codes are created.

You can show them using `getRecoveryCodes()`.

```php
use Illuminate\Http\Request;

public function confirmTwoFactor(Request $request)
{
    if ($request->user()->confirmTwoFactorAuth($request->code)) {
        return $request->user()->getRecoveryCodes();
    }
    
    return 'Try again!';
}
```

You're free on how to show these codes to the User, but **ensure** you show them at least one time after a successfully enabling Two-Factor Authentication, and ask him to print them somewhere.

> These Recovery Codes are handled automatically when the User sends it instead of a TOTP code. If it's a recovery code, the package will use and mark it as invalid, so it can't be used again.

The User can generate a fresh batch of codes using `generateRecoveryCodes()`, which replaces the previous batch.

```php
use Illuminate\Http\Request;

public function showRecoveryCodes(Request $request)
{
    return $request->user()->generateRecoveryCodes();
}
```

> If the User depletes his recovery codes without disabling Two-Factor Authentication, or Recovery Codes are deactivated, **he may be locked out forever without his Authenticator app**. Ensure you have countermeasures in these cases.

### Logging in

To login, the user must issue a TOTP code along their credentials. Simply use `Auth::attemptWhen()` with TwoFactor, which will automatically do the checks for you. By default, it checks for the `2fa_code` input name, but you can issue your own.

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Laragear\TwoFactor\TwoFactor;

public function login(Request $request)
{
    // ...
    
    $credentials = $request->only('email', 'password');
    
    if (Auth::attemptWhen($credentials, TwoFactor::hasCode(), $request->filled('remember'))) {
        return redirect()->home(); 
    }
    
    return back()->withErrors(['email' => 'Bad credentials'])
}
```

Behind the scenes, once the User is retrieved and validated from your guard of choice, it makes an additional check for a valid TOTP code. If it's invalid, it will return false and no authentication will happen.

> For [Laravel UI](https://github.com/laravel/ui), override the `attemptLogin()` method in your Login controller.
> For [Laravel Breeze](https://laravel.com/docs/starter-kits#laravel-breeze), you may need to edit the `LoginRequest::authenticate()` call.
> For [Laravel Fortify](https://laravel.com/docs/fortify) and [Jetstream](https://jetstream.laravel.com/), you may need to set a custom callback with the `Fortify::authenticateUsing()` method.

#### Separating the TOTP requirement

In some occasions you will want to tell the user the authentication failed not because the credentials were incorrect, but because of the TOTP code was invalid.

You can use the `hasCodeOrFails()` method that does the same, but throws a validation exception, which is handled gracefully by the framework. It even accepts a custom message in case of failure, otherwise a default [translation](#translations) line will be used.

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Laragear\TwoFactor\TwoFactor;

public function login(Request $request)
{
    // ...
    
    $credentials = $request->only('email', 'password');
    
    if (Auth::attemptWhen($credentials, TwoFactor::hasCodeOrFails(), $request->filled('remember'))) {
        return redirect()->home(); 
    }
    
    return back()->withErrors(['email', 'Authentication failed!']);
}
```

Since `InvalidCodeException` extends the `ValidationException`, you can catch it and do more complex things, like those fancy views that hold the login procedure until the correct TOTP code is issued. 

### Deactivation

You can deactivate Two-Factor Authentication for a given User using the `disableTwoFactorAuth()` method. This will automatically invalidate the authentication data, allowing the User to log in with just his credentials.

```php
public function disableTwoFactorAuth(Request $request)
{
    $request->user()->disableTwoFactorAuth();
    
    return 'Two-Factor Authentication has been disabled!';
}
```

## Events

The following events are fired in addition to the default Authentication events.

* `TwoFactorEnabled`: An User has enabled Two-Factor Authentication.
* `TwoFactorRecoveryCodesDepleted`: An User has used his last Recovery Code.
* `TwoFactorRecoveryCodesGenerated`: An User has generated a new set of Recovery Codes.
* `TwoFactorDisabled`: An User has disabled Two-Factor Authentication.

> You can use `TwoFactorRecoveryCodesDepleted` to tell the User to create more Recovery Codes or mail them some more.

## Middleware

TwoFactor comes with two middleware for your routes: `2fa.enabled` and `2fa.confirm`.

> To avoid unexpected results, middleware only act on your users models implementing the `TwoFactorAuthenticatable` contract. If a user model doesn't implement it, the middleware will bypass any 2FA logic.

### Require 2FA

If you need to ensure the User has Two-Factor Authentication enabled before entering a given route, you can use the `2fa.enabled` middleware. Users who implement the `TwoFactorAuthenticatable` contract and have 2FA disabled will be redirected to a route name containing the warning, which is `2fa.notice` by default.

```php
Route::get('system/settings', function () {
    // ...
})->middleware('2fa.enabled');
```

You can implement the view easily with the one included in this package, optionally with a URL to point the user to enable 2FA:

```php
use Illuminate\Support\Facades\Route;

Route::view('2fa-required', 'two-factor::notice', [
    'url' => url('settings/2fa')
])->name('2fa.notice');
```

### Confirm 2FA

Much like the [`password.confirm` middleware](https://laravel.com/docs/authentication#password-confirmation), you can also ask the user to confirm entering a route by issuing a 2FA Code with the `2fa.confirm` middleware.

```php
Route::get('api/token', function () {
    // ...
})->middleware('2fa.confirm');

Route::post('api/token/delete', function () {
    // ...
})->middleware('2fa.confirm');
```

The middleware will redirect the user to the named route `2fa.confirm` by default, but you can change it in the first parameter. To implement the receiving routes, TwoFactor comes with the `Confirm2FACodeController` anda view you can use for a quick start.

```php
use Illuminate\Support\Facades\Route;
use Laragear\TwoFactor\Http\Controllers\ConfirmTwoFactorCodeController;

Route::get('2fa-confirm', [ConfirmTwoFactorCodeController::class, 'form'])
    ->name('2fa.confirm');

Route::post('2fa-confirm', [ConfirmTwoFactorCodeController::class, 'confirm']);
```

Since a user without 2FA enabled won't be asked for a code, you can combine the middleware with `2fa.require` to ensure confirming is mandatory for users without 2FA enabled.

```php
use Illuminate\Support\Facades\Route;

Route::get('api/token', function () {
    // ...
})->middleware('2fa.require', '2fa.confirm');
```

## Validation

Sometimes you may want to manually trigger a TOTP validation in any part of your application for the authenticated user. You can validate a TOTP code for the authenticated user using the `topt` rule.

```php
public function checkTotp(Request $request)
{
    $request->validate([
        'code' => 'totp'
    ]);

    // ...
}
```

This rule will succeed only if  the user is authenticated, it has Two-Factor Authentication enabled, and the code is correct or is a recovery code.

> You can enforce the rule to NOT use recovery codes using `totp:code`.

## Translations

TwoFactor comes with translation files that you can use immediately in your application. These are also used for the [validation rule](#validation).

```php
public function disableTwoFactorAuth()
{
    // ...

    session()->flash('message', trans('two-factor::messages.success'));

    return back();
}
```

To add your own language, publish the translation files. These will be located in `lang/vendor/two-factor`:

```shell
php artisan vendor:publish --provider="Laragear\TwoFactor\TwoFactorServiceProvider" --tag="translations"
```

## Configuration

To further configure the package, publish the configuration file:

```shell
php artisan vendor:publish --provider="Laragear\TwoFactor\TwoFactorServiceProvider" --tag="config"
```

You will receive the `config/two-factor.php` config file with the following contents:

```php
return [
    'cache' => [
        'store' => null,
        'prefix' => '2fa.code'
    ],
    'recovery' => [
        'enabled' => true,
        'codes' => 10,
        'length' => 8,
	],
    'safe_devices' => [
        'enabled' => false,
        'max_devices' => 3,
        'expiration_days' => 14,
	],
    'confirm' => [
        'key' => '_2fa',
        'time' => 60 * 3,
    ],
    'secret_length' => 20,
    'issuer' => env('OTP_TOTP_ISSUER'),
    'totp' => [
        'digits' => 6,
        'seconds' => 30,
        'window' => 1,
        'algorithm' => 'sha1',
    ],
    'qr_code' => [
        'size' => 400,
        'margin' => 4
    ],
];
```

### Cache Store

```php
return  [
    'cache' => [
        'store' => null,
        'prefix' => '2fa.code'
    ],
];
```

[RFC 6238](https://tools.ietf.org/html/rfc6238#section-5) states that one-time passwords shouldn't be able to be usable more than once, even if is still inside the time window. For this, we need to use the Cache to ensure the same code cannot be used again.

You can change the store to use, which it's the default used by your application, and the prefix to use as cache keys, in case of collisions.

### Recovery

```php
return [
    'recovery' => [
        'enabled' => true,
        'codes' => 10,
        'length' => 8,
    ],
];
```

Recovery codes handling are enabled by default, but you can disable it. If you do, ensure Users can authenticate by other means, like sending an email with a link to a signed URL that logs him in and disables Two-Factor Authentication, or SMS.

The number and length of codes generated is configurable. 10 Codes of 8 random characters are enough for most authentication scenarios.

### Safe devices

```php
return [
    'safe_devices' => [
        'enabled' => false,
        'max_devices' => 3,
        'expiration_days' => 14,
    ],
];
```

Enabling this option will allow the application to "remember" a device using a cookie, allowing it to bypass Two-Factor Authentication once a code is verified in that device. When the User logs in again in that device, it won't be prompted for a 2FA Code again.

The cookie contains a random value which is checked against a list of safe devices saved for the authenticating user. It's considered a safe device if the value matches and has not expired.

There is a limit of devices that can be saved, but usually three is enough (phone, tablet and PC). New devices will displace the oldest devices registered. Devices are considered no longer "safe" until a set amount of days.

You can change the maximum number of devices saved and the amount of days of validity once they're registered. More devices and more expiration days will make the Two-Factor Authentication less secure.

> When disabling Two-Factor Authentication, the list of devices is flushed.

### Confirmation Middleware

```php
return [
    'confirm' => [
        'key' => '_2fa',
        'time' => 60 * 3,
    ],
];
```

These control which key to use in the session for handling [`2fa.confirm` middleware](#confirm-2fa), and the expiration time in minutes.

### Secret length

```php
return [
    'secret_length' => 20,
];
```

This controls the length (in bytes) used to create the Shared Secret. While a 160-bit shared secret is enough, you can tighten or loosen the secret length to your liking.

It's recommended to use 128-bit or 160-bit because some Authenticator apps may have problems with non-RFC-recommended lengths.

### TOTP Configuration 

```php
return [
    'issuer' => env('OTP_TOTP_ISSUER'),
    'totp' => [
        'digits' => 6,
        'seconds' => 30,
        'window' => 1,
        'algorithm' => 'sha1',
    ],
];
```

This controls TOTP code generation and verification mechanisms:

* Issuer: The name of the issuer of the TOTP. Default is the application name. 
* TOTP Digits: The amount of digits to ask for TOTP code. 
* TOTP Seconds: The number of seconds a code is considered valid.
* TOTP Window: Additional steps of seconds to keep a code as valid.
* TOTP Algorithm: The system-supported algorithm to handle code generation.

This configuration values are always passed down to the authentication app as URI parameters:

    otpauth://totp/Laravel:taylor@laravel.com?secret=THISISMYSECRETPLEASEDONOTSHAREIT&issuer=Laravel&label=taylor%40laravel.com&algorithm=SHA1&digits=6&period=30

These values are printed to each 2FA data record inside the application. Changes will only take effect for new activations.

> Do not edit these parameters if you plan to use publicly available Authenticator apps, since some of them **may not support non-standard configuration**, like more digits, different period of seconds or other algorithms.

### QR Code Configuration 

```php
return [
    'qr_code' => [
        'size' => 400,
        'margin' => 4
    ],
];
```

This controls the size and margin used to create the QR Code, which are created as SVG.

## Laravel Octane Compatibility

- There are no singletons using a stale application instance. 
- There are no singletons using a stale config instance. 
- There are no singletons using a stale request instance. 
- There are no static properties written during a request.

There should be no problems using this package with Laravel Octane.

## Security

If you discover any security related issues, please email darkghosthunter@gmail.com instead of using the issue tracker.

# License

This specific package version is licensed under the terms of the [MIT License](LICENSE.md), at time of publishing.

[Laravel](https://laravel.com) is a Trademark of [Taylor Otwell](https://github.com/TaylorOtwell/). Copyright © 2011-2022 Laravel LLC.
