# Middleware and bootstrap/app.php Configuration

Laravel 11+ removed `app/Http/Kernel.php`; all middleware and exception handling is configured fluently in `bootstrap/app.php`. This is unchanged in 13 and is the single place to look.

## bootstrap/app.php shape

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',                    // health-check route (zero-downtime deployments)
    )
    ->withMiddleware(function (Middleware $middleware): void {
        // middleware configuration — see below
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        // exception rendering/chain customization
    })->create();
```

`api: __DIR__.'/../routes/api.php'` is added to `withRouting` automatically when you run `php artisan install:api`. `->withSchedule(...)` and `->withCommands(...)` closures are also available for console-side wiring.

## Default middleware stacks (know what's already there — don't re-add)

Global stack (every request): `ValidatePathEncoding`, `InvokeDeferredCallbacks`, `TrustProxies`, `HandleCors`, `PreventRequestsDuringMaintenance`, `ValidatePostSize`, `TrimStrings`, `ConvertEmptyStringsToNull` (plus optional `TrustHosts`).

`web` group: `EncryptCookies`, `AddQueuedCookiesToResponse`, `StartSession`, `ShareErrorsFromSession`, **`PreventRequestForgery`**, `SubstituteBindings`.

`api` group: `SubstituteBindings` (Sanctum's `EnsureFrontendRequestsAreStateful` and `throttle:api` are pre-commented, enabled by `install:api`).

Default aliases you can use by name: `auth`, `auth.basic`, `auth.session`, `cache.headers`, `can`, `guest`, `password.confirm`, `precognitive`, `signed`, `throttle`, `verified` (+ `subscribed` when Spark is installed).

## Registering middleware (the `Middleware` object API)

```php
use App\Http\Middleware\EnsureTokenIsValid;
use Illuminate\Session\Middleware\StartSession;

->withMiddleware(function (Middleware $middleware): void {
    // Global
    $middleware->append(EnsureTokenIsValid::class);      // end of global stack
    $middleware->prepend(EnsureTokenIsValid::class);     // start

    // Groups — append/prepend/replace/remove by name
    $middleware->web(append: [EnsureUserIsSubscribed::class]);
    $middleware->api(prepend: [EnsureTokenIsValid::class]);
    $middleware->web(replace: [StartSession::class => StartCustomSession::class]);
    $middleware->web(remove: [StartSession::class]);

    // Arbitrary groups
    $middleware->appendToGroup('group-name', [First::class]);
    $middleware->prependToGroup('group-name', [Second::class]);

    // Aliases for use in route definitions
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class,
    ]);

    // Execution priority (rarely needed)
    $middleware->priority([...]);
    $middleware->appendToPriorityList(after: StartSession::class, append: EnsureTokenIsValid::class);
})
```

**Rule of thumb:** use `web(...)`/`api(...)` adjustments for tweaks; use `$middleware->group('web', [...])` only when you want to redefine a group's entire contents manually (you then own the whole list).

## Request forgery protection (CSRF) — the 13 configuration

`PreventRequestForgery` (default in `web`) checks the browser's `Sec-Fetch-Site` header first: same-origin requests pass without token verification. Older/insecure requests fall back to session-token validation (the classic `_token`/`X-CSRF-TOKEN` mechanism — `@csrf` in Blade still works exactly as before). Failure is a **419** (token path) or **403** (origin-only mode).

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(
        originOnly: true,          // skip token fallback entirely (403 on failure) — HTTPS required
        allowSameSite: true,       // accept subdomain origins (dashboard.example.com ← example.com)
        except: ['stripe/*', 'http://example.com/webhooks/*'],
    );
})
```

Notes: origin verification only happens over HTTPS (browsers don't send `Sec-Fetch-Site` otherwise). Prefer excluding webhooks by placing them outside the `web` group (e.g. in `api.php`) rather than via `except`.

## Writing a middleware class

```bash
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
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('/home');
        }

        return $next($request);
    }
}
```

- Work *before* the request by acting before `$next($request)`; *after* by capturing its return value.
- Middleware resolve via the service container — constructor type-hints are injected.
- **Parameters:** `handle(Request $request, Closure $next, string $role, ...)`; pass in route definitions as `EnsureUserHasRole::class.':editor,publisher'` or use the alias.
- **Terminable middleware:** add `terminate(Request $request, Response $response): void` for work after the response is sent (FastCGI). `terminate` gets a *fresh* instance — register a singleton in `AppServiceProvider::register()` if you need shared state.

## Assigning to routes and controllers

```php
Route::get('/profile', [ProfileController::class, 'show'])->middleware('auth');

// Exclusions
Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/', fn () => /* ... */);
    Route::get('/open', fn () => /* ... */)->withoutMiddleware([EnsureTokenIsValid::class]);
});
```

On controllers (preferred, colocated — 13 style):

```php
use Illuminate\Routing\Attributes\Controllers\Middleware;

#[Middleware('auth')]
class UserController
{
    #[Middleware('subscribed')]           // merged with class-level
    public function index() { /* ... */ }
}
```

Method-level closure middleware is also supported: `#[Middleware(static function (Request $request, Closure $next) { return $next($request); })]`. See `routing-controllers.md` for the full attribute set including `#[Authorize]` and `#[WithoutMiddleware]`.

## Exceptions in bootstrap/app.php

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', [], 404);
    });
    $exceptions->report(function (PaymentFailedException $e) {
        // custom reporting (sentry, metrics) — return false to stop default logging
    });
    $exceptions->dontReport([...]);        // never log these types
    $exceptions->throttle(fn (Throwable $e) => Limit::perMinute(300)); // log flooding guard
});
```

See `error-logging.md` for the full exception-handling model.
