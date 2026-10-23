---
name: laravel-13
description: "Laravel 13.x (current major, released March 2026) development standards — structure, middleware, routing, Eloquent, validation, queues, testing, Blade, and the new AI/agent tooling. Use this skill for ANY work on a Laravel 13 application: creating projects, writing or reviewing controllers, models, migrations, form requests, jobs, tests, or Blade views; upgrading from Laravel 12; debugging Laravel errors; or using Laravel 13's AI SDK, Boost, JSON:API resources, or vector search. Trigger even when the user just says \"Laravel\" without a version if composer.json requires laravel/framework ^13.0 — and check composer.json if the version is unclear. Do NOT use for Laravel 11/12 or non-Laravel PHP projects."
---

# Laravel 13 Development

## Version snapshot (verified against laravel.com/framework/docs/13.x, Oct 2026)

- **Released:** March 17, 2026. Bug fixes until Q3 2027, security fixes until March 17, 2028.
- **PHP requirement:** 8.3 – 8.5 (minimum 8.3). The docs' installer (`php.new`) ships PHP 8.5. Use modern PHP syntax: constructor promotion, readonly, enums, first-class callables, property hooks where useful.
- **Supported databases:** MariaDB 10.3+, MySQL 5.7+, PostgreSQL 10.0+, SQLite 3.26.0+, SQL Server 2017+. SQLite is the default for new apps.
- **Upgrade effort:** Intentionally small. Laravel 13 prioritized minimal breaking changes; most 12.x apps upgrade with near-zero code changes. Use constraint `^13.0`.
- **Era themes:** AI-native workflows (first-party AI SDK, Boost, `laravel.com/for/agents`), attribute-first configuration (PHP attributes across controllers, jobs, Eloquent, validation, testing), JSON:API resources, and semantic/vector search.

### What is genuinely new in 13 (do not confuse with 12)

1. **Laravel AI SDK** (`composer require laravel/ai`) — unified agents, tools, embeddings, images, audio, vector stores. See `references/ai-tooling.md`.
2. **JSON:API resources** — `JsonApiResource` with `$attributes`/`$relationships`, sparse fieldsets, `?include=` support.
3. **`PreventRequestForgery` middleware** replaces token-only CSRF: `Sec-Fetch-Site` origin verification first, token fallback. Configurable via `$middleware->preventRequestForgery(...)`.
4. **Queue routing** — `Queue::route(Job::class, connection: ..., queue: ...)` and `Queue::forward(...)` in a service provider.
5. **Expanded PHP attributes** — `#[Middleware]`/`#[Authorize]`/`#[WithoutMiddleware]` on controllers; `#[Tries]`, `#[Backoff]`, `#[Timeout]`, `#[FailOnTimeout]`, `#[MaxExceptions]`, `#[UniqueFor]`, `#[DebounceFor]`, `#[WithoutRelations]` on jobs; `#[Table]`, `#[WithoutTimestamps]`, `#[DateFormat]`, `#[WithoutIncrementing]`, `#[UseResource]`, `#[UseResourceCollection]` on models/resources; `#[StopOnFirstFailure]`, `#[FailOnUnknownFields]`, `#[RedirectTo]`, `#[ErrorBag]` on form requests; `#[UnitTest]` in tests.
6. **Eloquent model conventions moved to attributes** — table name, key, key type, incrementing, timestamps, date format are now declared with `#[Table(...)]` and friends instead of protected properties (properties still work, but attributes are the documented style).
7. **Vector / semantic search** — `vector()` migration columns, `whereVectorSimilarTo()`, `Str::of(...)->toEmbeddings()`, pgvector / MariaDB 11.7.
8. **`Cache::touch($key, $ttl)`** — extend a cache item's TTL without re-reading it.
9. **Model→resource convention** — `$model->toResource()` and `$collection->toResourceCollection()` auto-discover resource classes.

Full details with source URLs: `references/version-notes.md`.

### Idioms to UNLEARN (pre-13 habits that are now wrong)

| Old habit (Laravel ≤12) | Laravel 13 way |
|---|---|
| `class Job implements ShouldQueue { use Dispatchable, InteractsWithQueue, Queueable, SerializesModels; }` | `use Queueable;` — the single `Illuminate\Foundation\Queue\Queueable` trait covers all four |
| `VerifyCsrfToken` / `ValidateCsrfToken` | `Illuminate\Foundation\Http\Middleware\PreventRequestForgery` (in default `web` group) |
| `app/Http/Kernel.php` for middleware | `bootstrap/app.php` → `->withMiddleware(...)` fluent API |
| Eloquent `$table`, `$primaryKey`, `$timestamps`, `$dateFormat` properties | `#[Table(...)]`, `#[WithoutTimestamps]`, `#[DateFormat(...)]` attributes (properties still supported) |
| `php artisan serve` as the only dev command | `composer run dev` — runs server + queue worker + Vite together |
| Manual `new UserResource($user)` everywhere | `$user->toResource()` / `User::all()->toResourceCollection()` convention |
| `now()->addMinutes(30)` | Carbon 3 fluent: `now()->plus(minutes: 30)` (both work; docs use `plus`) |

