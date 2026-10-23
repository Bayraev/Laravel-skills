# Version Notes — Verified Laravel 10.x Facts

Every fact below was checked against the official Laravel 10.x documentation (base: `https://laravel.com/framework/docs/10.x/`). Use this file to settle "is that 10 or 11?" questions quickly, and as the citation list when reviewing version-sensitive code.

## Support window and requirements

| Fact | Detail | Source |
| --- | --- | --- |
| Release date | February 14, 2023 | [releases](https://laravel.com/framework/docs/10.x/releases) |
| PHP requirement | PHP 8.1.0 minimum; supports 8.1–8.3 | [releases](https://laravel.com/framework/docs/10.x/releases), [upgrade](https://laravel.com/framework/docs/10.x/upgrade) |
| Composer requirement | Composer 2.2.0 minimum | [upgrade](https://laravel.com/framework/docs/10.x/upgrade) |
| Bug fixes until | August 6, 2024 | [releases](https://laravel.com/framework/docs/10.x/releases) |
| Security fixes until | February 4, 2025 (now EOL) | [releases](https://laravel.com/framework/docs/10.x/releases) |
| New project | `composer create-project "laravel/laravel:^10.0" example-app` | [installation](https://laravel.com/framework/docs/10.x/installation) |
| Default database in fresh skeleton | MySQL (`DB_CONNECTION=mysql` at 127.0.0.1); SQLite is an opt-in (`DB_CONNECTION=sqlite`, then `php artisan migrate`) | [installation](https://laravel.com/framework/docs/10.x/installation) |

## Skeleton and wiring (differs from Laravel 11+)

| Fact | Detail | Source |
| --- | --- | --- |
| Native types | Argument and return types added to **all** application skeleton methods and framework stubs; redundant doc-block type-hints removed | [releases](https://laravel.com/framework/docs/10.x/releases) |
| HTTP kernel | `app/Http/Kernel.php` registers global middleware (`$middleware`), groups (`$middlewareGroups`: `web`/`api`), and aliases (`$middlewareAliases`) | [middleware](https://laravel.com/framework/docs/10.x/middleware) |
| `$middlewareAliases` rename | `$routeMiddleware` was renamed to `$middlewareAliases` in new 10.x apps (renaming existing apps optional) | [upgrade](https://laravel.com/framework/docs/10.x/upgrade) |
| Route provider | `App\Providers\RouteServiceProvider` loads `routes/web.php` + `routes/api.php`, applies `web`/`api` middleware groups, auto-prefixes `/api`, hosts default API rate limiter in `boot()` | [routing](https://laravel.com/framework/docs/10.x/routing) |
| Exception handler | `App\Exceptions\Handler` with a `register()` method using `reportable()` / `renderable()`; optional `$withoutDuplicates = true` | [errors](https://laravel.com/framework/docs/10.x/errors) |
| Scheduling | Console Kernel `schedule(Schedule $schedule): void` in `app/Console/Kernel.php` | [artisan / structure](https://laravel.com/framework/docs/10.x/structure) |
| `registerPolicies()` | Called automatically by the framework now — remove manual calls from `AuthServiceProvider::boot()` when upgrading from 9 | [upgrade](https://laravel.com/framework/docs/10.x/upgrade) |
| Default middleware priority | Includes `HandlePrecognitiveRequests` (Laravel Precognition, for Inertia live validation) | [middleware](https://laravel.com/framework/docs/10.x/middleware) |

## New in Laravel 10

| Feature | Detail | Source |
| --- | --- | --- |
| Process service | `Process` facade: `run()`, `start()`, `timeout()`, `forever()`, `env()`, `path()`, `input()`, `Process::concurrently(Pool $pool)`, `Process::fake()` + `assertRan()` for tests | [releases](https://laravel.com/framework/docs/10.x/releases), [processes](https://laravel.com/framework/docs/10.x/processes) |
| Laravel Pennant | First-party feature flags: `Feature::define()`, `Feature::active()`, `@feature` Blade directive; `array` and `database` drivers | [releases](https://laravel.com/framework/docs/10.x/releases) |
| Test profiling | `php artisan test --profile` lists the slowest tests | [releases](https://laravel.com/framework/docs/10.x/releases) |
| Pest scaffolding | `laravel new example-app --pest` | [releases](https://laravel.com/framework/docs/10.x/releases) |
| Generator prompts | All `make:` commands accept zero arguments and prompt interactively (e.g. `php artisan make:controller`) | [releases](https://laravel.com/framework/docs/10.x/releases) |
| `dispatch_sync` | `Bus::dispatchNow` / `dispatch_now()` removed; use `Bus::dispatchSync` / `dispatch_sync()` | [upgrade](https://laravel.com/framework/docs/10.x/upgrade) |

## Behavioral changes vs 9.x (upgrade traps)

- **Eloquent `$dates` property removed** — cast dates via `protected $casts = ['deployed_at' => 'datetime'];`. ([upgrade](https://laravel.com/framework/docs/10.x/upgrade))
- **Monolog 3** — logging integrations (Bugsnag, Rollbar, custom Monolog usage) must be Monolog 3 / Laravel 10 compatible. ([upgrade](https://laravel.com/framework/docs/10.x/upgrade))
- **`dispatch()` on a non-`ShouldQueue` class** now returns a `PendingBatch`; use `dispatch_sync()` to get the old `handle()` return value. ([upgrade](https://laravel.com/framework/docs/10.x/upgrade))
- **`->change()` column modification requires `doctrine/dbal`** when using SQLite (always) and for `dropColumn`/`renameIndex` on SQLite < 3.35 and column renames on older MySQL/PostgreSQL releases. Laravel 11 removed this requirement — do not assume native `change()` here. ([migrations](https://laravel.com/framework/docs/10.x/migrations))

## Things absent in 10 (present in 11+) — do not use

- `bootstrap/app.php` fluent configuration (`withMiddleware`, `withExceptions`, `withSchedule`, `withCommands`)
- `protected function casts(): array` method on models — 10.x uses the `protected $casts` property ([eloquent-mutators](https://laravel.com/framework/docs/10.x/eloquent-mutators))
- `php artisan install:api`, `php artisan install:broadcasting`, the `/up` health route
- Per-file `routes/console.php` `Schedule::` registration; slim skeleton directories

## Ecosystem notes for 10.x projects (as of Oct 2026)

- Laravel 10 apps receive no security fixes since Feb 2025 — recommend planning an upgrade to a supported line for anything internet-facing; this skill's job is correctness while the app remains on 10.
- Laravel Boost (AI-agent tooling) explicitly supports Laravel 10 on PHP 8.1+ if the team wants richer agent context. ([installation](https://laravel.com/framework/docs/10.x/installation))
- First-party packages (Sanctum, Horizon, Scout, Socialite, etc.) must be pinned to versions compatible with `laravel/framework ^10.0` — check each package's 10.x branch in its upgrade notes when bumping.

## Quick adjudication guide

When reviewing code or advice and unsure which version it targets, check these tells:

| See this | It means | Correct for 10.x? |
| --- | --- | --- |
| `app/Http/Kernel.php` with `$middlewareAliases` | Laravel 10 skeleton | Yes — canonical |
| `bootstrap/app.php` containing `->withMiddleware(` | Laravel 11+ | No — rewrite against `Kernel.php` |
| `protected $casts = [...]` property | Works everywhere; 10.x docs style | Yes |
| `protected function casts(): array` | Laravel 11+ | No — convert to property |
| `dispatch_now(` | Laravel ≤9 | No — `dispatch_sync()` |
| `Bus::dispatchNow(` | Laravel ≤9 | No — `Bus::dispatchSync()` |
| `->change()` migration with `doctrine/dbal` in composer.json | Laravel 10 pattern | Yes — verify dbal installed |
| `Schedule::` facade inside `routes/console.php` | Laravel 11+ | No — `app/Console/Kernel.php::schedule()` |
| `php artisan install:api` | Laravel 11+ | No — wire route files via `RouteServiceProvider` |
| `/up` health route | Laravel 11+ | No |
| `Handler` class with `register()` in `app/Exceptions/` | Laravel 10 | Yes — canonical |
| `LaravelReverb`, `laravel/pulse` integrations on 10 | Later-era packages | Check package's Laravel 10 support individually |

When still uncertain, trust the 10.x docs page over any blog post or training-data memory, and prefer code already in the project as the style anchor.
