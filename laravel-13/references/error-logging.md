# Error Handling and Logging

## How exceptions flow

1. `handle()` in `bootstrap/app.php` builds the app with `->withExceptions(...)` — the configuration entry point.
2. Uncaught exceptions are reported (log + any reporters) then rendered (debug → whoops-ish trace page; production → generic error page).
3. `APP_DEBUG=true` (local only — **never in production**, it leaks credentials/stack/source) controls detail.

## Configuring via bootstrap/app.php

```php
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;

->withExceptions(function (Exceptions $exceptions): void {
    // Custom rendering — JSON for APIs, view for web
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return $request->is('api/*')
            ? response()->json(['message' => $e->getMessage()], 400)
            : response()->view('errors.invalid-order', [], 400);
    });

    // Custom reporting (return false to stop the default log channel)
    $exceptions->report(function (PaymentFailedException $e) {
        // metrics/sentry ping
    });

    $exceptions->dontReport([
        InvalidCouponException::class,      // never logged
    ]);

    // Guard against log flooding
    $exceptions->throttle(fn (Throwable $e) => Limit::perMinute(300));

    // Map an exception to an HTTP status
    $exceptions->render(fn (ModelNotFoundException $e, Request $r) => response()->json(['message' => 'Not found'], 404));
});
```

## Reportable/renderable exceptions

Instead of central config, put behavior on the exception class:

```php
class OrderException extends Exception
{
    public function report(): void      // runs instead of default logging; return false to also log
    {
        // notify ops
    }

    public function render($request): Response   // custom response
    {
        return response()->view('errors.order', [], 500);
    }
}
```

Report levels per class: `$exceptions->level(PDOException::class, \Psr\Log\LogLevel::CRITICAL);` in `withExceptions`.

## HTTP exceptions

```php
abort(404, 'Product not found');
abort_if($user->cannot('update', $post), 403);
abort_unless($user->isAdmin(), 403);
```

Custom error pages: create `resources/views/errors/404.blade.php`, `403.blade.php`, `500.blade.php`, or `errors/4xx.blade.php` as a fallback tier. HTTP exception messages are visible to users; don't put secrets in them.

## The exception handler's report() from anywhere

```php
report('Something went wrong');                 // helper — logs without interrupting flow
report($exception);
```

## Logging

Channels configured in `config/logging.php`; default stack = `single` (local) / `daily` (production-friendly). Levels: `emergency, alert, critical, error, warning, notice, info, debug`.

```php
use Illuminate\Support\Facades\Log;

Log::info('User logged in.', ['user_id' => $user->id, 'ip' => $request->ip()]);
Log::error('Payment failed', ['order' => $order->id, 'reason' => $e->getMessage()]);
Log::warning(...); Log::debug(...);

logger('Order shipped', $context);              // global helper
```

**Context discipline:** always pass structured context (arrays), never string-interpolate values into the message — structured fields stay searchable and queryable.

Shared context across all entries: `Log::withContext(['tenant_id' => ...])` (e.g. in middleware).

Named channels and stacks:

```php
Log::channel('daily')->info('...');
Log::stack(['single', 'slack'])->critical('Disk almost full');
```

Monolog customization: `Log::build([...])`, channel handlers/formatters/processors via `withMonologChannel` hooks — use for JSON-structured production logs or custom processors.

On-demand channels:

```php
$channel = Log::build(['driver' => 'single', 'path' => storage_path('logs/billing.log')]);
Log::stack(['slack', $channel])->info('...');
```

## Tailing logs

```bash
php artisan pail          # live tail all logs, filterable
php artisan pail --filter='User.*'
```

Pail is the 11+ built-in log tailer — prefer it over `tail -f storage/logs/laravel.log`.

## Production health & monitoring

- Health check endpoint: `GET /up` (defined in `bootstrap/app.php` `withRouting`).
- `php artisan about` — quick env/cache/driver overview; useful when debugging a server you don't know.
- For deeper observability use first-party tools: Laravel Telescope (dev debugging), Laravel Pulse (production performance insights), Nightwatch (cloud observability). Deploy: `php artisan down` → `composer install --no-dev` → `php artisan migrate --force` → `optimize` → `queue:restart` → `php artisan up`.

## Common debugging loop (use it before guessing)

1. Read the stack trace in `storage/logs/laravel.log` (or `php artisan pail` while reproducing).
2. Reproduce interactively in `php artisan tinker`.
3. Inspect the exact SQL: `Model::toSql()` / `->toRawSql()`, `DB::enableQueryLog()`, or `php artisan db:show`.
4. Check route registration: `php artisan route:list --path=the/path`.
5. Suspect stale caches: `php artisan optimize:clear` — config/route caches mask the true cause surprisingly often.
