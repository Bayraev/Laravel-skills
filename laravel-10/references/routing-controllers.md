# Routing and Controllers

Source: [routing](https://laravel.com/framework/docs/10.x/routing), [controllers](https://laravel.com/framework/docs/10.x/controllers). All patterns below are Laravel 10; the key 10-specific point is that `routes/web.php` and `routes/api.php` are loaded by `App\Providers\RouteServiceProvider`, which applies the `web`/`api` middleware groups and the automatic `/api` prefix — none of this is configured in `bootstrap/app.php`.

## Route files

| File | Loaded into | Behavior |
| --- | --- | --- |
| `routes/web.php` | `web` group | Session state, CSRF protection, cookie encryption |
| `routes/api.php` | `api` group | Stateless, throttle:`api`, auto-prefixed with `/api` |
| `routes/console.php` | — | Closure-based Artisan commands |
| `routes/channels.php` | — | Broadcasting channel registrations |

## Writing routes

```php
use App\Http\Controllers\PhotoController;
use Illuminate\Support\Facades\Route;

// Closure route
Route::get('/greeting', function () {
    return 'Hello World';
});

// Controller route — array syntax (no string Controller@method syntax)
Route::get('/photos/{photo}', [PhotoController::class, 'show']);

// Route naming for stable URL generation
Route::get('/photos', [PhotoController::class, 'index'])->name('photos.index');

// Group middleware, prefixes, and names together
Route::middleware(['auth', 'verified'])->prefix('admin')->name('admin.')->group(function () {
    Route::get('/users', [AdminUserController::class, 'index'])->name('users.index');
});
```

- Verbs: `Route::get/post/put/patch/delete/options`, plus `Route::match(['get', 'post'], ...)` and `Route::any(...)`.
- **Redirects**: `Route::redirect('/here', '/there', 301)`. **Views**: `Route::view('/welcome', 'welcome', ['name' => 'Taylor'])`.
- Nested groups merge intelligently: middleware and `where` conditions merge; names and prefixes concatenate.

### Route parameters

```php
Route::get('/user/{id}', function (string $id) {
    return 'User '.$id;
});

// Optional parameter — give the closure a default
Route::get('/user/{name?}', function (string $name = 'Guest') {
    return $name;
});

// Constrain with where() (patterns: 'number' → [0-9]+, 'alpha', 'alnum', 'uuid', 'ulid')
Route::get('/user/{id}', function (string $id) { /* ... */ })
    ->whereNumber('id');

Route::get('/user/{id}/{name}', fn (int $id, string $name) => /* ... */)
    ->where(['id' => '[0-9]+', 'name' => '[a-z]+']);

// Global patterns live in RouteServiceProvider::boot()
Route::pattern('id', '[0-9]+');
```

Laravel 10 skeletons type-hint route closure parameters (`string $id`, `int $id`) — do the same.

### Route model binding

```php
use App\Models\User;

// Implicit binding: {user} resolves to User by primary key, 404 if missing
Route::get('/users/{user}', [UserController::class, 'show']);

// Custom key: /users/taylor → User where slug = 'taylor'
Route::get('/users/{user:slug}', [UserController::class, 'show']);

// Relationship-scoped: /users/{user}/posts/{post:slug}
Route::get('/users/{user}/posts/{post:slug}', [PostController::class, 'show']);

// Soft-deleted models included
Route::get('/users/{user}', [UserController::class, 'show'])->withTrashed();
```

Explicit bindings and custom resolution closures (`Route::bind()`) are defined in `RouteServiceProvider::boot()` — a 10.x-specific location.

### Rate limiting

Rate limiters live in `RouteServiceProvider::boot()` (the default `api` limiter is defined there):

```php
namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class RouteServiceProvider extends ServiceProvider
{
    protected function configureRateLimiting(): void
    {
        RateLimiter::for('api', function (Request $request) {
            return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
        });

        RateLimiter::for('uploads', function (Request $request) {
            return Limit::perMinute(10)->by($request->user()?->id);
        });
    }
}
```

Apply with `->middleware('throttle:uploads')`.

## Controllers

Generate with `make:controller`. In Laravel 10, running a `make:` command without arguments prompts interactively for what it needs.

```bash
php artisan make:controller PhotoController --resource            # CRUD methods
php artisan make:controller PhotoController --resource --model=Photo   # + typed model hints
php artisan make:controller ProvisionServer --invokable           # single __invoke()
php artisan make:controller PhotoController --api                 # resource minus create/edit
```

### Plain controller

```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Photo;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PhotoController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        $photo = Photo::create($request->validated());

        return redirect()->route('photos.show', $photo);
    }
}
```

The skeleton's base `Controller` uses `AuthorizesRequests` and `ValidatesRequests`, so `$this->authorize(...)` and `$this->validate(...)` work in subclasses. Not extending it is fine — you just lose those helpers plus the controller-level `middleware()` method:

```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;

class PhotoController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
        $this->middleware('log')->only('index');
        $this->middleware('subscribed')->except('store');
    }
}
```

### Invokable (single action) controllers

When one endpoint deserves its own class — a favorite Laravel 10 pattern for focused endpoints, and validation works inside `__invoke` exactly like any method:

```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Process;
use Symfony\Component\HttpFoundation\Response;

class ProvisionServer extends Controller
{
    public function __invoke(Request $request): Response
    {
        $validated = $request->validate([
            'name' => ['required', 'string', 'max:255'],
        ]);

        $result = Process::run('provision '.$validated['name']);

        return response($result->output(), $result->successful() ? 200 : 422);
    }
}
```

Register it by class name — no method needed:

```php
use App\Http\Controllers\ProvisionServer;

Route::post('/server', ProvisionServer::class);
```

### Resource controllers

```php
Route::resource('photos', PhotoController::class);

Route::resources([
    'photos' => PhotoController::class,
    'posts' => PostController::class,
]);

// Limit to a subset
Route::resource('photos', PhotoController::class)->only(['index', 'show']);
Route::resource('photos', PhotoController::class)->except(['create', 'edit']);

// Nested
Route::resource('photos.comments', PhotoCommentController::class)->shallow();
```

The standard verb/URI/name mapping (`photos.index`, `photos.create`, `photos.store`, `photos.show`, `photos.edit`, `photos.update`, `photos.destroy`) — verify your exact surface with `php artisan route:list`.

## API routes (no `install:api` on 10)

`routes/api.php` exists from the start on 10.x. Adding more route files requires a `RouteServiceProvider` change (this is the Laravel 10 way; there is no `php artisan install:api`):

```php
// app/Providers/RouteServiceProvider.php — boot()
$this->routes(function () {
    Route::middleware('api')
        ->prefix('api')
        ->group(base_path('routes/api.php'));

    Route::middleware('web')
        ->group(base_path('routes/web.php'));
});
```

## Route caching

```bash
php artisan route:cache    # production only; requires no closure-based routes
php artisan route:clear
```

`route:cache` fails if any route uses a closure — controllers only. Stale route caches are a classic "my new route 404s" cause in deployed apps; run `route:clear` when in doubt.
