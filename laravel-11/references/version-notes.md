# Laravel 11 Version-Specific Facts (Verified)

Every fact below was checked against the official Laravel 11.x documentation or the official
application skeleton in October 2026. Each entry cites its source. Use this file to settle
disputes about what is or isn't "Laravel 11 behavior" and to avoid writing Laravel 10/12 idioms.

## Support timeline

| Fact | Detail | Source |
| --- | --- | --- |
| Release date | March 12, 2024 | https://laravel.com/framework/docs/11.x/releases |
| PHP support | 8.2 – 8.4 (minimum 8.2) | https://laravel.com/framework/docs/11.x/releases |
| Bug fixes until | September 3, 2025 | https://laravel.com/framework/docs/11.x/releases |
| Security fixes until | March 12, 2026 | https://laravel.com/framework/docs/11.x/releases |
| Support policy | All versions: 18 months bug fixes, 2 years security fixes | https://laravel.com/framework/docs/11.x/releases |

As of late 2026, Laravel 11 is past end of life for bug fixes; security fixes ran out March 2026.
Existing 11 apps are still common, so the skill stays relevant — but when a user is starting a
brand-new app, suggest they consider a supported version.

## Streamlined application structure (the big one)

| Fact | Source |
| --- | --- |
| `bootstrap/app.php` is a code-first configuration file: `Application::configure()->withRouting()->withMiddleware()->withExceptions()->create()` | https://laravel.com/framework/docs/11.x/releases |
| New apps contain a single `AppServiceProvider` (was five providers); event discovery is on by default | https://laravel.com/framework/docs/11.x/releases |
| The nine former app-level middleware (auth, CSRF, trimming, etc.) moved into the framework; customization via `withMiddleware` in `bootstrap/app.php` | https://laravel.com/framework/docs/11.x/releases |
| No HTTP kernel needed (all middleware configuration is fluent) | https://laravel.com/framework/docs/11.x/middleware |
| Exception handling configured in `bootstrap/app.php` via `withExceptions` — no `app/Exceptions/Handler.php` | https://laravel.com/framework/docs/11.x/errors |
| Scheduling via `Schedule` facade in `routes/console.php` — no console Kernel | https://laravel.com/framework/docs/11.x/releases |
| Base `Controller` is an empty abstract class; `AuthorizesRequests`/`ValidatesRequests` traits removed from it | https://laravel.com/framework/docs/11.x/releases |
| `routes/api.php` and `routes/channels.php` not included by default; created by `php artisan install:api` / `install:broadcasting` | https://laravel.com/framework/docs/11.x/structure |
| Default route files are only `web.php` and `console.php` | https://laravel.com/framework/docs/11.x/structure |
| New apps use SQLite and the `database` driver for session, cache, and queue | https://laravel.com/framework/docs/11.x/releases |
| Default migrations consolidated to 3 files: `0001_01_01_000000_create_users_table.php`, `0001_01_01_000001_create_cache_table.php`, `0001_01_01_000002_create_jobs_table.php` | https://github.com/laravel/laravel/tree/11.x/database/migrations (verified via the 11.x branch) |
| `withCommands([...])` registers additional command directories in `bootstrap/app.php` | https://laravel.com/framework/docs/11.x/artisan |

## Fluent middleware API details

| Fact | Source |
| --- | --- |
| `$middleware->append()` / `prepend()` for global stack; `->use([...])` replaces it | https://laravel.com/framework/docs/11.x/middleware |
| `$middleware->web(append: [...])`, `api(prepend: [...])`, plus `replace:` and `remove:` arguments; `appendToGroup`/`prependToGroup`/`group` for arbitrary groups | https://laravel.com/framework/docs/11.x/middleware |
| `$middleware->alias([...])` defines middleware aliases; built-ins include `auth`, `can`, `throttle`, `signed`, `verified`, `password.confirm`, `auth.basic`, `auth.session`, `guest`, `precognitive`, `cache.headers`, `subscribed` | https://laravel.com/framework/docs/11.x/middleware |
| `$middleware->validateCsrfTokens(except: [...])` replaces the old `VerifyCsrfToken::$except` | https://laravel.com/framework/docs/11.x/releases |
| `$middleware->redirectGuestsTo(...)` replaces custom `Authenticate::redirectTo()` | https://laravel.com/framework/docs/11.x/authentication |
| `$middleware->statefulApi()` enables Sanctum SPA cookie auth on api routes | https://laravel.com/framework/docs/11.x/sanctum |
| `$middleware->throttleWithRedis()` maps `throttle` to the Redis middleware | https://laravel.com/framework/docs/11.x/routing |
| `$middleware->trustProxies(at: [...])` / `trustHosts(at: [...])` replace the TrustProxies/TrustHosts middleware files | https://laravel.com/framework/docs/11.x/requests |
| Default global stack: `InvokeDeferredCallbacks`, `TrustProxies`, `HandleCors`, `PreventRequestsDuringMaintenance`, `ValidatePostSize`, `TrimStrings`, `ConvertEmptyStringsToNull` | https://laravel.com/framework/docs/11.x/middleware |

