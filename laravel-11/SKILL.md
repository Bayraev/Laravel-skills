---
name: laravel-11
description: Production-grade Laravel 11.x development standards and version-accurate conventions — the streamlined skeleton (bootstrap/app.php, no HTTP/console kernels, no exception handler), the fluent middleware API, SQLite + database-driver defaults, routing, Eloquent, validation, queues, testing, Sanctum, and Artisan. Use this skill whenever working in any Laravel 11 project (composer.json requires laravel/framework ^11.0), whenever the user mentions Laravel 11, and for ANY Laravel task involving middleware registration, bootstrap/app.php, rate limiters, health routes, per-second throttling, Sanctum API tokens, or the slim skeleton — even if they never say "Laravel 11" out loud. Do NOT use for Laravel 10, 12, 13, Lumen, or plain PHP projects; those versions have different file structures and APIs.
---

# Laravel 11 Development

This skill produces idiomatic, version-accurate Laravel 11 code. Laravel 11 (March 2024) restructured
the application skeleton more aggressively than any release since Laravel 5, so the most common
failure mode is writing Laravel 10-style code: an `app/Http/Kernel.php` that doesn't exist, a
`RouteServiceProvider` that was deleted, middleware registered in the wrong place. Everything in
this skill is grounded in the official 11.x docs (https://laravel.com/framework/docs/11.x/{topic}).

## Version snapshot

Verify the project version before writing code — read `composer.json` and look for
`"laravel/framework": "^11.0"`. If it says `^10.0`, `^12.0` or higher, stop and tell the user this
skill targets 11 specifically.

| Fact | Value |
| --- | --- |
| Released | March 12, 2024 |
| PHP requirement | 8.2 – 8.4 (8.2 minimum) |
| Bug fixes until | September 3, 2025 (security fixes until March 12, 2026) |
| Default database | SQLite (`database/database.sqlite`), created and migrated on install |
| Session / cache / queue driver | `database` by default — no Redis/extra services needed |
| Default test framework | Pest or PHPUnit, chosen at `laravel new` |

Laravel 11 apps are recognizable by their slim skeleton. If you see any of these files, the app
was upgraded from an older Laravel (or is not 11): `app/Http/Kernel.php`, `app/Console/Kernel.php`,
`app/Exceptions/Handler.php`, `app/Providers/{Route,Event,Auth,Broadcast}ServiceProvider.php`.
They still *work* if present (upgraded apps keep them), but never create them in a fresh 11 app.

## What changed from 10 to 11 — never do these the old way

| Task | Laravel 10 (do NOT do) | Laravel 11 (do this) |
| --- | --- | --- |
| Register middleware / aliases / groups | Edit `app/Http/Kernel.php` | `->withMiddleware(...)` in `bootstrap/app.php` |
| Configure exception handling | `app/Exceptions/Handler.php` | `->withExceptions(...)` in `bootstrap/app.php` |
| Register routes / health route | RouteServiceProvider | `->withRouting(...)` in `bootstrap/app.php` |
| Schedule tasks | `$schedule->command(...)` in `app/Console/Kernel.php` | `Schedule::command(...)` facade in `routes/console.php` |
| Enable API routes | Uncomment `api.php` wiring | `php artisan install:api` (installs Sanctum too) |
| Register events/listeners | `EventServiceProvider` `$listen` | Event discovery is automatic; manual registration in `AppServiceProvider` |
| Define model casts | `protected $casts = [...]` property | `protected function casts(): array` method (property still works) |
| Rate limit granularity | `Limit::perMinute()` only | also `Limit::perSecond()` |

Full detail with code: `references/version-notes.md`.

## Workflow

1. **Confirm the version** — `composer.json` → `laravel/framework`. Also check `.env` for
   `DB_CONNECTION` (SQLite by default in 11) before assuming MySQL.
2. **Read the relevant reference file** from the table below before writing code. The references
   carry the version-specific detail; this file only carries the map.
3. **Write Laravel 11-idiomatic PHP**: strict comparison, typed method signatures
   (`public function store(Request $request): RedirectResponse`), constructor property promotion,
   named arguments where the docs use them.
4. **Verify with the framework**, not by eyeballing: `php artisan route:list -v`, `php artisan about`,
   `php artisan migrate --pretend`, `vendor/bin/pest` / `php artisan test`. If the project has
   Laravel Pint, run `./vendor/bin/pint --dirty` before finishing.

## Reference map

Read only what the task needs — these files are loaded on demand, not upfront.

| Task / topic | Read |
| --- | --- |
| Where things live now; the slim skeleton; `bootstrap/app.php` anatomy; single `AppServiceProvider` | `references/structure-conventions.md` |
| Registering global/group/alias middleware, CSRF exceptions, trust proxies/hosts, the `Middleware` fluent API — **the biggest 10→11 change** | `references/middleware-config.md` |
| Route files, model binding, enum binding, rate limiters, API routes, route caching | `references/routing-controllers.md` |
| Request input retrieval, validation, form requests, the 422 JSON contract | `references/requests-validation.md` |
| Eloquent models, the `casts()` method, relations, strictness, `model:show` | `references/eloquent-orm.md` |
| Schema builder, the 3 consolidated default migrations, altering columns, seeding | `references/migrations-seeding.md` |
| Blade templates, components, CSRF/method directives, old input, Vite | `references/blade-frontend.md` |
| Queued jobs, the `Queueable` trait, job middleware, workers, failed jobs, testing interactions | `references/queues-jobs.md` |
| Pest/PHPUnit setup, HTTP feature tests, database tests, fakes, parallel runs | `references/testing.md` |
| Artisan commands, closure commands in `routes/console.php`, scheduling, the `/up` health route, Tinker | `references/artisan-cli.md` |
| `withExceptions` error handling, reporting/rendering config, log channels, Pail | `references/error-logging.md` |
| Session auth, guards, route protection, password rehashing, Sanctum API/SPA/mobile tokens | `references/auth-basics.md` |
| Every verified version-specific fact with its official source URL | `references/version-notes.md` |

## Non-negotiable conventions

These hold across every Laravel 11 codebase; the references explain the why.

- **Never create a Kernel or Handler class.** All application wiring lives in `bootstrap/app.php`
  (`withRouting`, `withMiddleware`, `withExceptions`, `withCommands`, `withBroadcasting`) or in
  `app/Providers/AppServiceProvider.php`. This is the defining convention of Laravel 11.
- **Controllers extend the local abstract `Controller`** (`app/Http/Controllers/Controller.php`), which
  in Laravel 11 is an empty abstract class with no traits. Add `AuthorizesRequests` /
  `ValidatesRequests` per-controller only if you actually need them.
- **Validate through form requests** (`php artisan make:request StorePostRequest`) for anything
  non-trivial; use `$request->validate([...])` for quick cases. Never hand-roll validation loops.
- **Mass assignment protection stays on**: define `$fillable` (or `$guarded = []` only with
  hand-crafted arrays). In non-production, enable `Model::preventSilentlyDiscardingAttributes()` so
  silent discards become exceptions.
- **`APP_DEBUG` must be `false` in production.** It is the single most common critical Laravel
  misconfiguration and leaks config values to end users.
- **Route-name everything you redirect to**, and redirect after successful POST/PUT/DELETE
  (`to_route(...)`) — the POST/redirect/GET pattern.
- **Write tests for features, not just units.** Laravel's own docs' guidance: most tests should be
  feature tests. In-memory SQLite (`:memory:`) is heavily optimized in 11 and is the fast default.
- **Run `php artisan route:cache`, `config:cache`, `view:cache` only at deploy time**, never during
  local development — cached routes silently ignore new route definitions.

## Environment quirks worth remembering

- New 11 apps run with `DB_CONNECTION=sqlite` and a `database/database.sqlite` file that already has
  the default tables migrated. Don't install MySQL to make a demo work.
- The default queue connection in a new app is `sync` (runs inline); the `database` driver needs the
  `jobs` table, which ships in the default `0001_01_01_000002_create_jobs_table.php` migration.
- `php artisan install:api` is the sanctioned way to add `routes/api.php` + Sanctum; don't hand-wire
  the file into `bootstrap/app.php` yourself.
- If a vendor package asks you to register a provider or facade, the 11-way is the
  `bootstrap/providers.php` file (packages) or `AppServiceProvider` — not a new provider class unless
  the package requires one.