## Workflow

1. **Orient.** Read `composer.json` (framework constraint, installed packages) and `php artisan --version` output if available. Confirm Laravel 13; if the project is on 11/12, say so — this skill's guidance may not apply.
2. **Install dev tooling once per project** (see `references/ai-tooling.md`): `composer require laravel/boost --dev && php artisan boost:install`. Boost gives agents MCP tools for schema, routes, logs, docs search. Check for `AGENTS.md`/`CLAUDE.md`/`.ai/guidelines/` and follow project conventions over generic defaults.
3. **Place code by convention** (see `references/structure-conventions.md`). Laravel's predictability is why it's agent-friendly — don't invent new locations.
4. **Read the relevant reference file** (table below) before writing code in that domain. Verify APIs against it; do not rely on pre-13 memory for anything attribute- or AI-related.
5. **Verify with the framework:** `php artisan route:list`, `php artisan migrate --pretend`, `php artisan test`, `php -l`. Boost's Tinker tool is ideal for checking hypotheses against the real app.
6. **Test what you write.** New feature code deserves a Pest test (default: `php artisan make:test`, most tests are Feature tests).

## Decision table — which reference to read

| Task / topic | Read |
|---|---|
| Project layout, where a class goes, route files, service providers | `references/structure-conventions.md` |
| `bootstrap/app.php`, middleware registration/groups/aliases, exception handling config, CSRF/forgery config | `references/middleware-config.md` |
| Routes, controllers (incl. attributes, resource controllers), rate limiting, model binding | `references/routing-controllers.md` |
| Request input, form requests, validation rules, custom rules | `references/requests-validation.md` |
| Models, relationships, casts, attributes, API resources, JSON:API resources | `references/eloquent-orm.md` |
| Migrations, schema builder, vector columns, seeding, factories | `references/migrations-seeding.md` |
| Blade components, layouts, Vite, starter kits, Inertia/Livewire choice | `references/blade-frontend.md` |
| Jobs, queues, job middleware, batching, routing, workers, Redis/Horizon | `references/queues-jobs.md` |
| Pest/PHPUnit tests, HTTP tests, database tests, fakes, parallel runs | `references/testing.md` |
| Artisan commands (writing + common commands), scheduling, caching | `references/artisan-cli.md` |
| Exception reporting/rendering, logging channels, Pail, health checks | `references/error-logging.md` |
| Laravel Boost, LSP, `/for/agents`, AI SDK (agents, tools, embeddings), vector search | `references/ai-tooling.md` |
| Exact version facts, upgrade notes, source URLs | `references/version-notes.md` |

## Ground rules that keep code idiomatic

- **Prefer attributes** for middleware, authorization, and job settings when supported — they colocate behavior with the class and are the 13-era documented style.
- **Always validate via form requests** for non-trivial endpoints; use `#[FailOnUnknownFields]` on APIs to reject unexpected payload keys (defense in depth against mass-assignment-style issues — still fill `$fillable`).
- **Queued jobs use `Queueable`**, declare retry behavior with attributes (`#[Tries]`, `#[Backoff]`), and set routing centrally with `Queue::route(...)` instead of sprinkling `->onQueue()` at every dispatch site.
- **Return resources from APIs** (`toResource()` / `toResourceCollection()`), never raw models, so serialization stays intentional.
- **Write migrations as anonymous classes** (`return new class extends Migration`), keep them reversible, and use `foreignId()->constrained()` for relationships.
- **Explain choices with docblocks sparingly** — Laravel code is self-documenting; docblock only when intent isn't obvious.
- When a documented API differs from memory, **trust `https://laravel.com/framework/docs/13.x/{topic}`** (or the raw markdown at `raw.githubusercontent.com/laravel/docs/13.x/{topic}.md`) — fetch it rather than guessing.

## Common task recipes (quick answers)

**New app:** `laravel new example-app` (or with a starter kit: `laravel new my-app`, choose React/Svelte/Vue/Livewire; community kit: `laravel new my-app --using=vendor/kit`). Then `composer run dev`. Install API routes with `php artisan install:api` (adds `routes/api.php` + Sanctum).

**Typical endpoint:** route in `routes/web.php`/`api.php` → thin controller (`php artisan make:controller PhotoController --model=Photo --resource --requests`) → form request with rules → `$model->toResource()` response.

**Typical background work:** `php artisan make:job ProcessPodcast` → `Queueable` + `handle()` with DI → `ProcessPodcast::dispatch($model)` → route it in `AppServiceProvider::boot()` with `Queue::route(ProcessPodcast::class, queue: 'podcasts')` → worker via `php artisan queue:work` (Horizon for Redis).

**Run the test suite:** `php artisan test` (or `vendor/bin/pest`). Parallel: `php artisan test --parallel`. Coverage: `--coverage --min=80`.
