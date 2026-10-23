# Structure and Conventions (Laravel 10 Classic Skeleton)

Laravel 10 keeps the full classic skeleton. Knowing where things live matters because Laravel 11+ deleted or relocated several of these files — if you find advice online referencing `bootstrap/app.php` config, it does not apply here. Source: [structure](https://laravel.com/framework/docs/10.x/structure), [providers](https://laravel.com/framework/docs/10.x/providers).

## Root directory

| Path | Purpose |
| --- | --- |
| `app/` | Your code, `App\` namespace (PSR-4) |
| `bootstrap/` | `app.php` framework bootstrap + `cache/` (route/config/services caches — safe to delete, they rebuild) |
| `config/` | All config files; read values with `config('key')`, write environment-specific values via `.env` |
| `database/` | `migrations/`, `factories/`, `seeders/`; may hold `database.sqlite` |
| `public/` | `index.php` front controller + built assets. Web server docroot must point here, never at project root |
| `resources/` | `views/` (Blade), `css/`, `js/` (Vite sources) |
| `routes/` | `web.php`, `api.php`, `console.php`, `channels.php` |
| `storage/` | `app/` (user files; `app/public` exposed via `php artisan storage:link`), `framework/`, `logs/` |
| `tests/` | `Feature/` + `Unit/`, run via `php artisan test` |

## The `app/` directory

Present from the start: `Http/` (Controllers, Middleware, Requests), `Models/`, `Providers/`, `Console/`, `Exceptions/`.

Directories like `Jobs/`, `Events/`, `Listeners/`, `Mail/`, `Notifications/`, `Policies/`, `Rules/`, `Broadcasting/` **do not exist until the first `make:` command creates them** — don't be surprised by their absence, and don't create them by hand; run the generator instead.

Two directories deserve special attention:

### `app/Http/Kernel.php` — the Laravel 10 middleware hub

```php
namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    protected $middleware = [
        // \App\Http\Middleware\TrustHosts::class,
        \App\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ];

    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],

        'api' => [
            \Illuminate\Routing\Middleware\ThrottleRequests::class.':api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
    ];

    protected $middlewareAliases = [
        'auth' => \App\Http\Middleware\Authenticate::class,
        // 'auth.basic' => ..., 'guest' => ..., 'throttle' => ..., etc.
    ];
}
```

- Global stack: `$middleware`. Groups: `$middlewareGroups`. Aliases used on routes: `$middlewareAliases` (renamed from `$routeMiddleware` in 10.0). Priority override (rare): `$middlewarePriority`.
- When a task says "register a middleware", this file (plus `make:middleware` to generate the class) is the answer on 10.x — never `bootstrap/app.php`.

### `app/Providers/` — bootstrapping logic

| Provider | Laravel 10 role |
| --- | --- |
| `AppServiceProvider` | Default home for container bindings and app-level boot logic (`boot(): void`) — e.g. `Model::preventLazyLoading(! $this->app->isProduction())` |
| `AuthServiceProvider` | `Gate::define()` calls in `boot()`; `$policies` array (auto-discovery makes most entries unnecessary); `registerPolicies()` is called automatically in 10 |
| `RouteServiceProvider` | Loads route files into `web`/`api` groups, sets the `/api` prefix and controller namespace, defines the default `api` rate limiter in `boot()` |
| `EventServiceProvider` | `$listen` array + `shouldDiscoverEvents()` for auto-discovery |
| `BroadcastServiceProvider` | Channel auth routes (loaded only if broadcasting enabled) |

Providers are registered in `bootstrap/providers.php` in 10.x — a plain PHP array return. Adding a provider = create class + append to that array (or `php artisan make:provider` + edit).

Gates and policies details: `references/middleware-auth.md`. Route loading details: `references/routing-controllers.md`.

## Configuration access rules

- **`.env` → `config/*.php` → `config()` helper.** `env()` may only appear inside `config/` files: production caches config, and outside `config/` a cached app returns `null` for every `env()` call. This is the single most common production-only bug in Laravel.
- `.env` never enters version control; `.env.example` documents the keys.
- Per-environment overrides: `.env.testing` wins during tests; `.env.dusk.local` during Dusk.
- Cache after config/route changes in production: `php artisan config:cache`, `route:cache`. During development, leave caches off — stale caches make route/model edits appear broken.

## Naming and style (matching the 10.x typed skeleton)

Laravel 10 stubs gave every generated method native types. Match them in hand-written code:

```php
public function boot(): void {}
public function handle(Request $request, Closure $next): Response {}
public function rules(): array {}
public function authorize(): bool {}
public function handle(): void {}          // job handle (void unless you return something)
public function up(): void {}              // ...and down(): void on migrations
```

Use PHP 8.1 features freely — constructor promotion, `readonly` properties, enums, `match`, arrow functions, first-class callable syntax — but nothing from 8.2+ (`readonly` classes, DNF types, `true`/`null` standalone types), since Laravel 10 supports PHP 8.1. Format with Laravel Pint (`vendor/bin/pint`) when present in `composer.json`.

## Placing new code — the decision list

When adding a class, find its home in this order:

1. **Does a generator exist?** (`make:model`, `make:job`, `make:rule`...) — use it; the class lands in the right directory with the right types.
2. **No generator?** Follow the directory's existing convention: service/domain classes can live in `app/Services/`, `app/Actions/`, `app/Domain/`... whichever the project already established. Laravel 10 imposes no restriction beyond Composer autoloading.
3. **Composer check**: any class in `app/` autoloads via PSR-4 (`App\` → `app/`). New top-level `app/` directories just work; classes outside `app/` need `composer.json` autoload updates + `composer dump-autoload`.

Avoid these common placement mistakes: HTTP-layer classes outside `app/Http` (jobs handling requests belong in controllers), business logic in `routes/` closures (grows untestable), and `App\Console\Commands` classes invoked only by other code (extract the work into an invokable service the command calls — commands are a shell, not a library).

## Service container essentials

Constructor injection is automatic for controllers, jobs, listeners, middleware, form request `rules()`, and anything resolved via `app()`:

```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Services\PhotoService;

class PhotoController extends Controller
{
    public function __construct(
        protected readonly PhotoService $photos,   // PHP 8.1 promotion + readonly
    ) {}

    // $this->photos is available in every action
}
```

Bind interfaces to implementations in `AppServiceProvider::register()`:

```php
use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;

$this->app->bind(PaymentGateway::class, StripeGateway::class);          // new instance each resolve
$this->app->singleton(PaymentGateway::class, StripeGateway::class);     // shared instance
$this->app->scoped(PaymentGateway::class, StripeGateway::class);        // singleton per request/job
```

Contextual binding when a specific consumer needs a specific implementation:

```php
$this->app->when(ReportGenerator::class)
    ->needs(PaymentGateway::class)
    ->give(NullGateway::class);
```

Facades vs injection: both are documented styles. Facades (`Cache::get(...)`) match the docs' examples; constructor injection makes dependencies explicit and mocks trivial (`$this->mock(PhotoService::class)`). Follow the surrounding file's convention.
