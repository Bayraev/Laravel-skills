# Laravel 11 Error Handling and Logging

Sources: https://laravel.com/framework/docs/11.x/errors and
https://laravel.com/framework/docs/11.x/logging.

## Error handling now lives in `bootstrap/app.php`

Laravel 11 removed `app/Exceptions/Handler.php`. Configure exceptions with
`->withExceptions(function (Exceptions $exceptions) { ... })`, where `$exceptions` is an
`Illuminate\Foundation\Configuration\Exceptions` instance.

## Configuration

`APP_DEBUG=true` (local) shows full stack traces; `APP_DEBUG=false` (production — non-negotiable)
hides them. Leaving it true in production exposes config values and credentials to end users.

## Reporting exceptions

```php
use App\Exceptions\InvalidOrderException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions) {
    // Custom reporting for a type (Laravel infers the type from the closure hint)...
    $exceptions->report(function (InvalidOrderException $e) {
        // Send to an external tracker...
    });

    // ...and stop default logging for it if handled:
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    })->stop();

    // Ignore types entirely (they still render)...
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);

    // Re-report types Laravel ignores by default (404s, 419 CSRF, etc.)...
    $exceptions->stopIgnoring(HttpException::class);

    // Per-type log level...
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);

    // Global context added to every exception log entry...
    $exceptions->context(fn () => [
        'request_id' => request()->header('Request-Id'),
    ]);

    // Deduplicate report() calls for the same exception instance...
    $exceptions->dontReportDuplicates();
})
```

The `report($e)` helper reports without rendering an error page — for logging and continuing:

```php
public function isValid(string $value): bool
{
    try {
        // ...
    } catch (Throwable $e) {
        report($e);

        return false;
    }
}
```

## Rendering exceptions

```php
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

->withExceptions(function (Exceptions $exceptions) {
    // Custom response for your exception...
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });

    // Override built-in exceptions (return nothing = default rendering)...
    $exceptions->render(function (NotFoundHttpException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json(['message' => 'Record not found.'], 404);
        }
    });

    // Control when responses render as JSON instead of HTML...
    $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
        return $request->is('api/*') || $request->expectsJson();
    });

    // Last-word tweak of the final response...
    $exceptions->respond(function (Response $response) {
        if ($response->getStatusCode() === 419) {
            return back()->with(['message' => 'The page expired, please try again.']);
        }

        return $response;
    });
})
```

Exception classes can also define their own `report()` / `render(Request $request)` methods —
Laravel calls them automatically. A `context(): array` method on the exception adds per-exception
log context (e.g. `['order_id' => $this->orderId]`). Mark exceptions
`implements ShouldntReport` to make them never-reportable at the class level.

## Throttling reported exceptions

High-volume exception reporting can flood logs and trackers:

```php
use Illuminate\Support\Lottery;
use Illuminate\Cache\RateLimiting\Limit;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->throttle(function (Throwable $e) {
        return match (true) {
            $e instanceof BroadcastException => Limit::perMinute(300),   // rate limit
            $e instanceof ApiMonitoringException => Lottery::odds(1, 1000), // sample 0.1%
            default => Limit::none(),
        };
    });
})
```

## HTTP errors and custom pages

```php
abort(404);
abort(403, 'You may not edit this order.');
abort_if(! auth()->check(), 403);
abort_unless($user->isAdmin(), 403);
```

Custom error pages: `resources/views/errors/404.blade.php` (per status),
`resources/views/errors/4xx.blade.php` and `5xx.blade.php` (fallbacks — note 404/500/503 have
dedicated internal pages and need individual overrides). Publish defaults with
`php artisan vendor:publish --tag=laravel-errors`.

## Logging basics

Channels live in `config/logging.php`; the default channel is `stack`. Eight RFC 5424 levels:
`emergency`, `alert`, `critical`, `error`, `warning`, `notice`, `info`, `debug`.

```php
use Illuminate\Support\Facades\Log;

Log::info('User {id} failed to login.', ['id' => $user->id]);
Log::error('Payment failed', ['order' => $order->id]);

Log::channel('slack')->warning('Disk usage high');
Log::stack(['single', 'slack'])->critical('The system is down!');
Log::build(['driver' => 'single', 'path' => storage_path('logs/custom.log')])->info('...');
```

Channel drivers: `single`, `daily` (rotates, keep N days via `LOG_DAILY_DAYS`), `slack`,
`syslog`, `errorlog`, `papertrail`, `stack`, `monolog`, `custom`.

Context that flows through a request — add it once in middleware:

```php
Log::withContext(['request-id' => $requestId]);     // current channel + future channels in this process
Log::shareContext(['request-id' => $requestId]);    // across ALL channels
```

Laravel automatically adds the authenticated user's ID to exception log context.

Deprecation warnings can get their own channel: `LOG_DEPRECATIONS_CHANNEL` env or a
`deprecations` channel in `config/logging.php`; `LOG_DEPRECATIONS_TRACE=true` adds stack traces.

## Monolog customization

Tap into an existing channel's Monolog instance:

```php
'single' => [
    'driver' => 'single',
    'tap' => [App\Logging\CustomizeFormatter::class],
    'path' => storage_path('logs/laravel.log'),
],
```

```php
class CustomizeFormatter
{
    public function __invoke(Logger $logger): void
    {
        foreach ($logger->getHandlers() as $handler) {
            $handler->setFormatter(new LineFormatter('[%datetime%] %channel%.%level_name%: %message% %context% %extra%'));
        }
    }
}
```

Arbitrary Monolog handlers/formatters via the `monolog` driver (`handler`, `with`, `formatter`,
`processors` keys); full control via the `custom` driver with a `via` factory class.

## Tailing logs with Pail

```shell
composer require laravel/pail --dev

php artisan pail                                # tail all logs
php artisan pail -vv                            # + exception stack traces
php artisan pail --filter="QueryException"      # by type/message/trace
php artisan pail --message="User created"
php artisan pail --level=error
php artisan pail --user=1
```

Pail works across drivers (even Sentry/Flare) unlike raw `tail`. Requires PHP 8.2+ and PCNTL —
which a Laravel 11 app always has.
