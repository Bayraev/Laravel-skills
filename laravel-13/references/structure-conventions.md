# Structure and Conventions

Where code lives in a Laravel 13 application. Laravel's predictable structure is a feature — it's why AI agents and new teammates can navigate the codebase without a tour. Do not invent alternative locations; Composer PSR-4 autoloads `app/` as `App\`, and everything below assumes that.

## Root directory

```
example-app/
├── app/                 # Your application code (only Http, Models, Providers exist initially)
├── bootstrap/           # app.php (framework bootstrap + config hub), cache/ (framework caches)
├── config/              # Configuration files (each documented)
├── database/            # migrations/, factories/, seeders/, database.sqlite (default)
├── public/              # index.php front controller, built assets — the only web-exposed dir
├── resources/           # views/, css/, js/ (pre-compiled sources)
├── routes/              # web.php, console.php (+ api.php, channels.php when installed)
├── storage/             # logs/, compiled views, sessions, file cache; app/public for user files
├── tests/               # Feature/, Unit/
└── vendor/              # Composer dependencies (never edit)
```

Key points:

- **Serve from the web root only** (`public/`). Never serve the project root — it would expose `.env` and source.
- `storage/app/public` holds user-generated public files; expose via `php artisan storage:link` (creates `public/storage` symlink).
- `bootstrap/app.php` is the application configuration hub (middleware, exceptions, scheduling, commands) — see `middleware-config.md`.
- `bootstrap/cache` holds framework caches (routes, config, services). Wipe with `php artisan optimize:clear`.

## The app/ directory

A fresh app contains only `app/Http`, `app/Models`, and `app/Providers`. Other directories are created on demand by `make:` commands — don't create them speculatively:

| Directory | Created by | Contains |
|---|---|---|
| `app/Http/Controllers` | `make:controller` | Controllers |
| `app/Http/Middleware` | `make:middleware` | Custom middleware |
| `app/Http/Requests` | `make:request` | Form requests (validation + authorization) |
| `app/Http/Resources` | `make:resource` | API resource classes (JSON + JSON:API) |
| `app/Models` | `make:model` | Eloquent models |
| `app/Providers` | (initial) | Service providers; `AppServiceProvider` exists from the start |
| `app/Console/Commands` | `make:command` | Artisan commands |
| `app/Jobs` | `make:job` | Queueable jobs |
| `app/Events` | `make:event` / `event:generate` | Event classes |
| `app/Listeners` | `make:listener` / `event:generate` | Event listeners |
| `app/Notifications` | `make:notification` | Transactional notifications |
| `app/Mail` | `make:mail` | Mailable classes |
| `app/Policies` | `make:policy` | Authorization policies |
| `app/Rules` | `make:rule` | Custom validation rule objects |
| `app/Broadcasting` | `make:channel` | Broadcast channel classes |
| `app/Exceptions` | `make:exception` | Custom exceptions |
| `app/Ai/Agents`, `app/Ai/Tools` | `make:agent`, `make:tool` | Laravel AI SDK agent and tool classes |

Discover all generators: `php artisan list make`.

**Thinking model:** HTTP (`app/Http`) and CLI (`app/Console`) are *delivery mechanisms*, not where business logic lives. Keep controllers thin; push domain logic into models, services, or actions. For growing apps, domain service classes may live in `app/Services` or `app/Actions` (convention, not framework-enforced) — group by domain, not by type, once the app is large.

## Route files

| File | Applied middleware | Purpose |
|---|---|---|
| `routes/web.php` | `web` group (sessions, cookies, PreventRequestForgery) | Stateful web UI routes |
| `routes/api.php` (via `install:api`) | `api` group (stateless; prefix `/api`) | API routes, Sanctum token auth |
| `routes/console.php` | none | Closure-based Artisan commands **and scheduled tasks** |
| `routes/channels.php` (via `install:broadcasting`) | — | Event broadcast channel authorization |

If a route needs session state or CSRF protection, it belongs in `web.php`. RESTful APIs consumed by an SPA/mobile app go in `api.php` with Sanctum.

## Service providers

- `app/Providers/AppServiceProvider.php` is present from the start. Register bindings in `register()`, framework boot-time configuration (rate limiters, `Queue::route`, `JsonResource::withoutWrapping()`, `Model::unguard()`) in `boot()`.
- Additional providers (`make:provider`) must be registered in `bootstrap/providers.php` (the 11+ registration file — not `config/app.php`).
- Default first-party providers are registered by the framework itself; you only manage your own.

## Configuration

- All configuration lives in `config/*.php`; values come from `.env` (never commit `.env`; commit `.env.example`).
- Environment-specific overrides: `.env.testing` is used automatically under Pest/PHPUnit and with `--env=testing`.
- Access config at runtime: `config('app.name')`, `env()` only inside `config/` files (config caching turns `env()` calls elsewhere into `null`).
- Cache for production: `php artisan optimize` (config, routes, views, events). Clear: `php artisan optimize:clear`. **Never run `config:cache` in local dev with a testing workflow without clearing — stale cached config silently breaks tests.**

## New-project essentials

```bash
laravel new example-app          # interactive (starter kit, DB, testing framework choices)
cd example-app
npm install && npm run build     # build frontend assets once
composer run dev                 # server + queue worker + Vite, http://localhost:8000
```

- SQLite is pre-configured and migrated. Switch to MySQL/Postgres by editing `DB_*` in `.env`, then `php artisan migrate`.
- `php artisan install:api` adds `routes/api.php` + Sanctum. `php artisan install:broadcasting` adds Reverb/channels scaffolding.
- Starter kits (React, Svelte, Vue, Livewire) provide auth scaffolding; select during `laravel new` or use a community kit with `laravel new my-app --using=vendor/starter-kit`.

## Naming conventions (follow exactly — tools and agents rely on them)

- Controllers: `PhotosController` or `PhotoController` — pick one, stay consistent; suffix `Controller`.
- Models: singular `StudlyCase` (`Flight`); tables are snake_case plural (`flights`) — Eloquent infers this.
- Form requests: `StorePostRequest` / `UpdatePostRequest` (verb + resource + `Request`).
- Resources: `PostResource`, collections `PostCollection` (the `toResource()` convention discovers these names).
- Migrations: snake_case past-tense actions — `create_flights_table`, `add_status_to_flights_table` (the name drives stub pre-filling).
- Jobs: imperative verbs — `ProcessPodcast`.
- Tests: `*Test` suffix, `tests/Feature` by default.
- Route names: `photos.index`, `photos.show` (resource convention) — reference via `route('photos.show', $photo)`.
