# Laravel 11 Middleware Configuration

Source: https://laravel.com/framework/docs/11.x/middleware.
This is the single largest breaking change between Laravel 10 and 11 — read this before touching
any middleware.

## Why it changed

Laravel 10 apps carried nine framework middleware as app files plus an `app/Http/Kernel.php` that
wired global middleware, groups, and aliases. Laravel 11 moved all of that into the framework and
replaced the kernel with a fluent configuration object,
`Illuminate\Foundation\Configuration\Middleware`, customized inside `bootstrap/app.php`. Middleware
classes you write still live in `app/Http/Middleware` and are unchanged in shape.

## Writing a middleware class

```shell
php artisan make:middleware EnsureTokenIsValid
```

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('/home');
        }

        return $next($request);
    }
}
```

Keep this exact signature: `Request`, `Closure`, return `Symfony\Component\HttpFoundation\Response`.
Work before `$next($request)` is "before" logic; capture the return of `$next($request)` and act on
it for "after" logic. All middleware resolve through the service container, so constructor
type-hints are injected.

Terminable middleware add a `terminate(Request $request, Response $response): void` method that
runs after the response is sent (FastCGI). Laravel resolves a *fresh* instance for `terminate`;
register the class as a singleton in `AppServiceProvider::register()` if you need shared state.

## Registering middleware: the fluent API

Everything happens inside `->withMiddleware(function (Middleware $middleware) { ... })` in
`bootstrap/app.php`.

### Global middleware (every request)

```php
use App\Http\Middleware\EnsureTokenIsValid;

->withMiddleware(function (Middleware $middleware) {
    $middleware->append(EnsureTokenIsValid::class);   // end of global stack
    // $middleware->prepend(Other::class);            // start of global stack
})
```

The default global stack in 11 (framework-owned): `InvokeDeferredCallbacks`, `TrustProxies`,
`HandleCors`, `PreventRequestsDuringMaintenance`, `ValidatePostSize`, `TrimStrings`,
`ConvertEmptyStringsToNull`. Override the whole list with `$middleware->use([...])` — copy the
default list above as your starting point if you do.

### Middleware groups

Append/prepend to the built-in `web` and `api` groups:

```php
use App\Http\Middleware\EnsureTokenIsValid;
use App\Http\Middleware\EnsureUserIsSubscribed;
use Illuminate\Session\Middleware\StartSession;

->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        EnsureUserIsSubscribed::class,
    ]);

    $middleware->api(prepend: [
        EnsureTokenIsValid::class,
    ]);

    // Replace one entry with your own class...
    $middleware->web(replace: [
        StartSession::class => StartCustomSession::class,
    ]);

    // Or remove an entry entirely...
    $middleware->web(remove: [StartSession::class]);
})
```

Default `web` group: `EncryptCookies`, `AddQueuedCookiesToResponse`, `StartSession`,
`ShareErrorsFromSession`, `ValidateCsrfToken`, `SubstituteBindings`.
Default `api` group: just `SubstituteBindings`.

Custom groups use the same methods — `$middleware->appendToGroup('group-name', [...])`,
`prependToGroup(...)` — and full manual control is `$middleware->group('web', [...])`.

### Aliases

Short names usable in route definitions:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class,
    ]);
})
```

Built-in aliases available without any registration: `auth`, `auth.basic`, `auth.session`,
`cache.headers`, `can`, `guest`, `password.confirm`, `precognitive`, `signed`, `throttle`,
`verified`.

### Priority ordering

When assignment order can't be controlled:

```php
$middleware->priority([
    \Illuminate\Cookie\Middleware\EncryptCookies::class,
    \Illuminate\Session\Middleware\StartSession::class,
    // ... framework defaults, then your middleware
]);
```

### Common one-liner customizations

All of these were app-file edits in Laravel 10; all are fluent calls in 11:

```php
->withMiddleware(function (Middleware $middleware) {
    // CSRF exceptions (was VerifyCsrfToken::$except)...
    $middleware->validateCsrfTokens(except: ['stripe/*']);

    // Trusted proxies (was TrustProxies middleware file)...
    $middleware->trustProxies(at: ['192.168.1.1', '10.0.0.0/8']);

    // Trusted hosts (was TrustHosts middleware file)...
    $middleware->trustHosts(at: ['laravel.test']);

    // Input normalization exceptions...
    $middleware->convertEmptyStringsToNull(except: [
        fn (Request $request) => $request->is('admin/*'),
    ]);

    // Use Redis for the "throttle" middleware...
    $middleware->throttleWithRedis();

    // Where "auth" redirects guests (was Authenticate::redirectTo)...
    $middleware->redirectGuestsTo('/login');

    // Sanctum SPA support (was in api group)...
    $middleware->statefulApi();
})
```

## Assigning middleware to routes

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', fn () => /* ... */)->middleware(EnsureTokenIsValid::class);

Route::get('/', fn () => /* ... */)->middleware([First::class, Second::class]);

Route::middleware(['auth', 'subscribed'])->group(function () {
    Route::get('/billing', fn () => /* ... */);
});

// Exclude a middleware from one route inside a grouped set...
Route::get('/profile', fn () => /* ... */)->withoutMiddleware([EnsureTokenIsValid::class]);
```

Note `withoutMiddleware` cannot remove *global* middleware — only route/group middleware.

## Middleware parameters

```php
class EnsureUserHasRole
{
    public function handle(Request $request, Closure $next, string $role): Response
    {
        if (! $request->user()->hasRole($role)) {
            // Redirect...
        }

        return $next($request);
    }
}
```

Parameters are separated with `:` and delimited with `,` when attaching:

```php
Route::put('/post/{id}', fn (string $id) => /* ... */)
    ->middleware(EnsureUserHasRole::class.':editor');

Route::put('/post/{id}', fn (string $id) => /* ... */)
    ->middleware(EnsureUserHasRole::class.':editor,publisher');
```

## Practical guidance

- Prefer a `web(append: [...])` / `api(prepend: [...])` call over redefining groups — redefinition
  must restate the full list and silently drops framework updates.
- Give every app-written middleware a route alias when it's used more than once; class-string
  references are fine for single uses.
- CORS is handled by the global `HandleCors` middleware. Customize via
  `php artisan config:publish cors` then edit `config/cors.php` — not by writing CORS middleware.
- To debug what runs on a route: `php artisan route:list -v` (shows middleware; `-vv` expands
  groups).
