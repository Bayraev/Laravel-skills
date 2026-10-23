# Starter Kits and Frontend Stacks (Laravel 12)

The 12.x-specific frontend story. Laravel 12 replaced Breeze/Jetstream (both frozen — no further updates) with new application starter kits, and added WorkOS AuthKit variants. Source: https://laravel.com/docs/12.x/starter-kits

## Choosing a stack

| You want... | Use |
|---|---|
| PHP-only, server-rendered, minimal JS | Blade + Alpine (no kit needed; `laravel new` with no starter kit) |
| Dynamic UIs while staying in PHP | **Livewire kit** (Livewire 4 line, Tailwind, Flux UI, Volt) |
| React SPA with server routing | **React kit** (Inertia 2, React 19, TypeScript, Tailwind, shadcn/ui) |
| Vue 3 Composition API | **Vue kit** (Inertia 2, Vue 3, TypeScript, Tailwind, shadcn-vue) |
| Svelte 5 | **Svelte kit** (Inertia 2, Svelte 5, TypeScript, Tailwind, shadcn-svelte) |
| Social login / passkeys / SSO out of the box | Any kit, **WorkOS AuthKit variant** |
| API-only backend | No kit; `php artisan install:api` + Sanctum (`routing-controllers.md`) |

## Creating an app with a kit

```shell
laravel new my-app          # prompts for testing framework, database, starter kit
cd my-app
npm install && npm run build
composer run dev            # Laravel server + queue worker + Vite, http://localhost:8000
```

Community kits install directly: `laravel new my-app --using=vendor/starter-kit`.

Important mental model: **there is no upgrade path for a kit, by design**. All backend and frontend code is published into your app — you own it. Kits are starting points, not dependencies.

## What every kit includes

All four kits share the same backend: **Laravel Fortify** provides the authentication machinery — routes and controllers for login, registration, password reset, email verification, password confirmation, and **two-factor authentication (2FA, enabled by default)**.

- Fortify routes are registered based on the `features` array in `config/fortify.php`: `Features::registration()`, `Features::resetPasswords()`, `Features::emailVerification()`, `Features::twoFactorAuthentication(['confirm' => true, 'confirmPassword' => true])`. Comment a feature out to disable it.
- Registration and password-reset logic is customized in **action classes** under `app/Actions/Fortify/`: `CreateNewUser.php`, `ResetUserPassword.php`, `PasswordValidationRules.php`. Edit those — don't fork Fortify.
- Inspect the full route surface with `php artisan route:list`.

```php
// app/Actions/Fortify/CreateNewUser.php — the sanctioned customization point
public function create(array $input): User
{
    Validator::make($input, [
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'email', 'max:255', 'unique:users'],
        'password' => $this->passwordRules(),
    ])->validate();

    return User::create([
        'name' => $input['name'],
        'email' => $input['email'],
        'password' => Hash::make($input['password']),
    ]);
}
```

Email verification: uncomment `MustVerifyEmail` on the `User` model and protect routes with `['auth', 'verified']` middleware.

**JS kits use Wayfinder** for type-safe, build-time-generated route definitions. If you disable a Fortify feature, also remove frontend references to its routes or **the build fails** — that's Wayfinder enforcing correctness, not a bug.

## Inertia (React / Svelte / Vue kits)

Inertia bridges Laravel routing/controllers to SPA pages: controllers return `Inertia::render(...)` instead of Blade views, and the named page component (in `resources/js/pages`) hydrates with the passed props.

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Inertia\Inertia;
use Inertia\Response;

class UserController extends Controller
{
    public function show(string $id): Response
    {
        return Inertia::render('users/show', [
            'user' => User::findOrFail($id),
        ]);
    }
}
```

Frontend code lives in `resources/js/` with a stable shape across the three JS kits:

```
resources/js/
├── components/    # reusable components (+ components/ui/ for published shadcn)
├── composables/   # (Vue) composables / hooks
├── layouts/       # app layouts + auth layouts
├── lib/           # utilities
├── pages/         # Inertia page components
└── types/         # TypeScript definitions
```

Publish more shadcn components as needed — e.g. `npx shadcn@latest add switch` (React), `npx shadcn-vue@latest add switch` (Vue), `npx shadcn-svelte@latest add switch` (Svelte).

**Layout variants** (documented, cheap to switch): app layout "sidebar" (default) or "header" — swap the import at the top of `resources/js/layouts/app-layout.tsx` / `AppLayout.svelte` / `AppLayout.vue`; sidebar variant "sidebar" / "inset" / "floating"; auth layout "simple" / "card" / "split".

**SSR** is supported when needed:

```shell
npm run build:ssr      # build the SSR bundle
composer dev:ssr       # local dev: server + SSR together
```

## Livewire kit

Frontend code stays in Blade — `resources/views/` (not `resources/js/`), with Flux UI components (`<flux:...>`) and Volt/Livewire pages:

```
resources/views/
├── components/           # reusable components
├── flux/                 # customized Flux components
├── layouts/              # app + auth layouts
├── pages/                # Livewire pages
└── partials/             # Blade partials
```

Livewire components pair a PHP class (or a Volt single-file component) with reactive Blade markup — `wire:click`, `wire:model` bind the frontend to backend state over AJAX without writing JS:

```php
<?php

use Livewire\Component;

new class extends Component
{
    public $count = 0;

    public function increment(): void
    {
        $this->count++;
    }
};
?>

<div>
    <button wire:click="increment">+</button>
    <h1>{{ $count }}</h1>
</div>
```

Layout variants work like the JS kits: switch the layout used by `resources/views/layouts/app.blade.php` (add the `container` attribute to `<flux:main>` when using the header layout) and `resources/views/layouts/auth.blade.php`.

## WorkOS AuthKit variants

Each kit has a WorkOS-powered variant (choose at `laravel new`) offering social authentication (Google, Microsoft, GitHub, Apple), passkeys, email Magic Auth, and SSO — free up to 1M monthly active users. Configure in `.env`:

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

Recommended settings from the docs: disable "Email + Password" in the WorkOS dashboard (skip password handling entirely), and align the AuthKit session inactivity timeout with your Laravel session lifetime (typically 2 hours). Email verification is unnecessary with WorkOS.

## Legacy kits

**Laravel Breeze and Laravel Jetstream will not receive additional updates.** For a new Laravel 12 app, don't install them. Existing Breeze/Jetstream apps keep working (Fortify v1 for Jetstream), but treat them as frozen — new auth work should follow the starter-kit patterns above. Vue 3 + Inertia + Tailwind remains available as a Jetstream stack for legacy parity only.
