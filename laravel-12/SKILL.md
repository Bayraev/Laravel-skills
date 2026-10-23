---
name: laravel-12
description: Production-grade Laravel 12.x development standards (PHP 8.2+, Feb 2025 release) covering the streamlined skeleton with bootstrap/app.php configuration, Eloquent, routing, middleware, form requests and validation, migrations, queues, Pest/PHPUnit testing, Blade, Sanctum, and the React/Vue/Svelte/Livewire starter kits. Use this skill whenever the user works on any Laravel 12 application — building features, fixing bugs, writing tests, or answering architecture questions — even if they just say "Laravel" without a version, as long as the project targets 12.x (laravel/framework ^12.0 in composer.json). Trigger on mentions of Artisan, Eloquent, Blade, Inertia, Livewire, Fortify, Sanctum, Pest, Vite in a Laravel 12 context. Do NOT use for Laravel 11 or 13.x projects, plain PHP, Symfony, or WordPress.
---

# Laravel 12 Development

## Version snapshot (why this skill exists)

Laravel 12 was released **February 24, 2025** and requires **PHP 8.2–8.5**. It is deliberately a **maintenance release**: it keeps the Laravel 11 streamlined skeleton and adds almost no breaking changes, so most Laravel 11 apps upgrade without code changes. What IS new in 12:

- **New starter kits** for React, Svelte, Vue, and Livewire (Inertia 2 + TypeScript + Tailwind + shadcn/ui for the JS kits; Flux UI for Livewire), plus optional **WorkOS AuthKit** variants (social login, passkeys, SSO). Laravel Breeze and Jetstream no longer receive updates.
- **`HasUuids` now generates UUIDv7** (sortable) — `HasVersion4Uuids` preserves old behavior.
- Requires **Carbon 3**, **PHPUnit ^11.0**, **Pest ^3.0**.
- Default local filesystem disk root moved to `storage/app/private`.

Support status (as of late 2026): bug fixes ended **August 13, 2026**; security fixes run until **February 24, 2027**. Laravel 13 (Q1 2026, PHP 8.3+) is current — do not suggest 13-only features in a 12 codebase.

Source: https://laravel.com/docs/12.x/releases and https://laravel.com/docs/12.x/upgrade

## First step: confirm the version

Before writing code, check `composer.json` for `"laravel/framework": "^12.0"`. If the project is on 11 or 13, stop and say so — this skill's guidance targets 12.x specifically (a sibling skill may exist for other versions). If the project has no composer.json yet and the user wants a new app, it's a Laravel 12 project only if they say so; otherwise ask.

## The mental model that prevents most mistakes

Laravel 12 apps are configured through **`bootstrap/app.php`**, not through `config/*.php` edits or `Kernel.php` (which does not exist). Nearly everything an app-level developer customizes lives in one fluent builder:

```php
<?php

use Illuminate\Foundation\Application;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        // ...
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        // ...
    })
    ->create();
```

If you find yourself editing `app/Http/Kernel.php`, `app/Exceptions/Handler.php`, or `RouteServiceProvider`, you are writing Laravel 10-style code. Route files default to `routes/web.php` and `routes/console.php` only; `routes/api.php` appears after `php artisan install:api` (which also installs Sanctum), and `routes/channels.php` after `php artisan install:broadcasting`. Config files are slim and can be published on demand with `config:publish` (e.g. `php artisan config:publish cors`).

## Workflow

1. **Locate** the relevant code using the conventions in `references/structure-conventions.md` (paths are predictable — that is the point of the framework).
2. **Read the real code** around the change before writing anything: neighbors, service provider bindings, existing patterns.
3. **Pick the right layer** using the decision table below — don't put validation in controllers when a form request exists, don't put middleware logic in controllers, and don't fight the container.
4. **Generate, don't hand-write**: use `php artisan make:*` stubs so naming, namespaces, and placement match the framework's expectations.
5. **Verify**: `php artisan route:list` after route changes, `php artisan migrate --pretend` (or a test DB) for migrations, and run tests with `php artisan test` / `vendor/bin/pest`.

## Decision table — where to go next

Read only the reference you need; each is self-contained.

