# Middleware Configuration (Laravel 12)

Middleware filters and inspects HTTP requests. In Laravel 12 — unlike Laravel ≤10 — there is **no HTTP Kernel**: middleware is registered fluently in `bootstrap/app.php` via the `Illuminate\Foundation\Configuration\Middleware` object. Every registration example below happens inside that closure.

## Writing a middleware

```shell
php artisan make:middleware EnsureTokenIsValid
```

The generated class lives in `app/Http/Middleware`. The `handle` contract is strict: take `Request` + `Closure`, return `Symfony\Component\HttpFoundation\Response`:

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

Work **before** the app by acting first; work **after** by capturing the response: `$response = $next($request); ... return $response;`. Middleware are resolved through the service container, so constructor type-hints are injected — but keep them stateless; a fresh instance runs per request unless you register a singleton.

**Parameters**: extra arguments arrive after `$next`, passed from the route with `:` and commas:

```php
public function handle(Request $request, Closure $next, string $role, string ...$roles): Response
{
    if (! $request->user()?->hasRole($role, ...$roles)) {
        abort(403);
    }

    return $next($request);
}
```

```php
Route::put('/post/{post}', [PostController::class, 'update'])
    ->middleware(EnsureUserHasRole::class.':editor,publisher');
```

**Terminable middleware**: define `terminate(Request $request, Response $response): void` to run after the response is sent (FastCGI/Octane). Laravel resolves a *fresh* instance for `terminate` — bind the class as a singleton in `AppServiceProvider::register()` if you need shared state between `handle` and `terminate`.

## Registering in bootstrap/app.php

All of the following is `bootstrap/app.php`:

```php
use App\Http\Middleware\EnsureTokenIsValid;
use App\Http\Middleware\EnsureUserIsSubscribed;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    // ...
    ->withMiddleware(function (Middleware $middleware): void {
        // Global stack: append to the end...
        $middleware->append(EnsureTokenIsValid::class);

        // ...or prepend to the very front:
        $middleware->prepend(EnsureTokenIsValid::class);

        // Custom group:
        $middleware->appendToGroup('billing', [EnsureUserIsSubscribed::class]);
        $middleware->prependToGroup('billing', [EnsureTokenIsValid::class]);

        // Aliases (short names usable on routes):
        $middleware->alias([
            'subscribed' => EnsureUserIsSubscribed::class,
        ]);

        // Sanctum SPA cookie auth on api routes:
        $middleware->statefulApi();

        // Use Redis for the throttle middleware:
        $middleware->throttleWithRedis();
    })
    ->create();
```

Why aliases matter: route middleware strings like `'auth'`, `'guest'`, `'throttle:api'` are aliases, and framework defaults are only available that way. Defining your own keeps route files readable and stable when class names refactor.

### Adjusting the built-in web/api groups

The `web()` and `api()` helpers accept named `append`, `prepend`, `remove`, and `replace` arguments — far safer than redefining the groups wholesale:

```php
$middleware->web(append: [EnsureUserIsSubscribed::class]);

$middleware->api(prepend: [EnsureTokenIsValid::class]);

$middleware->web(remove: [
    \Illuminate\Session\Middleware\StartSession::class,
]);

$middleware->web(replace: [
    \Illuminate\Session\Middleware\StartSession::class => \App\Http\Middleware\StartCustomSession::class,
]);
```

Only use `$middleware->group('web', [...])` or `$middleware->use([...])` (full manual redefinition of the groups / global stack) when you genuinely need total control — you take ownership of every future default after that.

**Default global stack** (for reference): `InvokeDeferredCallbacks`, `TrustProxies`, `HandleCors`, `PreventRequestsDuringMaintenance`, `ValidatePostSize`, `TrimStrings`, `ConvertEmptyStringsToNull`. Note `TrimStrings` + `ConvertEmptyStringsToNull` — this is why "optional" request fields must be `nullable` in validation rules.

**Default `web` group**: `EncryptCookies`, `AddQueuedCookiesToResponse`, `StartSession`, `ShareErrorsFromSession`, `ValidateCsrfToken`, `SubstituteBindings`.
**Default `api` group**: `SubstituteBindings` only (stateless by design).

### Default aliases you can rely on

`auth`, `auth.basic`, `auth.session`, `cache.headers`, `can`, `guest`, `password.confirm`, `precognitive`, `signed`, `throttle`, `verified`. (`subscribed` exists for Spark billing when installed.) Sanctum's `abilities`/`ability` aliases are **not** default — define them yourself if you use token abilities:

```php
use Laravel\Sanctum\Http\Middleware\CheckAbilities;
use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;

$middleware->alias([
    'abilities' => CheckAbilities::class,
    'ability' => CheckForAnyAbility::class,
]);
```

### Middleware priority

When middleware must run in a specific order regardless of declaration order, set a priority list. Use sparingly — it's a smell if you need it often:

```php
$middleware->priority([
    \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
    \Illuminate\Cookie\Middleware\EncryptCookies::class,
    \Illuminate\Session\Middleware\StartSession::class,
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
    \Illuminate\Auth\Middleware\Authorize::class,
]);
```

## Assigning to routes

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', [ProfileController::class, 'show'])
    ->middleware(EnsureTokenIsValid::class);

Route::middleware([First::class, Second::class])->group(function (): void {
    // ...
});

// Exclude middleware from a nested route or group:
Route::middleware([EnsureTokenIsValid::class])->group(function (): void {
    Route::get('/profile', fn () => view('profile'))
        ->withoutMiddleware([EnsureTokenIsValid::class]);
});

Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function (): void {
    // ...
});
```

`withoutMiddleware` only removes **route** middleware — it cannot exempt a route from **global** middleware (that's what global means).

## Auth middleware patterns worth knowing

- `auth` — redirect guests to the login route (web) or return 401 JSON (api). Guard-specific: `auth:web`, `auth:sanctum`.
- `auth.basic` — HTTP Basic challenge, no persistence.
- `can:update,post` — authorizes via policies (`{post}` is a route-model-bound value). Prefer policies over bespoke middleware for authorization; see `can` usage in `routing-controllers.md`.
- `verified` — requires a verified email (works with the starter kits' `MustVerifyEmail` setup).
- `throttle:60,1` or `throttle:uploads` — rate limiting. Named limiters are defined in `AppServiceProvider::boot()` (covered in `routing-controllers.md`).

## Testing middleware

Middleware runs as part of the HTTP layer, so feature tests exercise it for free (`$this->get('/profile')` passes through the stack). `withoutMiddleware()` and `withoutMiddleware([SomeMiddleware::class])` on a test disable it when isolating a controller concern. Assert the outcomes, not the middleware call order.
