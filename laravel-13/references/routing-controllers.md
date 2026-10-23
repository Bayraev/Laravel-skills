# Routing and Controllers

## Route files and basics

- `routes/web.php` — stateful routes (`web` group: sessions, cookies, forgery protection).
- `routes/api.php` (after `php artisan install:api`) — stateless, auto-prefixed `/api`, Sanctum.
- Inspect: `php artisan route:list` (add `--except-vendor` to hide package routes; `--path=photos` to filter).

```php
use App\Http\Controllers\PhotoController;

Route::get('/greeting', fn () => 'Hello World');
Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
Route::redirect('/here', '/there', 301);

Route::get('/users/{user}/posts/{post:slug}', [PostController::class, 'show']);  // scoped binding
```

Route parameters: required `{id}`, optional `{id?}`, constrained with `->whereNumber('id')`, `->whereAlpha`, `->whereUuid`, `->whereIn('type', ['a','b'])`, or `->where('id', '[0-9]+')`.

## Route model binding

```php
use App\Models\Post;

Route::get('/posts/{post}', function (Post $post) {
    return $post->toResource();
});

// Custom key: /posts/{post:slug}
// Custom not-found behavior:
Route::get('/posts/{post}', [PostController::class, 'show'])->missing(function (Request $request) {
    return Redirect::route('posts.index');
});
```

Nested bindings are auto-scoped when the child is a custom-keyed parameter: `/users/{user}/posts/{post:slug}` verifies the post belongs to the user. Enums bind automatically: `function (Status $status)` gets a 404 for invalid values.

## Route groups

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::controller(PhotoController::class)->group(function () {
        Route::get('/photos', 'index');
    });
    Route::prefix('admin')->name('admin.')->group(function () {
        Route::get('/stats', fn () => /* ... */)->name('stats');  // route('admin.stats')
    });
});
```

## Controllers

```bash
php artisan make:controller PhotoController                        # plain
php artisan make:controller PhotoController --resource            # 7 RESTful actions
php artisan make:controller PhotoController --model=Photo --resource --requests
php artisan make:controller PhotoController --api                  # API resource (no create/edit)
php artisan make:controller ProvisionServer --invokable            # single __invoke action
```

Keep controllers thin: validate (form request), delegate (model/service), return (view/redirect/resource). Controllers are not required to extend `Controller`, but do so when sharing helpers.

### Controller middleware — the 13 attribute style

```php
<?php

namespace App\Http\Controllers;

use App\Models\Comment;
use App\Models\Post;
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Illuminate\Routing\Attributes\Controllers\Middleware;

#[Middleware('auth')]
class CommentController
{
    #[Middleware('subscribed')]
    #[Authorize('create', [Comment::class, 'post'])]   // == can middleware; second arg: class, route param, or args
    public function store(Post $post)
    {
        // ...
    }

    #[Authorize('delete', 'comment')]                  // 'comment' is the route-model-bound param
    public function destroy(Comment $comment)
    {
        // ...
    }
}
```

Alternatives (all still valid): the `HasMiddleware` interface with a static `middleware()` method, route-level `->middleware('auth')`, and closures via `#[Middleware(static fn (Request $r, Closure $next) => $next($r))]`. Use `#[WithoutMiddleware('subscribed', except: ['index'])]` to exclude; class-level `WithoutMiddleware` is inherited by child classes.

**Why attributes:** middleware and authorization land next to the code they protect — no jumping between route files and controllers to trace behavior. Policies themselves live in `app/Policies` (`php artisan make:policy PhotoPolicy --model=Photo`).

## Resource controllers

`Route::resource('photos', PhotoController::class);` registers:

| Verb | URI | Action | Route name |
|---|---|---|---|
| GET | `/photos` | index | photos.index |
| GET | `/photos/create` | create | photos.create |
| POST | `/photos` | store | photos.store |
| GET | `/photos/{photo}` | show | photos.show |
| GET | `/photos/{photo}/edit` | edit | photos.edit |
| PUT/PATCH | `/photos/{photo}` | update | photos.update |
| DELETE | `/photos/{photo}` | destroy | photos.destroy |

```php
Route::resource('photos', PhotoController::class)->only(['index', 'show']);
Route::apiResource('photos', PhotoController::class);               // excludes create/edit
Route::resource('photos.comments', CommentController::class)->shallow();
Route::resource('photos.comments', CommentController::class)->scoped(['comment' => 'slug']);
Route::resource('photos', PhotoController::class)->withTrashed(['show']);  // soft-deleted models
Route::softDeletableResources(['photos' => PhotoController::class]);

// Per-action middleware (fine-grained):
Route::resource('users', UserController::class)
    ->middleware(['auth', 'verified'])
    ->middlewareFor(['show', 'update'], 'auth')
    ->withoutMiddlewareFor('destroy', 'subscribed');

// Define supplemental routes BEFORE the resource so they aren't shadowed:
Route::get('/photos/popular', [PhotoController::class, 'popular']);
Route::resource('photos', PhotoController::class);
```

Singleton resources (one instance, e.g. a profile): `Route::singleton('profile', ProfileController::class)` (show/edit/update only), `->creatable()`, `->destroyable()`, `Route::apiSingleton(...)`.

## Rate limiting

Define in `AppServiceProvider::boot()`:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

Attach: `->middleware('throttle:api')` (or `throttle:60,1`). Response includes `Retry-After` headers; exceed returns 429. Rate limiters can return multiple limits and `Limit::none()` for exempt users.

## Caching

In production run `php artisan route:cache` (or `php artisan optimize`); closure-based routes/controllers **cannot** be cached — always use controller classes in production apps. Clear with `optimize:clear`.
