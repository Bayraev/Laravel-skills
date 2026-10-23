# Laravel 13 Version Notes — Verified Facts and Sources

All facts below were verified against the official Laravel 13.x documentation in October 2026. Primary URL pattern: `https://laravel.com/framework/docs/13.x/{topic}`; raw markdown: `https://raw.githubusercontent.com/laravel/docs/13.x/{topic}.md`.

## Release and support

- Laravel 13 released **March 17, 2026** (annual major release cadence, ~Q1 each year; semver; use `^13.0`).
- Bug fixes until **Q3 2027**; security fixes until **March 17, 2028**.
- Laravel 12 (Feb 24, 2025): bug fixes until Aug 13, 2026; security until Feb 24, 2027.
- Named arguments are not covered by Laravel's backwards-compatibility promise — parameter names may change between releases.
- The 13 release deliberately minimized breaking changes: "most Laravel applications may upgrade to Laravel 13 without changing much application code."
- Source: https://laravel.com/framework/docs/13.x/releases

## PHP and platform requirements

- **PHP 8.3 – 8.5** (minimum 8.3). The installation docs' `php.new` one-liners install PHP 8.5 (e.g. `https://php.new/install/linux/8.5`).
- Databases: MariaDB 10.3+, MySQL 5.7+, PostgreSQL 10.0+, SQLite 3.26.0+, SQL Server 2017+. Vector columns additionally need PostgreSQL `pgvector` or MariaDB 11.7+.
- Sources: https://laravel.com/framework/docs/13.x/releases, https://laravel.com/framework/docs/13.x/database

## New in Laravel 13 (from the release notes page)

### Laravel AI SDK (first-party)
- Package `laravel/ai` — unified API for text generation, tool-calling agents, embeddings, audio, images, vector stores.
- `php artisan make:agent`, `make:tool`; `Agent::make()->prompt(...)`; `Str::of('...')->toEmbeddings()`; `Image::of('...')->generate()`; `Audio::of('...')->generate()`.
- Docs: https://laravel.com/framework/docs/13.x/ai-sdk — see `ai-tooling.md` in this folder.

### JSON:API resources
- `Illuminate\Http\Resources\JsonApi\JsonApiResource`; generate with `php artisan make:resource PostResource --json-api`.
- Handles resource object serialization, relationship inclusion (`?include=author,comments`), sparse fieldsets, links, and sets `Content-Type: application/vnd.api+json`.
- Docs: https://laravel.com/framework/docs/13.x/eloquent-resources#jsonapi-resources

### Request forgery protection (PreventRequestForgery)
- `Illuminate\Foundation\Http\Middleware\PreventRequestForgery`, in the default `web` middleware group.
- Two layers: (1) `Sec-Fetch-Site` header origin verification (same-origin requests pass with no token check); (2) classic session-token CSRF validation fallback. HTTPS required for the origin check (browsers only send the header over secure connections).
- Configure in `bootstrap/app.php`: `$middleware->preventRequestForgery(originOnly: true)` (403 on failure instead of 419), `allowSameSite: true` (subdomain support), `except: ['stripe/*']`.
- Docs: https://laravel.com/framework/docs/13.x/csrf

### Queue routing
- `Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts')` — also accepts interfaces, traits, parent classes, or an array map. Call from a service provider `boot()`.
- `Queue::forward('reports', 'reports.fifo', 'sqs')` — move a queue's jobs to a different queue/connection without touching dispatch sites.
- Docs: https://laravel.com/framework/docs/13.x/queues#queue-routing

