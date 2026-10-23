# Laravel 11 Structure and Conventions

Source: https://laravel.com/framework/docs/11.x/structure and
https://laravel.com/framework/docs/11.x/releases (streamlined application structure).
Verified against the official 11.x application skeleton.

## Why the skeleton is slim

Laravel 11 moved framework plumbing (kernels, middleware, exception handler) into the framework
itself so the application contains only code you actually own. A fresh 11 app has roughly 100 fewer
files of boilerplate than a fresh 10 app. The practical consequence: there are exactly two places
application-wide wiring happens — `bootstrap/app.php` and `AppServiceProvider` — and the framework
sane-defaults everything else.

## The root directory

| Directory | Purpose |
| --- | --- |
| `app/` | Your code. Starts with only `Http/`, `Models/`, `Providers/` |
| `bootstrap/` | `app.php` (application configuration) + `cache/` (framework caches) |
| `config/` | Configuration files, read via `config('key.sub')` |
| `database/` | Migrations, factories, seeders, and the default `database.sqlite` |
| `public/` | `index.php` front controller and built assets — the only web-exposed dir |
| `resources/` | Blade views (`views/`), raw CSS/JS, language files |
| `routes/` | `web.php` and `console.php` only by default |
| `storage/` | Logs, compiled views, sessions, file caches; `storage/app/public` for user files |
| `tests/` | `Feature/` and `Unit/` test directories |

Create the `public/storage` symlink with `php artisan storage:link` when serving user uploads from
`storage/app/public`.

## The `app/` directory grows on demand

Only `Http/`, `Models/`, and `Providers/` exist after install. Other directories are created the
first time the matching `make:` command runs:

- `app/Console/Commands` — after `php artisan make:command`
- `app/Jobs` — after `php artisan make:job`
- `app/Events`, `app/Listeners` — after `make:event` / `event:generate`
- `app/Mail`, `app/Notifications`, `app/Policies`, `app/Rules`, `app/Broadcasting` — likewise

Do not pre-create these directories "for completeness"; Laravel conventions create them lazily, and
empty directories confuse the structure readers expect.

## `bootstrap/app.php` — the application configuration file

This file is the heart of a Laravel 11 app. It returns a configured `Application`:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

What each method accepts (full detail in the topic-specific references):

- `withRouting`: `web:`, `api:`, `commands:`, `health:`, `apiPrefix:`, `then:` (register extra
  routes after defaults), `using:` (take over route registration entirely).
- `withMiddleware`: the fluent middleware API — see `middleware-config.md`.
- `withExceptions`: report/render/throttle configuration — see `error-logging.md`.
- `withCommands([...])`: extra command directories or classes (see `artisan-cli.md`).
- `withBroadcasting(...)`: broadcasting channel routes with custom middleware.

## Routes directory

`routes/web.php` — stateful web routes, automatically wrapped in the `web` middleware group
(session, CSRF, cookie encryption).

`routes/console.php` — closure-based console commands AND the schedule:

```php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Schedule;

Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
})->purpose('Send a marketing email to a user');

Schedule::command('emails:send')->daily();
```

`routes/api.php` and `routes/channels.php` do NOT exist by default. Install them:

```shell
php artisan install:api           # creates routes/api.php, installs Sanctum, adds api wiring
php artisan install:broadcasting  # creates routes/channels.php
```

## Providers: one, and that's enough

A fresh 11 app ships a single provider, `app/Providers/AppServiceProvider.php`:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        //
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        //
    }
}
```

Everything the removed providers did is either automatic (event discovery, route model bindings,
policy discovery) or belongs here:

- Event → listener registration (when discovery can't find it): `Event::listen(...)` in `boot`.
- Gates: `Gate::define(...)` in `boot` (policies are auto-discovered).
- Rate limiters: `RateLimiter::for(...)` in `boot`.
- Strictness switches: `Model::preventLazyLoading(! $this->app->isProduction())` in `boot`.
- Container bindings: in `register`, never `boot`.

Package service providers are listed in `bootstrap/providers.php` (not in `config/app.php` —
that file no longer has a `providers` array in 11).

## The base controller

```php
<?php

namespace App\Http\Controllers;

abstract class Controller
{
    //
}
```

It no longer extends Laravel's internal `Controller`, and the `AuthorizesRequests` /
`ValidatesRequests` traits are gone from the base. Add them to individual controllers when needed:

```php
class PostController extends Controller
{
    use AuthorizesRequests;
    // ...
}
```

## Naming and placement conventions

- Controllers: singular `PascalCase` + `Controller` (`PostController`, `PaymentIntentController`).
- Models: singular, in `app/Models` — table name is snake_case plural by convention.
- Middleware: `EnsureX` / `VerifyX` imperative names in `app/Http/Middleware`.
- Form requests: `StorePostRequest` / `UpdatePostRequest` verbs matching the action.
- Events past tense (`UserRegistered`), listeners imperative (`SendWelcomeEmail`).
- One class per file, PSR-4, `App\` maps to `app/` via Composer.

## Configuration: environment first

Config values come from `.env` via `env()` calls inside `config/*.php` files. Read config at
runtime with `config('app.locale')` — never call `env()` outside a config file, because
`config:cache` compiles config and `env()` returns `null` in cached mode. That is the most common
subtle production bug in Laravel deployments.

Config files can be trimmed: 11 ships a minimal set, and `php artisan config:publish <name>`
(e.g. `config:publish cors`) publishes a vendor-managed config file into `config/` when you need
to customize it.
