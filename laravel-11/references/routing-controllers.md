# Laravel 11 Routing and Controllers

Source: https://laravel.com/framework/docs/11.x/routing.

## Route files and `withRouting`

Routes load through `bootstrap/app.php`. Default wiring:

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

- `web.php` routes get the `web` middleware group (session, CSRF, cookies).
- `api.php` doesn't exist until `php artisan install:api`; then it's auto-prefixed with `/api` and
  wrapped in the `api` group. Change the prefix with `apiPrefix: 'api/admin'`.
- Add extra route files with the `then:` closure, or take full control with `using:`:

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
    then: function () {
        Route::middleware('api')
            ->prefix('webhooks')
            ->name('webhooks.')
            ->group(base_path('routes/webhooks.php'));
    },
)
```

The `health: '/up'` directive registers a health-check endpoint for uptime monitors and
orchestrators (Kubernetes). Hitting it also dispatches a `DiagnosingHealth` event where you can
run app-specific checks. This is a Laravel 11 feature — don't hand-roll a `/health` route.

## Route basics

```php
use App\Http\Controllers\UserController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::get('/user', [UserController::class, 'index']);

Route::match(['get', 'post'], '/', fn () => /* ... */);
Route::any('/', fn () => /* ... */);

Route::redirect('/here', '/there', 301);
Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

Register verb-specific routes before `any`/`match`/`redirect` routes sharing the same URI.

### Parameters

```php
Route::get('/user/{id}', function (string $id) {
    return 'User '.$id;
});

Route::get('/user/{name?}', fn (?string $name = 'John') => $name);

// Constraints...
Route::get('/user/{id}', fn (string $id) => /* ... */)->whereNumber('id');
Route::get('/user/{name}', fn (string $name) => /* ... */)->whereAlpha('name');
Route::get('/category/{category}', fn (string $category) => /* ... */)->whereIn('category', ['movie', 'song']);
```

Global parameter patterns go in `AppServiceProvider::boot()`: `Route::pattern('id', '[0-9]+');`.

### Dependency injection

Type-hint dependencies first, route parameters after:

```php
Route::get('/user/{id}', function (Request $request, string $id) {
    return 'User '.$id;
});
```

## Route model binding

```php
use App\Models\User;
use App\Models\Post;

// Implicit binding: {user} matches the User type-hint; 404 if not found.
Route::get('/users/{user}', fn (User $user) => $user->email);

// Bind by a non-id column...
Route::get('/posts/{post:slug}', fn (Post $post) => $post);

// Scoped nested binding...
Route::get('/users/{user}/posts/{post:slug}', fn (User $user, Post $post) => $post);

// Force scoping without a custom key...
Route::get('/users/{user}/posts/{post}', fn (User $user, Post $post) => $post)->scopeBindings();

// Include soft-deleted models...
Route::get('/users/{user}', fn (User $user) => $user)->withTrashed();

// Custom "not found" behavior...
Route::get('/locations/{location:slug}', [LocationsController::class, 'show'])
    ->missing(fn (Request $request) => Redirect::route('locations.index'));
```

Enum binding (404s unless the segment is a valid backing value):

```php
enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}

Route::get('/categories/{category}', fn (Category $category) => $category->value);
```

To bind by something other than the model key globally, override `getRouteKeyName()` on the model
(`return 'slug';`) or use `Route::bind('user', fn (string $value) => User::where('name', $value)->firstOrFail())`
in `AppServiceProvider::boot()`.

## Route groups

```php
Route::middleware(['first', 'second'])->group(function () {
    Route::get('/', fn () => /* ... */);
});

Route::controller(OrderController::class)->group(function () {
    Route::get('/orders/{id}', 'show');
    Route::post('/orders', 'store');
});

Route::prefix('admin')->name('admin.')->group(function () {
    Route::get('/users', fn () => /* ... */)->name('users'); // name: admin.users
});

Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', fn (string $account, string $id) => /* ... */);
});
```

Register subdomain routes before root-domain routes so they aren't shadowed.

## Named routes

```php
Route::get('/user/profile', [UserProfileController::class, 'show'])->name('profile');

$url = route('profile', ['id' => 1]);
return redirect()->route('profile');
return to_route('profile');          // preferred helper for redirects
```

Route names must be unique; collisions silently break `route()` generation.

## Fallback routes

```php
Route::fallback(fn () => /* ... */);
```

Runs when nothing else matches — a good place for a styled 404 view. It gets the `web` group.

## Rate limiting

Define named limiters in `AppServiceProvider::boot()` (there is no `RouteServiceProvider` in 11):

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
}
```

Laravel 11 adds per-second granularity:

```php
RateLimiter::for('invoices', function (Request $request) {
    return Limit::perSecond(1);
});
```

Attach with the `throttle` middleware; exceeded requests get a 429 automatically:

```php
Route::middleware('throttle:api')->group(function () {
    Route::get('/invoices', fn () => /* ... */);
});
```

Multiple limits are evaluated in order; segment each with a unique `by()` value
(`Limit::perMinute(10)->by('minute:'.$user->id)` and `Limit::perDay(1000)->by('day:'.$user->id)`).

## Controllers

Standard resource controller workflow:

```shell
php artisan make:controller PostController --resource --model=Post
```

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class PostController extends Controller
{
    public function index(): View
    {
        return view('posts.index', ['posts' => Post::latest()->paginate(15)]);
    }

    public function show(Post $post): View
    {
        return view('posts.show', ['post' => $post]);
    }
}
```

Register the whole resource set in one line:

```php
Route::resource('posts', PostController::class);

// Only the routes you need...
Route::resource('posts', PostController::class)->only(['index', 'show']);

// API variant (no create/edit forms)...
Route::apiResource('posts', PostController::class);
```

The 11 base `Controller` is an empty abstract class — no traits. Bring `AuthorizesRequests` or
`ValidatesRequests` into a controller only when it uses them.

## Route caching

```shell
php artisan route:cache   # deploy time only
php artisan route:clear
```

Cached routes load much faster but new routes are ignored until a fresh cache — never cache in
local development. Closure-based routes ARE cacheable (unlike Laravel 7 and earlier).

## Inspecting routes

```shell
php artisan route:list            # overview
php artisan route:list -v         # + middleware
php artisan route:list -vv        # + expanded groups
php artisan route:list --path=api # only /api routes
php artisan route:list --except-vendor
```

Use `route:list -v` to verify middleware wiring after any middleware change — it shows exactly
what runs on each route.