### Expanded PHP attributes
- Controllers: `#[Middleware('auth')]`, `#[Middleware('log', only: ['index'])]`, `#[WithoutMiddleware(...)]`, `#[Authorize('create', [Comment::class, 'post'])]` (`Illuminate\Routing\Attributes\Controllers\*`).
- Jobs: `#[Tries]`, `#[Backoff]`, `#[Timeout]`, `#[FailOnTimeout]`, `#[MaxExceptions]`, `#[UniqueFor(3600)]`, `#[DebounceFor(30, maxWait: 120)]`, `#[WithoutRelations]` (`Illuminate\Queue\Attributes\*`).
- Eloquent/models: `#[Table(...)]`, `#[WithoutIncrementing]`, `#[WithoutTimestamps]`, `#[DateFormat('U')]`, `#[UseResource(...)]`, `#[UseResourceCollection(...)]`, `#[PreserveKeys]`, `#[Collects(...)]`.
- Form requests: `#[StopOnFirstFailure]`, `#[FailOnUnknownFields]` (+ global `FormRequest::failOnUnknownFields()`), `#[RedirectTo('/dashboard')]`, `#[RedirectToRoute('dashboard')]`, `#[ErrorBag('login')]`.
- Tests: `#[UnitTest]` skips booting the app for one method; `Illuminate\Foundation\Testing\WithCachedConfig` trait speeds suites.
- Sources: controllers, queues, eloquent, eloquent-resources, validation, testing doc pages.

### Cache TTL extension
- `Cache::touch('key', 3600)` or `Cache::touch('key', now()->addHours(2))` — extend TTL without retrieving/re-storing the value.
- Docs: https://laravel.com/framework/docs/13.x/cache

### Semantic / vector search
- Migration: `$table->vector('embedding', dimensions: 1536)` (->index() creates a cosine-distance vector index); `Schema::ensureVectorExtensionExists()` for pgvector.
- Query: `whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)`, plus `selectVectorDistance`, `whereVectorDistanceLessThan`, `orderByVectorDistance`. A plain string is auto-embedded via the AI SDK.
- Docs: https://laravel.com/framework/docs/13.x/queries#vector-similarity-clauses, https://laravel.com/framework/docs/13.x/search

### Model→resource convention
- `$model->toResource()`, `$collection->toResourceCollection()`, `$paginator->toResourceCollection()` — discover resources by name convention (`User` → `UserResource`, `UserCollection`), overridable with `#[UseResource]` / `#[UseResourceCollection]` or an explicit argument.
- Docs: https://laravel.com/framework/docs/13.x/eloquent-resources

## Other notable 13-era defaults (verified on doc pages)

- Job classes scaffold with the single `Illuminate\Foundation\Queue\Queueable` trait (replaces the old `Dispatchable, InteractsWithQueue, Queueable, SerializesModels` use-block).
- Fresh app ships `routes/web.php` + `routes/console.php` only; `api.php` via `php artisan install:api`; `channels.php` via `php artisan install:broadcasting`.
- Default `web` middleware group: EncryptCookies, AddQueuedCookiesToResponse, StartSession, ShareErrorsFromSession, **PreventRequestForgery**, SubstituteBindings. Default `api` group: SubstituteBindings (Sanctum's stateful middleware commented in when installed).
- `composer run dev` runs the dev server, queue worker, and Vite concurrently; app served at http://localhost:8000.
- Starter kits: React, Svelte, Vue, Livewire (layout/sidebar variants, teams, two-factor, WorkOS AuthKit option); community kits via `laravel new my-app --using=example/starter-kit`.
- IDE support: official **Laravel LSP** (`composer global require laravel/lsp`) with official extensions for VS Code/Cursor, Sublime, Zed; Neovim/OpenCode instructions in the LSP repo; PhpStorm has built-in Laravel support.
- Laravel Boost (`composer require laravel/boost --dev`, `php artisan boost:install`) supports Laravel 10–13 on PHP 8.1+.
- Search stack: `whereFullText` (MariaDB/MySQL/PostgreSQL) for keyword relevance; `whereVectorSimilarTo` for semantic; Scout for `Searchable` model indexing (database engine built in; Algolia, Meilisearch, Typesense, Turbopuffer drivers); AI SDK reranking as a second stage.

## Things that did NOT change (common misconceptions)

- `bootstrap/app.php` fluent configuration (`->withMiddleware`, `->withExceptions`, `->withSchedule`, `->withCommands`) is still the configuration hub — introduced in 11, unchanged in 13.
- Eloquent relationship methods, casts, scopes, and the query builder API are stable from 11/12; only model-convention *declaration style* shifted to attributes (old properties still work).
- Pest and PHPUnit both ship out of the box; `php artisan test` wraps either.
- Sanctum remains the API-token / SPA-auth package; `install:api` wires it up.