| Task / symptom | Reference |
|---|---|
| "Where does X go?", folder layout, new app anatomy, service providers | `references/structure-conventions.md` |
| Middleware (global, groups, aliases, params), CORS, throttle config, Sanctum stateful API | `references/middleware-config.md` |
| Routes, controllers, route model binding, rate limiting, API routes | `references/routing-controllers.md` |
| Request input, validation rules, form requests, custom rules, 422 responses | `references/requests-validation.md` |
| Models, relationships, casts, UUID/ULID keys, eager loading, strictness | `references/eloquent-orm.md` |
| Migrations, schema builder, seeders, factories | `references/migrations-seeding.md` |
| Blade templates, components, layouts, Vite asset bundling | `references/blade-frontend.md` |
| Starting a new app, React/Vue/Svelte/Livewire kits, Fortify auth, WorkOS, Inertia | `references/starter-kits-frontend.md` |
| Queued jobs, dispatching, workers, failures, retries | `references/queues-jobs.md` |
| Pest/PHPUnit tests, HTTP tests, database tests, fakes | `references/testing.md` |
| Artisan commands, scheduling, tinker, optimize/cache | `references/artisan-cli.md` |
| Exception handling, reporting, custom error pages, logging channels | `references/error-logging.md` |
| What changed in 12 vs 11 vs 13, upgrade specifics, support dates | `references/version-notes.md` |

## Non-negotiable conventions

These exist to keep the codebase idiomatic and reviewable; they are what a senior Laravel reviewer will expect.

- **PHP 8.2+ idioms**: constructor property promotion, `readonly` properties, enums, `match`, first-class callable syntax, named arguments where the docs use them. Strict types only if the file already has them.
- **Never** edit files under `vendor/`. Publish config/views with `config:publish` / `vendor:publish` instead.
- **Routes**: name every route (`->name('posts.show')`); redirect via `to_route()`.
- **Controllers**: keep them thin — validate via form requests, push work to services/actions or queued jobs, return typed responses (`Response`, `RedirectResponse`, `JsonResponse`).
- **Models**: fill out `$fillable` explicitly (never `$guarded = []` unless you can justify it), declare `$casts`, and prefer `HasUuids`/`HasUlids` over app-side ID generation.
- **Validation**: request data is validated at the boundary (form request), then `$request->validated()` flows inward — no re-validation in the domain.
- **Env access**: use `config()` everywhere except config files; never `env()` outside `config/*` (config caching breaks `env()` lookups elsewhere).
- **Testing**: prefer feature tests exercising HTTP + database over unit tests of framework-wired code. Most tests belong in `tests/Feature`.

## Quick correctness checks

- `php artisan route:list` — routes registered as expected?
- `php artisan test` — green?
- `php artisan about` — environment, cache states, drivers at a glance.
- `php artisan model:show App\Models\Post` — attributes, casts, relations of a model.
- `php artisan migrate --pretend` — see the SQL a migration will run before running it.

## Common Laravel 12 gotchas

- **`env()` returns null in production** when config is cached (`php artisan config:cache` is common in deployment). Only `env()` inside `config/` files is safe.
- **Route caching**: route closures cannot be cached — controllers only once `route:cache` is used. In 12, when duplicate route names exist, the **first** registered route wins (matching cached behavior).
- **SVG uploads fail `image` validation** in 12 — the rule now excludes SVGs by default (`image:allow_svg` or `File::image(allowSvg: true)` to re-enable).
- **`Storage::disk('local')`** writes to `storage/app/private` in 12 unless the disk is explicitly configured — file paths changed from 11.
- **Named arguments** on Laravel methods are not covered by the backward-compatibility promise — avoid calling framework methods with named args you don't see in the docs.
- **N+1 queries**: enable `Model::preventLazyLoading(! $this->app->isProduction())` in `AppServiceProvider::boot()` so lazy loads throw during development instead of silently multiplying queries.

## When you're unsure

Ground truth is the official 12.x documentation: `https://laravel.com/docs/12.x/{topic}` (e.g. `validation`, `eloquent`, `queues`, `starter-kits`). If a fact is version-sensitive, check `references/version-notes.md` first — it lists every verified 12.x delta with source URLs.
