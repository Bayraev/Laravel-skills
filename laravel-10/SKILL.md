---
name: laravel-10
description: Laravel 10.x development standards, patterns, and workflows — routing, controllers, Eloquent, migrations, validation, Blade, queues, testing, and Artisan for the Laravel 10 classic skeleton (PHP 8.1+, app/Http/Kernel.php middleware registration). Use this skill whenever the user works in a Laravel 10 codebase — composer.json requires laravel/framework ^10.0, or they mention "Laravel 10" / "L10" — including writing or reviewing controllers, models, migrations, routes, middleware, form requests, jobs, Blade views, or tests; debugging Laravel errors; running artisan commands; or asking "how do I X in Laravel". Also use it when upgrading an app to 10.x or porting Laravel 11+ habits back to 10. Do NOT use for Laravel 11/12/13 projects (different skeleton, bootstrap/app.php middleware, casts() method) or plain PHP without Laravel.
---

# Laravel 10 Development

You are working with **Laravel 10.x** (released February 14, 2023; requires PHP 8.1–8.3; security support ended February 4, 2025 — it remains in real-world use). Laravel 10 is the last release line with the classic full skeleton: `app/Http/Kernel.php` for middleware, `app/Console/Kernel.php` for scheduling, and `App\Exceptions\Handler` for exception handling all still exist and are the canonical places for that configuration. Because Laravel 11+ moved much of this into `bootstrap/app.php`, your training data likely contains newer patterns — this skill exists to keep you on the 10.x path.

## Confirm the version before relying on this skill

Check once, early:

```bash
grep '"laravel/framework"' composer.json   # expect "laravel/framework": "^10.0"
php artisan --version                       # expect Laravel Framework 10.x
```

If the project is 11+ (or a sibling version skill is installed), use that version's conventions instead. Laravel 10 and 11 look alike but wire differently; guessing wrong produces code that fails at runtime, not at lint time.

## Version snapshot (what makes 10.x, 10.x)

- **PHP 8.1.0 minimum, Composer 2.2.0 minimum**; supports PHP 8.1/8.2/8.3. Write PHP 8.1-compatible code: constructor promotion, `readonly` properties, enums, `match`, first-class callables, and full native argument/return types are all available; `readonly` classes and standalone `true`/`false`/`null` types (PHP 8.2+) are not.
- **Native types everywhere in the skeleton**: every generated stub (controllers, jobs, middleware, migrations, commands, tests) ships with argument and return types. Match that style — type every method you write, including `: void` and `: array`.
- **Middleware lives in `app/Http/Kernel.php`**: global stack in `$middleware`, groups in `$middlewareGroups` (`web`/`api`), aliases in `$middlewareAliases` (renamed from `$routeMiddleware` in 10.0). Do not create Laravel 11-style `bootstrap/app.php` middleware config.
- **Exception handling lives in `app/Exceptions/Handler.php`** via `register()` + `reportable()`/`renderable()` callbacks.
- **`App\Providers\RouteServiceProvider`** applies the `web`/`api` middleware groups, the `/api` prefix, and hosts API rate-limiter definitions.
- **Column modification (`->change()`) requires `doctrine/dbal`** on SQLite and on older MySQL/PostgreSQL. It is not database-native as in Laravel 11.
- **New in 10**: `Process` facade (external processes), Laravel Pennant (feature flags), `php artisan test --profile`, interactive prompts on all `make:` commands, `Bus::dispatchSync()`/`dispatch_sync()` replacing the removed `dispatch_now()`.
- **Monolog 3** under the hood; Eloquent's `$dates` property is gone — use `$casts`.

Full list with sources: read `references/version-notes.md`.

## Hard version boundaries — Laravel 11+ patterns that break on 10

These are the mistakes an agent most often makes by defaulting to newer habits. Each is a runtime failure or silently-wrong config on 10:

| Do not write (Laravel 11+) | Write instead (Laravel 10) |
| --- | --- |
| `->withMiddleware(...)` in `bootstrap/app.php` | `$middleware` / `$middlewareGroups` / `$middlewareAliases` in `app/Http/Kernel.php` |
| `withExceptions(...)` in `bootstrap/app.php` | `App\Exceptions\Handler::register()` with `reportable()`/`renderable()` |
| `Schedule::` in `routes/console.php` | `$schedule->...` in `app/Console/Kernel.php::schedule()` |
| `protected function casts(): array` on a model | `protected array $casts = [...]` property |
| `php artisan install:api` / `install:broadcasting` | Add the route file + provider wiring manually (see `references/routing-controllers.md`) |
| `->change()` without `doctrine/dbal` | `composer require doctrine/dbal` first (see `references/migrations-seeding.md`) |
| `#[\Override]`, PHP 8.2+ syntax (`readonly` classes, DNF types) | PHP 8.1 syntax (promotion, `readonly` properties, enums, `match`) |