## Other Laravel 11 features (new in this version)

| Fact | Source |
| --- | --- |
| Health routing: `health: '/up'` directive; dispatches `DiagnosingHealth` event | https://laravel.com/framework/docs/11.x/releases |
| Per-second rate limiting: `Limit::perSecond(1)` | https://laravel.com/framework/docs/11.x/releases |
| Graceful encryption key rotation via `APP_PREVIOUS_KEYS` env (comma-delimited; current `APP_KEY` encrypts, previous keys still decrypt) | https://laravel.com/framework/docs/11.x/releases |
| Automatic password rehashing on authentication (bcrypt work-factor upgrades) | https://laravel.com/framework/docs/11.x/releases |
| Model casts via `protected function casts(): array` method (e.g. `AsCollection::using(...)`) | https://laravel.com/framework/docs/11.x/releases |
| New commands: `make:class`, `make:enum`, `make:interface`, `make:trait` | https://laravel.com/framework/docs/11.x/releases |
| `once()` helper caches a callback result for the request lifetime | https://laravel.com/framework/docs/11.x/releases |
| Queue interaction testing: `$job->withFakeQueueInteractions()`, `assertReleased(delay:)`, `assertDeleted()`, `assertFailed()` | https://laravel.com/framework/docs/11.x/releases |
| In-memory SQLite testing is ~2x faster (PDO object reused across connections) | https://laravel.com/framework/docs/11.x/releases |
| Dedicated MariaDB driver with proper defaults | https://laravel.com/framework/docs/11.x/releases |
| Schema inspection: `Schema::getTables()`, `getViews()`, `getColumns()`, `getIndexes()`, `getForeignKeys()`; native column modify/rename/drop | https://laravel.com/framework/docs/11.x/releases |
| Laravel Prompts `validate:` accepts full Laravel validator rule arrays | https://laravel.com/framework/docs/11.x/releases |
| Laravel Reverb (first-party WebSocket server, `php artisan reverb:start`) ships with 11 | https://laravel.com/framework/docs/11.x/releases |
| `composer run dev` runs the dev server + queue worker + Vite together | https://laravel.com/framework/docs/11.x/installation |

## Job class shape (differs from Laravel 10 stubs)

The `make:job` stub in 11 uses a single `Illuminate\Foundation\Queue\Queueable` trait with
constructor property promotion, instead of 10's four traits (`Dispatchable`,
`InteractsWithQueue`, `Queueable`, `SerializesModels`).
Source: https://laravel.com/framework/docs/11.x/queues (class structure example).

## Sanctum installation path

`php artisan install:api` installs Sanctum AND creates `routes/api.php` with `/api` prefix.
Source: https://laravel.com/framework/docs/11.x/sanctum and
https://laravel.com/framework/docs/11.x/routing.

## Things that did NOT change in 11 (common misconceptions)

- Eloquent model conventions, relationships, and query builder — unchanged (source: 11.x/eloquent).
- Migration syntax (anonymous classes, `Blueprint`) — unchanged; only the default file set was
  consolidated (source: 11.x/migrations).
- Validation API (`validate()`, form requests, 422 JSON shape) — unchanged (source: 11.x/validation).
- Blade directives and component authoring — unchanged (source: 11.x/blade).
- PHPUnit/Pest testing APIs and `RefreshDatabase` — unchanged; only performance improved
  (source: 11.x/testing).
- `Auth::attempt()` and guards — unchanged except automatic rehashing now happens transparently
  (source: 11.x/authentication).
