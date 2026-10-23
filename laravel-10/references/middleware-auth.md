# Middleware, Auth, Gates/Policies, Sanctum

Sources: [middleware](https://laravel.com/framework/docs/10.x/middleware), [authentication](https://laravel.com/framework/docs/10.x/authentication), [authorization](https://laravel.com/framework/docs/10.x/authorization), [sanctum](https://laravel.com/framework/docs/10.x/sanctum).

**The Laravel 10 rule: middleware is registered in `app/Http/Kernel.php`.** (Laravel 11 moved this to `bootstrap/app.php` — don't follow that advice here.)

## Writing middleware

```bash
php artisan make:middleware EnsureTokenIsValid
```

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('home');
        }

        return $next($request);
    }
}
```

The 10.x stub returns `Symfony\Component\HttpFoundation\Response` — keep that type. Work before `$next` runs = before the controller; work after = after the response is built (`$response = $next($request); ...; return $response;`).

### Middleware parameters

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    public function handle(Request $request, Closure $next, string $role): Response
    {
        if (! $request->user()->hasRole($role)) {
            return redirect('dashboard');
        }

        return $next($request);
    }
}
```

```php
Route::put('/post/{id}', fn (string $id) => /* ... */)->middleware('role:editor');
```

### Terminable middleware

Add `terminate(Request $request, Response $response): void` for post-response work (FastCGI). Laravel resolves a fresh instance for `terminate()`; register the class as a singleton in `AppServiceProvider::register()` if state must persist between `handle`/`terminate`.

## Registering — `app/Http/Kernel.php`

```php
// Runs on every request
protected $middleware = [ /* ... */ ];

// Assignable by key
protected $middlewareGroups = [
    'web' => [ /* EncryptCookies, AddQueuedCookiesToResponse, StartSession, ShareErrorsFromSession, VerifyCsrfToken, SubstituteBindings */ ],
    'api' => [ /* ThrottleRequests:'api', SubstituteBindings */ ],
];

// Route-level aliases ('auth', 'guest', 'throttle', ...) — renamed from $routeMiddleware in 10.0
protected $middlewareAliases = [
    'auth' => \App\Http\Middleware\Authenticate::class,
    'is.admin' => \App\Http\Middleware\EnsureUserIsAdmin::class,  // your additions
];
```

Assign to routes: `->middleware('auth')`, `->middleware([First::class, Second::class])`, or groups `->middleware('web')`. Exclude with `->withoutMiddleware([...])` (route middleware only — never global).

Order matters when a middleware depends on another (auth before roles). Rarely needed, but `$middlewarePriority` forces an order for non-global middleware.

## Authentication basics

The `auth` alias (`\App\Http\Middleware\Authenticate::class`) guards routes; `auth.basic` uses HTTP basic; `guest` redirects authenticated users away from login/register.

```php
use Illuminate\Support\Facades\Auth;

Auth::attempt(['email' => $email, 'password' => $password], $remember = false); // session login
Auth::user();                          // current user or null
Auth::id();
Auth::logout();                        // kills the session
Auth::login($user, $remember = false); // login an existing model
$request->session()->regenerate();     // always after login (session fixation)
```

Out-of-the-box routes come from starter kits (Breeze/Jetstream) or hand-rolled controllers — Laravel 10 ships no `Auth::routes()` (that was ui package territory). Password resets use the `password` broker: `Password::sendResetLink($credentials)` / `Password::reset($creds, callback)`.

Guards are defined in `config/auth.php` (`web` session, `api` token). Access a specific guard: `auth('admin')->user()`, `Auth::guard('admin')`.

## Authorization — gates and policies

Prefer policies for model CRUD, gates for one-off actions unrelated to a model.

### Gates — defined in a provider's `boot()` (AuthServiceProvider by convention)

```php
namespace App\Providers;

use App\Models\Post;
use App\Models\User;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Gate::define('update-post', function (User $user, Post $post) {
            return $user->id === $post->user_id;
        });

        // Before-hook: runs first, e.g. grant admins everything
        Gate::before(fn (User $user) => $user->is_admin ? true : null);
    }
}
```

```php
if (Gate::allows('update-post', $post)) {}
$user->can('update-post', $post);
Gate::authorize('update-post', $post);              // throws 403 — inside controllers
Gate::any(['create-post', 'update-post'], $post);
Gate::none([...]);
```

`Gate::authorize()` returns `Response` with `->allowed()`, `->denied()`, `->message()` — useful for custom responses.

### Policies — auto-discovered, no registration needed

```bash
php artisan make:policy PostPolicy --model=Post
```

Auto-discovery maps `App\Models\Post` → `App\Policies\PostPolicy` (conventional location; explicit entries go in `AuthServiceProvider::$policies`). `registerPolicies()` is called automatically in Laravel 10.

```php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class PostPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function create(User $user): bool
    {
        return $user->can_publish;
    }

    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
}
```

Route/controller usage:

```php
// Middleware — policy auto-mapping: 'can:update,post' → PostPolicy@update with {post} resolved
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('can:update,post');

// Controller helper (from AuthorizesRequests on the base Controller)
$this->authorize('update', $post);
```

In Blade templates:

```blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@else
    <span>Read only</span>
@endcan
```

Note that `create` and `viewAny` receive no model instance (none exists yet / it's a list) — their policy methods take only the user, and callers pass the class name: `$user->can('create', Post::class)`.

## Sanctum — API tokens & SPA auth

Install per Sanctum's 10.x docs: `composer require laravel/sanctum`, publish config/migrations, add the `auth:sanctum`-supporting middleware. Sanctum abilities middleware must be aliased in `app/Http/Kernel.php` on Laravel 10:

```php
protected $middlewareAliases = [
    // ...
    'abilities' => \Laravel\Sanctum\Http\Middleware\CheckAbilities::class,
    'ability' => \Laravel\Sanctum\Http\Middleware\CheckForAnyAbility::class,
];
```

Issuing and using tokens:

```php
use App\Models\User;
use Illuminate\Http\Request;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);          // no abilities
    $token = $request->user()->createToken('name', ['server:update']);     // with abilities

    return ['token' => $token->plainTextToken];  // shown once; stored SHA-256-hashed
});

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn (Request $request) => $request->user());

    Route::get('/servers', fn (Request $request) => /* ... */)
        ->middleware('ability:servers:read');
});
```

SPA authentication: Sanctum's cookie-based scheme uses `/sanctum/csrf-cookie` + the `stateful` configuration in `config/sanctum.php` — the SPA requests hit the same domain, and session auth applies (no token juggling). First-party mobile apps use bearer tokens instead.
