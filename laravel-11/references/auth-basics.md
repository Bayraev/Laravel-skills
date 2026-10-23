# Laravel 11 Authentication (Session) and Sanctum (API)

Sources: https://laravel.com/framework/docs/11.x/authentication and
https://laravel.com/framework/docs/11.x/sanctum.

## Choosing the stack first

| App shape | Auth stack |
| --- | --- |
| Monolith with Blade/Livewire | Built-in session auth (`Auth` facade / `auth` middleware) |
| SPA (React/Vue, separate or same repo) | Sanctum SPA mode (cookie session, no tokens in JS) |
| Mobile app or third-party API consumers | Sanctum API tokens (Bearer) |
| Full OAuth2 server needed | Passport (rare) |

For a fast start, install a starter kit (Laravel Breeze is the minimal one) and read its
generated controllers as reference implementations — the Laravel docs explicitly recommend this
as a learning path even when you don't ship it.

## Core concepts

Guards define *how* a request is authenticated (`session`, `sanctum`, custom). Providers define
*where users come from* (Eloquent `App\Models\User` by default, or database). Both configure in
`config/auth.php`. The `users` table needs a >= 60-char password column and a nullable 100-char
`remember_token` — the default 11 migration includes both.

## The auth workflow (no starter kit)

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class LoginController extends Controller
{
    /**
     * Handle an authentication attempt.
     */
    public function authenticate(Request $request): RedirectResponse
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);

        if (Auth::attempt($credentials)) {
            $request->session()->regenerate();   // prevent session fixation — never skip

            return redirect()->intended('dashboard');
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ])->onlyInput('email');
    }

    /**
     * Log the user out of the application.
     */
    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

Extra conditions can ride along: `Auth::attempt(['email' => $e, 'password' => $p, 'active' => 1])`,
or `Auth::attemptWhen([...], fn (User $u) => $u->isNotBanned())`. "Remember me" is the second
`attempt()` argument: `Auth::attempt($credentials, $remember)`.

Other entry points: `Auth::login($user)`, `Auth::loginUsingId(1)`, `Auth::once($credentials)`
(stateless, single request).

## Accessing the authenticated user

```php
use Illuminate\Support\Facades\Auth;

$user = Auth::user();
$id = Auth::id();

if (Auth::check()) { /* logged in */ }

// or via the request...
$user = $request->user();
```

## Protecting routes

```php
Route::get('/flights', fn () => /* ... */)->middleware('auth');
Route::get('/admin', fn () => /* ... */)->middleware('auth:admin');   // specific guard
Route::get('/profile', fn () => /* ... */)->middleware('auth.basic'); // HTTP Basic prompt
Route::get('/settings', fn () => /* ... */)->middleware('password.confirm'); // re-ask password
```

In Laravel 11, where `auth` sends guests is configured in `bootstrap/app.php` — not in an
`Authenticate` middleware subclass:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->redirectGuestsTo(fn (Request $request) => route('login'));
})
```

Login throttling ships with the starter kits (locked out 1 minute after several failures, keyed
to email+IP). For hand-rolled logins, add `throttle:6,1` middleware yourself.

## Automatic password rehashing (new in 11)

Laravel 11 transparently rehashes user passwords with the current bcrypt work factor when they
authenticate — if you raise `BCRYPT_ROUNDS` over time, existing hashes upgrade on next login
without any user action or migration. Custom user providers implement
`rehashPasswordIfRequired()`.

## Sanctum: installation

```shell
php artisan install:api
```

Installs Sanctum, creates `routes/api.php` (auto-prefixed `/api`), and runs the personal access
token migration. That single command is the Laravel 11 path — don't composer-require Sanctum
manually in a fresh app.

## API tokens (mobile apps / third parties)

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

Route::post('/sanctum/token', function (Request $request) {
    $request->validate([
        'email' => 'required|email',
        'password' => 'required',
        'device_name' => 'required',
    ]);

    $user = User::where('email', $request->email)->first();

    if (! $user || ! Hash::check($request->password, $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    return $user->createToken($request->device_name)->plainTextToken;
});
```

Show the `plainTextToken` exactly once — only the SHA-256 hash is stored. Clients send it as
`Authorization: Bearer <token>`.

Abilities (scopes) and revocation:

```php
$token = $user->createToken('token-name', ['server:update'])->plainTextToken;

if ($user->tokenCan('server:update')) { /* ... */ }

$user->tokens()->delete();                          // revoke all
$request->user()->currentAccessToken()->delete();   // revoke the request's token
$user->tokens()->where('id', $tokenId)->delete();   // revoke one
```

Ability middleware — register aliases in `bootstrap/app.php` (Laravel 11 style):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'abilities' => CheckAbilities::class,        // token must have ALL listed
        'ability' => CheckForAnyAbility::class,      // token must have ANY listed
    ]);
})

// routes/api.php
Route::get('/orders', fn () => /* ... */)->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);
```

Protect routes: `->middleware('auth:sanctum')` (works in both `web.php` and `api.php` — it accepts
either a session cookie or a Bearer token). Prune expired tokens via
`Schedule::command('sanctum:prune-expired --hours=24')->daily();` in `routes/console.php`.

## SPA authentication (cookie-based, no tokens)

Sanctum authenticates first-party SPAs with Laravel's session cookie — CSRF-safe and immune to
token theft via XSS. The SPA and API must share a top-level domain.

Backend (`bootstrap/app.php`):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->statefulApi();
})
```

Add SPA domains to the `stateful` list in `config/sanctum.php`. Frontend flow:

```js
await axios.get('/sanctum/csrf-cookie');   // sets XSRF-TOKEN cookie
await axios.post('/login', credentials);   // your session-based login endpoint
```

Set `axios.defaults.withCredentials = true` (and `withXSRFToken = true`) on the client. Send
`Accept: application/json` and same-domain `Referer`/`Origin` headers.

## Testing authenticated endpoints

```php
use Laravel\Sanctum\Sanctum;

Sanctum::actingAs(User::factory()->create(), ['*']);
// or with limited abilities:
Sanctum::actingAs(User::factory()->create(), ['view-tasks']);
```

## Authorization vs authentication

Authentication = who you are (this file). Authorization = what you may do — policies and gates
(`Gate::define` in `AppServiceProvider`, auto-discovered `app/Policies/*Policy.php`, the `can`
middleware / `$this->authorize()`). Don't build permission checks inside auth guards.