## Workflow

**Generate, don't hand-write.** Laravel 10's `make:` commands produce correctly-typed stubs and prompt for missing arguments interactively. Creating any framework class by hand risks missing a type, import, or trait the ecosystem assumes:

```bash
php artisan make:controller PhotoController --resource      # resource controller
php artisan make:controller ProvisionServer --invokable     # single-action controller
php artisan make:model Flight -mfsc                         # model + migration + factory + seeder + controller
php artisan make:request StorePostRequest                   # form request
php artisan make:middleware EnsureTokenIsValid
php artisan make:job ProcessPodcast
php artisan make:command                  # no name given → 10.x prompts interactively
```

Then edit the generated file. This mirrors the docs and keeps diffs reviewable.

**Migrate in small steps.** One concern per migration, generated with `make:migration`. Never edit a migration that teammates or production have already run — add a new migration instead. In dev, `php artisan migrate:fresh --seed` is the reset lever (destroys data — never against production).

**Test what you build.** Laravel 10 ships PHPUnit (or Pest, if the project chose `--pest`). Feature-test HTTP endpoints, unit-test isolated logic, and run the suite with `php artisan test` (not raw phpunit) for the nicer output — and `--profile` to catch slow tests. `RefreshDatabase` + `migrate:fresh` per test is the default pattern; see `references/testing.md`.

**Keep the toolchain honest.** Run `php artisan route:list` after route changes, `php artisan migrate:status` after migration work, and `vendor/bin/pint` (if installed) before committing. `php artisan about` gives a one-screen environment diagnostic.

**Debug with the right instruments.** `php artisan tinker` for interactive probing (`User::where(...)->first()`), `laravel.log` via `storage/logs` (`tail -f storage/logs/laravel.log`), and `APP_DEBUG=true` local-only stack traces. Never leave `APP_DEBUG=true` in production — it leaks credentials and paths.

## Project conventions

- **PSR-4**: `App\` maps to `app/`. Generated classes land in conventional directories (`app/Http/Controllers`, `app/Models`, `app/Jobs`, `app/Policies`, ...) — follow the existing placement when adding code; the docs' structure page is summarized in `references/structure-conventions.md`.
- **One class per file**, file name = class name, namespaces match directory depth.
- **Routes**: web routes in `routes/web.php` (session, CSRF), API routes in `routes/api.php` (stateless, auto-prefixed `/api`), console closures in `routes/console.php`.
- **Controllers stay thin**: validate via a Form Request, delegate to a model/action/job, return a Response/View. Fat controllers are the most common review finding in Laravel codebases.
- **Config via `.env` + `config/*.php`**, accessed with `config('key')` — never `env()` outside config files (config is cached in production, `env()` returns null there).
- **Facades** (`Route`, `DB`, `Cache`, `Process`) are the docs' default style; constructor injection is equally valid when you prefer it for testability. Match the file you're editing.

## Decision table — where to go next

Read only the reference you need, when you need it:

| Task / topic | Read |
| --- | --- |
| Verified 10.x version facts, upgrade-from-9 gotchas, sources | `references/version-notes.md` |
| Skeleton layout, Kernel, providers, config, `.env` | `references/structure-conventions.md` |
| Routes, route model binding, controllers (resource/invokable), API routes | `references/routing-controllers.md` |
| Input handling, `$request->validate()`, Form Requests, error format | `references/requests-validation.md` |
| Middleware (Kernel registration), auth, gates/policies, Sanctum tokens | `references/middleware-auth.md` |
| Models, `$casts`, relationships, queries, collections | `references/eloquent-orm.md` |
| Schema builder, migrations, factories, seeders, the dbal caveat | `references/migrations-seeding.md` |
| Blade templates, components, layouts, Vite assets | `references/blade-frontend.md` |
| Queue jobs, `dispatch_sync`, chaining, workers, failed jobs | `references/queues-jobs.md` |
| Feature/HTTP tests, `php artisan test`, database testing | `references/testing.md` |
| Artisan commands, generators, tinker, scheduling, optimization | `references/artisan-cli.md` |
| `App\Exceptions\Handler`, reportable/renderable, logging channels | `references/error-logging.md` |
| `Process` facade (new in 10): run/pool/test external processes | `references/process-service.md` |

## When facts are uncertain

The 10.x docs are canonical and stable: `https://laravel.com/framework/docs/10.x/{topic}` (e.g. `routing`, `eloquent`, `middleware`). If a task touches an area this skill doesn't cover (broadcasting, notifications, mail, events, storage, billing), fetch the matching 10.x page rather than trusting memory of another version — that is the single highest-value habit for Laravel 10 work.
