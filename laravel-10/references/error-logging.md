# Error Handling and Logging

Sources: [errors](https://laravel.com/framework/docs/10.x/errors), [logging](https://laravel.com/framework/docs/10.x/logging). **Laravel 10-specific:** exception handling is configured in `app/Exceptions/Handler.php` via `register()` — Laravel 11 replaced this file with `bootstrap/app.php` fluent config, which does not exist here. Logging runs on **Monolog 3**.

## Configuration

- `APP_DEBUG=true` → full stack traces in responses; local only. In production (`APP_DEBUG=false`) errors render the generic 500 page while still logging full detail.
- Error detail respects `debug` in `config/app.php` (driven by `APP_DEBUG`).
- Report some errors, ignore others, via the exception handler's `$dontReport` list.

## The exception handler (`app/Exceptions/Handler.php`)

```php
namespace App\Exceptions;

use App\Exceptions\InvalidOrderException;
use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;
use Throwable;

class Handler extends ExceptionHandler
{
    /** Exceptions never reported. */
    protected $dontReport = [
        InvalidOrderException::class,
    ];

    /** Report an exception instance only once. */
    protected $withoutDuplicates = true;

    public function register(): void
    {
        // Custom reporting (still logged by default — ->stop() or `return false;` to prevent)
        $this->reportable(function (InvalidOrderException $e) {
            // e.g. notify an external service; return false to stop default logging
        })->stop();

        // Custom rendering (return a Response or null to fall through to default)
        $this->renderable(function (InvalidOrderException $e, Request $request) {
            if ($request->is('api/*')) {
                return response()->json(['message' => 'Invalid order'], 422);
            }

            return null;   // fall through
        });
    }
}
```

Reportable closures are type-hinted — Laravel dispatches on the exception class. Multiple reportables run in registration order.

## HTTP exceptions

```php
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

abort(404);                          // NotFoundHttpException → error view / JSON
abort(403, 'You do not own this order.');
abort_if($user->is_banned, 403);
abort_unless($user->is_admin, 403);
```

Error views: `resources/views/errors/404.blade.php`, `500.blade.php`, or `errors/4xx.blade.php` as a fallback group. API requests (expecting JSON) get `{ "message": "...", "exception": ... }` instead. Custom exceptions may implement `HttpExceptionInterface` (use `Symfony\Component\HttpKernel\Exception\HttpException`) to control status + headers.

The built-in mapping: `NotFoundHttpException` → 404, `AuthenticationException` → login redirect / 401 JSON, `AuthorizationException` → 403, `ValidationException` → redirect back / 422 JSON, `ModelNotFoundException` → 404. `ThrottleRequestsException` → 429.

## Reporting helpers

```php
use Illuminate\Support\Facades\Log;
use Throwable;

report($exception);                  // report without rendering (script keeps running)
report_if($shouldReport, $e);
report_unless($condition, $e);
Log::error('Payment failed', ['order_id' => $orderId]);   // structured context array
```

Exception log context: the authenticated user id is added automatically. Add global context on the handler:

```php
// App\Exceptions\Handler — global context added to every exception's log entry
class Handler extends ExceptionHandler
{
    public function context(): array
    {
        return ['tenant_id' => tenant('id')];
    }
}
```

Deduplicate repeats: `$withoutDuplicates = true` (above) — noisy repeated exceptions report once.

## Logging

Channels in `config/logging.php`; default stack `single`/`daily` via `LOG_CHANNEL`. Per-channel drivers: `single`, `daily` (rotating files in `storage/logs/laravel.log`), `slack`, `syslog`, `errorlog`, `papertrail`, `stderr`, custom Monolog handlers.

```php
use Illuminate\Support\Facades\Log;

Log::emergency($message, $context);   // also alert(), critical(), error(),
Log::warning(...);  Log::notice(...); //  notice(), info(), debug()
Log::channel('slack')->error('Deploy failed');
Log::stack(['single', 'slack'])->info('Deploy started');
```

```env
LOG_CHANNEL=daily
LOG_LEVEL=debug          # minimum level written (emergency→debug)
LOG_DAILY_DAYS=14
```

## Practical debugging workflow for Laravel 10 apps

1. Reproduce locally; watch the log in a second terminal: `tail -f storage/logs/laravel.log`.
2. If the log has the exception, read the first stack frame inside `app/` — that's your code, even if the exception came from a vendor class.
3. If nothing logged, the failure may be earlier: check `php artisan about` (cache/config state), `php artisan optimize:clear`, web server error log, or a swallowed exception (`try`/`catch` with empty catch, `rescue()` swallowing).
4. Use tinker to poke the failing call in isolation: `php artisan tinker --execute="..."`.
5. When the bug is data-dependent, reproduce with a factory in a failing test first, then fix — the test then guards the regression.

The `report()` helper and `Log::` calls should carry context arrays (ids, input) — a bare message string is far harder to act on later.
