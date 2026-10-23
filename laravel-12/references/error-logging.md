# Error Handling and Logging (Laravel 12)

## Table of contents

- [Configuration](#configuration)
- [withExceptions: the fluent exception API](#withexceptions-the-fluent-exception-api)
- [Reportable / renderable exceptions](#reportable-renderable-exceptions)
- [HTTP exceptions and custom error pages](#http-exceptions-and-custom-error-pages)
- [Logging](#logging)

## Configuration

`APP_DEBUG` (`.env`) drives error verbosity: `true` locally, **always `false` in production** — when true, the debug page exposes config values, credentials, and stack traces to end users. `config/app.php`'s `debug` option mirrors it.

There is no `app/Exceptions/Handler.php` in Laravel 12 — all exception customization is fluent, inside `bootstrap/app.php`:

```php
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;

return Application::configure(basePath: dirname(__DIR__))
    // ...
    ->withExceptions(function (Exceptions $exceptions): void {
        // everything below lives here
    })
    ->create();
```

## withExceptions: the fluent exception API

**Custom reporting** (runs in addition to default logging; chain `->stop()` to prevent the default log, or `return false`):

```php
use App\Exceptions\InvalidOrderException;

$exceptions->report(function (InvalidOrderException $e): void {
    // e.g. ship to an external tracker
})->stop();
```

**Log levels per type** — because severity should route to the right channel:

```php
use PDOException;
use Psr\Log\LogLevel;

$exceptions->level(PDOException::class, LogLevel::CRITICAL);
```

**Never report some types** (they still render, just don't pollute logs):

```php
use App\Exceptions\PodcastProcessingException;
use Illuminate\Contracts\Debug\ShouldntReport;

$exceptions->dontReport([
    PodcastProcessingException::class,
]);

// Conditional suppression:
$exceptions->dontReportWhen(function (Throwable $e): bool {
    return $e instanceof PodcastProcessingException && $e->reason() === 'Subscription expired';
});

// Re-enable types the framework ignores by default (404s, 419s):
$exceptions->stopIgnoring(HttpException::class);
```

Or mark the exception class itself — it will never be reported, but still renders:

```php
class PodcastProcessingException extends Exception implements ShouldntReport
{
    //
}
```

**Custom rendering** — return a Response (or nothing to fall through to default rendering):

```php
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

$exceptions->render(function (InvalidOrderException $e, Request $request) {
    return response()->view('errors.invalid-order', status: 500);
});

// e.g. shape 404s for an API prefix:
$exceptions->render(function (NotFoundHttpException $e, Request $request) {
    if ($request->is('api/*')) {
        return response()->json(['message' => 'Record not found.'], 404);
    }
});
```

**Deciding JSON vs HTML globally** — Laravel picks based on `Accept`; override when you need a hard rule:

```php
$exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e): bool {
    if ($request->is('admin/*')) {
        return true;
    }

    return $request->expectsJson();
});
```

**Post-processing every response** — e.g. friendlier 419 (page expired) handling:

```php
use Symfony\Component\HttpFoundation\Response;

$exceptions->respond(function (Response $response): Response {
    if ($response->getStatusCode() === 419) {
        return back()->with(['message' => 'The page expired, please try again.']);
    }

    return $response;
});
```

**Global log context** appended to every exception's log entry (Laravel already adds the user ID):

```php
$exceptions->context(fn (): array => ['tenant' => tenant()->id]);
```

**Deduplication** — when `report()` might be called on the same instance repeatedly, only the first is reported:

```php
$exceptions->dontReportDuplicates();
```

**Throttling reported exceptions** — keep noisy failures from flooding logs/trackers. Return a `Lottery` (sample), a `Limit` (rate limit), or mix:

```php
use App\Exceptions\ApiMonitoringException;
use Illuminate\Broadcasting\BroadcastException;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Lottery;
use Throwable;

$exceptions->throttle(function (Throwable $e) {
    return match (true) {
        $e instanceof BroadcastException => Limit::perMinute(300),
        $e instanceof ApiMonitoringException => Lottery::odds(1, 1000),
        default => Limit::none(),
    };
});
```

Use a `Limit` when a third-party outage would otherwise generate thousands of identical reports a minute; use a `Lottery` when statistical sampling is enough. Key limits with `Limit::perMinute(300)->by($e->getMessage())` to split by cause rather than by class.

## Reportable / renderable exceptions

For exceptions that carry their own behavior, put `report()` / `render()` directly on the class instead of centralizing in `bootstrap/app.php` — Laravel calls them automatically:

```php
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class InvalidOrderException extends Exception
{
    /**
     * Report the exception.
     */
    public function report(): void
    {
        //
    }

    /**
     * Render the exception as an HTTP response.
     */
    public function render(Request $request): Response
    {
        return response()->view('errors.invalid-order', status: 500);
    }
}
```

Return `false` from either method to fall back to the framework's default behavior. Also useful: a `context(): array` method on the exception adds instance-specific data (`['order_id' => $this->orderId]`) to its log entry — cleaner than global context for one-off detail.

The `report($e)` helper reports an exception while continuing the current request — the standard pattern for "log it, but degrade gracefully" inside a try/catch.

## HTTP exceptions and custom error pages

```php
abort(404);
abort(403, 'You do not own this post.');
abort_if(! auth()->user()->isAdmin(), 403);
abort_unless($post->user_id === auth()->id(), 403);
```

Custom error pages are just Blade views named after status codes: `resources/views/errors/404.blade.php` renders for every 404 (the `HttpException` is available as `$exception`). Fallback pages for a whole class: `resources/views/errors/4xx.blade.php` and `5xx.blade.php` cover any unmatched 4xx/5xx — but note they do NOT override 404, 500, and 503, which have dedicated internal pages; define those individually. Publish and customize the defaults: `php artisan vendor:publish --tag=laravel-errors`.

## Logging

Channels are configured in `config/logging.php` (published on demand). The default channel is `stack` (stdout in dev, `storage/logs/laravel.log` in most deploys). Write logs through the facade or helper — never `error_log()`:

```php
use Illuminate\Support\Facades\Log;

Log::emergency($message, $context);
Log::alert($message);
Log::critical($message);
Log::error($message, ['order_id' => $order->id]);
Log::warning($message);
Log::notice($message);
Log::info($message);
Log::debug($message);

logger('Quick info line');                    // helper, defaults to info
logger()->error('Something went wrong');
```

Context passed as the second argument is structured — prefer it over string interpolation so log aggregation can query fields. Contextual logging scopes follow-on entries:

```php
Log::withContext(['request_id' => $request->header('X-Request-Id')]);
Log::channel('slack')->error('Deploy failed');
```

Channel patterns worth knowing: `stack` (fan-out), `daily` (rotated files), `slack`, `syslog`, `stderr` (containers/Cloud), and `emergency` (fallback when the configured channel fails). Testing asserts on the fake: `Log::spy()` or `Log::shouldReceive()->...`.

In production, route exceptions to a real observability tool — the framework's report hooks above are the integration point for Sentry/Flare (their packages register themselves as reporters) — and let `storage/logs` be the fallback, not the monitoring strategy.
