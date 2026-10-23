# Routing and Controllers (Laravel 12)

## Table of contents

- [Route files and bootstrap](#route-files-and-bootstrap)
- [Defining routes](#defining-routes)
- [Route parameters and constraints](#route-parameters-and-constraints)
- [Named routes](#named-routes)
- [Route groups](#route-groups)
- [Route model binding](#route-model-binding)
- [Rate limiting](#rate-limiting)
- [Controllers](#controllers)
- [API routes](#api-routes)
- [Route caching and inspection](#route-caching-and-inspection)

## Route files and bootstrap

Routes are loaded by `bootstrap/app.php`:

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',   // free health-check endpoint
        // api: __DIR__.'/../routes/api.php',        // after install:api
        // apiPrefix: 'api/admin',                   // customize the /api prefix
    )
    ->create();
```

The `health: '/up'` route is a zero-config uptime endpoint (used by load balancers and Laravel Cloud). For extra route files, use `then:` (register alongside defaults) or `using:` (take full manual control — the framework then registers no HTTP routes itself):

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    then: function (): void {
        Route::middleware('api')
            ->prefix('webhooks')
            ->name('webhooks.')
            ->group(base_path('routes/webhooks.php'));
    },
)
```

## Defining routes

```php
use App\Http\Controllers\UserController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::get('/user', [UserController::class, 'index']);
Route::post('/user', [UserController::class, 'store']);
Route::match(['get', 'post'], '/search', fn () => 'Search results');

Route::redirect('/here', '/there', 301);          // 302 default, 301 here
Route::permanentRedirect('/old', '/new');
Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

Ordering rule: when the same URI is registered multiple times, specific-verb routes (`get`/`post`/...) must be declared before `any`/`match`/`redirect` routes, or the greedy route wins.

Closure routes are fine for tiny endpoints, but any route destined for production with controller logic should live in a controller — closures break `route:cache` (see [Route caching](#route-caching-and-inspection)).

**Dependency injection** in handlers is automatic; route parameters come after injected dependencies:

```php
Route::get('/user/{id}', function (Request $request, string $id) {
    return 'User '.$id;
});
```

## Route parameters and constraints

```php
Route::get('/user/{name?}', function (?string $name = 'Guest') {
    return $name;
});

Route::get('/user/{id}/{name}', function (string $id, string $name) { /* ... */ })
    ->where(['id' => '[0-9]+', 'name' => '[a-z]+']);
```

Prefer the constraint helpers over raw regex — they communicate intent:

```php
Route::get('/user/{id}', fn (string $id) => $id)->whereNumber('id');
Route::get('/user/{name}', fn (string $name) => $name)->whereAlpha('name');
Route::get('/user/{slug}', fn (string $slug) => $slug)->whereAlphaNumeric('slug');
Route::get('/user/{id}', fn (string $id) => $id)->whereUuid('id');
Route::get('/user/{id}', fn (string $id) => $id)->whereUlid('id');
Route::get('/category/{category}', fn (string $category) => $category)
    ->whereIn('category', ['movie', 'song']);
```

Global patterns (one rule for every `{id}` in the app) go in `AppServiceProvider::boot()`:

```php
use Illuminate\Support\Facades\Route;

Route::pattern('id', '[0-9]+');
```

Non-matching requests automatically become **404s** — you rarely need to handle constraint failure yourself.

## Named routes

```php
Route::get('/user/profile', [ProfileController::class, 'show'])->name('profile');

$url = route('profile');                       // generation
return to_route('profile');                    // redirect helper
return redirect()->route('profile');
```

Name every route. Route names are the stable contract between backend and frontend (Blade `route()` calls, Inertia/Wayfinder references, test `assertRedirectToRoute`), and renaming a path without renaming the route never breaks consumers. In 12, when duplicate route names exist, the **first** registered route wins — keep names unique.

## Route groups

```php
Route::middleware(['auth', 'verified'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function (): void {
        Route::get('/users', [AdminUserController::class, 'index'])->name('users.index');
        Route::controller(ReportController::class)->group(function (): void {
            Route::get('/reports', 'index')->name('reports.index');
            Route::post('/reports', 'store')->name('reports.store');
        });
    });
```

Groups merge sensibly: middleware and `where` merge, names and prefixes concatenate. Subdomain routing via `Route::domain('{account}.example.com')` must be registered **before** root-domain routes or the root routes shadow them.

## Route model binding

The single biggest controller-slimming feature — let Laravel fetch (or 404 on) the model:

```php
use App\Models\User;

Route::get('/users/{user}', function (User $user) {
    return $user->email;
});

// Custom key column:
Route::get('/posts/{post:slug}', [PostController::class, 'show']);

// Soft-deleted models included:
Route::get('/users/{user}', fn (User $user) => $user->email)->withTrashed();

// Scoped children (post must belong to user):
Route::get('/users/{user}/posts/{post:slug}', [UserPostController::class, 'show'])
    ->scopeBindings();

Route::scopeBindings()->group(function (): void { /* all bindings scoped */ });

// Custom missing-model behavior instead of 404:
Route::get('/locations/{location:slug}', [LocationController::class, 'show'])
    ->name('locations.view')
    ->missing(function (Request $request) {
        return Redirect::route('locations.index');
    });
```

**Enum binding** — a `string`-backed enum type-hint 404s automatically on invalid values:

```php
enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

Explicit binding (`Route::model('user', User::class)`) and custom resolution (`Route::bind('user', fn (string $value) => User::where('name', $value)->firstOrFail())`) go in `AppServiceProvider::boot()`. Models can also override `getRouteKeyName(): string` (return `'slug'` to bind by slug everywhere) or `resolveRouteBinding()`.

## Rate limiting

Define named limiters in `AppServiceProvider::boot()`:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });

    RateLimiter::for('uploads', function (Request $request) {
        return [
            Limit::perMinute(10)->by('minute:'.$request->user()->id),
            Limit::perDay(1000)->by('day:'.$request->user()->id),
        ];
    });
}
```

Attach with the `throttle` middleware: `Route::middleware('throttle:uploads')->group(...)`.

Two 12-era refinements worth using:

- **Segment keys must be unique** across multiple limits — prefix them (`'minute:'.$id`) as above, or the limits collide.
- **Response-based counting** — only count requests whose response matches a condition, e.g. throttle *failed* lookups to blunt enumeration attacks:

```php
use Symfony\Component\HttpFoundation\Response;

RateLimiter::for('resource-not-found', function (Request $request) {
    return Limit::perMinute(10)
        ->by($request->user()?->id ?: $request->ip())
        ->after(function (Response $response): bool {
            return $response->status() === 404;
        });
});
```

A custom 429 payload comes from `Limit::response(fn (Request $request, array $headers) => response('Slow down', 429, $headers))`.

## Controllers

```shell
php artisan make:controller PostController            # plain
php artisan make:controller PostController --resource # with index/create/store/show/edit/update/destroy
php artisan make:controller ShowDashboard --invokable # single __invoke() action
```

Resource controllers map to the 7 RESTful actions; register with `Route::resource('posts', PostController::class)` (or `--api` for the 5 stateless actions; `Route::apiResource(...)`). Scoping nested resources: `Route::resource('users.posts', PostController::class)->scoped(['post' => 'slug'])`.

Idiomatic controller shape — thin, typed, form-request validated, route-named:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class PostController extends Controller
{
    public function index(): View
    {
        return view('posts.index', [
            'posts' => Post::latest()->with('author')->paginate(15),
        ]);
    }

    public function store(StorePostRequest $request): RedirectResponse
    {
        $post = $request->user()->posts()->create($request->validated());

        return to_route('posts.show', $post);
    }

    public function update(UpdatePostRequest $request, Post $post): RedirectResponse
    {
        $post->update($request->validated());

        return to_route('posts.show', $post);
    }
}
```

Authorization lives in policies (`$this->authorize('update', $post)` or the `can` middleware / `Gate` facade), not in controllers' first lines of if-statements. Validation lives in form requests (`requests-validation.md`). Controllers orchestrate; they don't validate, authorize, or compute.

## API routes

After `php artisan install:api` (installs Sanctum, creates `routes/api.php`):

- Routes get the `api` middleware group automatically and the `/api` URI prefix.
- Protect with `->middleware('auth:sanctum')` for token auth; add `$middleware->statefulApi()` in `bootstrap/app.php` when a first-party SPA authenticates via cookies.
- JSON 404s and validation errors (422) are automatic for routes under `api` when the client sends `Accept: application/json` — or customize globally via `shouldRenderJsonWhen` (see `error-logging.md`).

## Route caching and inspection

```shell
php artisan route:list                     # all routes
php artisan route:list -v                  # + middleware
php artisan route:list --path=api          # filter by URI
php artisan route:list --except-vendor     # hide package routes
php artisan route:cache                    # production: single-file route registration
php artisan route:clear
```

`route:cache` is a real production win but demands: **controller routes only** (closures can't be cached), and awareness that stale caches hide route edits — re-run `route:cache` after deploying route changes (most deploy scripts do this via `php artisan optimize`).
