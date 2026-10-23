# Blade, Frontend, and Starter Kits

## Choosing a frontend approach

| Approach | Use when | Kit |
|---|---|---|
| Blade + Livewire | PHP-first, dynamic UIs without writing much JS | Livewire starter kit |
| Blade + Inertia (React/Svelte/Vue) | SPA feel with server-side routing/data | React / Svelte / Vue starter kits |
| Laravel as API backend | Next.js/mobile frontend, tokens via Sanctum | no kit (or minimal) |

Starter kits (React, Svelte, Vue, Livewire) ship auth (login, registration, password reset, email verification, optional 2FA), teams, and configurable layouts/sidebar variants; the WorkOS AuthKit option provides hosted enterprise auth. Community kits: `laravel new my-app --using=vendor/starter-kit`. Kits are a starting point — they live in your app and you own them.

## Blade fundamentals

Views live in `resources/views` as nested dot paths: `view('admin.profile', [...])` → `resources/views/admin/profile.blade.php`.

```blade
{{-- resources/views/layout.blade.php or components/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <title>@yield('title', config('app.name'))</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    {{ $slot }} {{-- in a component layout --}}
</body>
</html>
```

Escaping: `{{ $value }}` (escaped — always this for user data), `{!! $trustedHtml !!}` (raw — only for HTML you generated or sanitized).

Control structures:

```blade
@if (count($records) === 0)
    <p>No records.</p>
@elseif ($records->isNotEmpty())
    <p>Has records.</p>
@endif

@foreach ($users as $user)
    <li>{{ $loop->iteration }}: {{ $user->name }} @if ($loop->last)(end)@endif</li>
@endforeach

@forelse ($posts as $post)
    <p>{{ $post->title }}</p>
@empty
    <p>No posts.</p>
@endforelse

@isset($record) ... @endisset
@auth <a href="/dashboard">Dashboard</a> @else <a href="/login">Login</a> @endauth
@can('update', $post) ... @endcan
```

Forms: always `@csrf` inside POST/PUT/PATCH/DELETE forms; method spoofing `@method('PUT')`. Validation errors: `@error('title') <p>{{ $message }}</p> @enderror`; repopulate with `old('title')`:

```blade
<input type="text" name="title" value="{{ old('title') }}" class="@error('title') is-invalid @enderror" />
```

## Components (the modern organizing unit)

```bash
php artisan make:component Alert
```

Class-based components pair `app/View/Components/Alert.php` with `resources/views/components/alert.blade.php`:

```php
class Alert extends Component
{
    public function __construct(
        public string $type = 'info',
    ) {}

    public function render(): View
    {
        return view('components.alert');
    }
}
```

```blade
{{-- usage --}}
<x-alert type="error" :message="$errorMessage" />

{{-- resources/views/components/alert.blade.php --}}
<div {{ $attributes->merge(['class' => 'alert']) }}>
    {{ $message ?? $slot }}
</div>
```

- Attributes bag: `$attributes->merge(['class' => '...'])`, `$attributes->only('x-data')`.
- Named slots: `<x-slot:title>...</x-slot:title>` → `$title`.
- Anonymous components: just a Blade file in `components/`, with `@props(['type' => 'info'])` at the top — no PHP class needed. Prefer anonymous for presentational pieces; class-based when logic is involved.
- Inline: `make:component Alert --inline` (render string in the class).

Layouts as components: `<x-layouts-app>` with `{{ $slot }}` — this is the starter-kit pattern; `@extends/@yield/@section` still works but components are the current idiom.

## Livewire essentials (when the kit uses it)

```php
use Livewire\Component;

class SearchPosts extends Component
{
    public string $query = '';

    #[Computed]
    public function posts()
    {
        return Post::where('title', 'like', "%{$this->query}%")->limit(10)->get();
    }

    public function render()
    {
        return view('livewire.search-posts');
    }
}
```

```blade
<input wire:model.live.debounce.300ms="query">
@foreach ($posts as $post) <p>{{ $post->title }}</p> @endforeach
```

Actions are public methods (`wire:click="save"`), validation uses `#[Validate]` attributes or `$this->validate()`. Keep heavy queries in `#[Computed]` properties so they memoize per-request.

## Vite (asset bundling)

```blade
@vite(['resources/css/app.css', 'resources/js/app.js'])
@vite('resources/js/app.js', 'build')     // custom build dir/config
```

```bash
npm install && npm run build     # production build
npm run dev                      # dev server (HMR) — normally via composer run dev
```

Config in `vite.config.js` uses `laravel-vite-plugin`; cold assets after deploy issues are usually missing `npm run build` or a stale `hot` file — `php artisan optimize:clear` + rebuild. Tailwind CSS 4 is the starter-kit default.

## Inertia essentials (React/Svelte/Vue kits)

Controllers return Inertia responses, not views:

```php
public function index(): Response
{
    return Inertia::render('Posts/Index', [
        'posts' => Post::paginate(),
    ]);
}
```

Shared props via `HandleInertiaRequests` middleware (`share()` method — auth user, flash messages). SSR is built in (`@inertiajs/server`); enable in kit config. Validation errors arrive as Inertia error bags automatically (422 responses from form requests work unchanged).

## Frontend polish rules

- Prefer components over repeated Blade partials; colocate component styles.
- Use `@class([...])` for conditional classes, `old()` + `@error` for every form field.
- Flash messages: `session(['status' => 'Saved'])` → `<x-alert>` reading `session('status')`.
- CSRF for fetch/Axios: token is in the `XSRF-TOKEN` cookie; set `X-XSRF-TOKEN` header (most HTTP clients do this automatically with `withCredentials`).
