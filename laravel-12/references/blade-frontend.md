# Blade Templates and Frontend Assets (Laravel 12)

Blade is Laravel's templating language: compiled to plain PHP, cached, and safe by default. Use this reference for Blade views, components, layouts, and Vite asset wiring. For full-stack JavaScript (Inertia/React/Vue/Livewire starter kits), see `starter-kits-frontend.md`.

## Template basics

```blade
{{-- resources/views/posts/index.blade.php --}}
@extends('layouts.app')

@section('title', 'All posts')

@section('content')
    <h1>All posts</h1>

    @forelse ($posts as $post)
        <article>
            <h2><a href="{{ route('posts.show', $post) }}">{{ $post->title }}</a></h2>
            <p>{{ Str::limit($post->body, 160) }}</p>
        </article>
    @empty
        <p>No posts yet.</p>
    @endforelse

    {{ $posts->links() }}
@endsection
```

`{{ $value }}` escapes via `htmlspecialchars` — that's the default because unescaped output (`{!! $html !!}`) is an XSS hole and must be reserved for HTML you generated and trust. Control directives mirror PHP (`@if/@elseif/@else/@endif`, `@foreach/@endforeach`, `@isset/@empty/@auth/@guest/@error/@csrf/@method/@json/@include/@each`) — use them instead of raw `<?php ?>` so templates stay scannable.

Forms need CSRF protection and method spoofing for non-GET verbs:

```blade
<form method="POST" action="{{ route('posts.store') }}">
    @csrf
    @method('PUT') {{-- only when faking PUT/PATCH/DELETE --}}

    <label for="title">Title</label>
    <input id="title" type="text" name="title" value="{{ old('title') }}"
           class="@error('title') is-invalid @enderror" />
    @error('title')
        <p class="text-red-600 text-sm">{{ $message }}</p>
    @enderror
</form>
```

## Components (the modern layout pattern)

Prefer **component layouts** over `@extends/@section` for new work — slots compose better than named sections:

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ $title ?? config('app.name') }}</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body class="antialiased">
    {{ $slot }}
</body>
</html>
```

```blade
{{-- resources/views/posts/index.blade.php --}}
<x-layouts.app>
    <x-slot:title>All posts</x-slot:title>

    <x-posts.listing :posts="$posts" />
</x-layouts.app>
```

**Class-based components** (`php artisan make:component Alert`) live in `App\View\Components` with a `resources/views/components/alert.blade.php` view and can carry logic:

```php
<?php

namespace App\View\Components;

use Illuminate\Contracts\View\View;
use Illuminate\View\Component;

class Alert extends Component
{
    public function __construct(
        public string $type = 'info',
    ) {}

    public function shouldBeError(): bool
    {
        return in_array($this->type, ['error', 'danger'], true);
    }

    public function render(): View
    {
        return view('components.alert');
    }
}
```

```blade
{{-- resources/views/components/alert.blade.php --}}
@props(['type' => 'info', 'dismissible' => false])

<div {{ $attributes->merge(['class' => "alert alert-{$type}"]) }}>
    @if ($dismissible)
        <button data-dismiss>x</button>
    @endif

    {{ $slot }}
</div>
```

Rules of thumb: **anonymous components** (just a Blade file under `resources/views/components/`) for markup-only pieces; **class-based** when there's logic worth testing. `@props(['type' => 'info'])` declares the public API of an anonymous component; everything else lands on `$attributes` (merge classes, add data attributes: `<x-alert :type="$t" data-track="signup" />`).

Components can be organized into subdirectories: `resources/views/components/forms/input.blade.php` → `<x-forms.input />`.

## Views and data

Return views from controllers; keep view logic in the template or a view composer:

```php
return view('posts.index', ['posts' => $posts]);
return view('posts.show', compact('post'));           // fine, but explicit arrays read better in review
```

`@include('partials.post-meta', ['post' => $post])` shares data into a partial; `view('posts.show', ['post' => $post])->with('stats', $stats)` chains extra data. `View::share('key', $value)` (global, use sparingly) and `View::composer('profile::*', fn ($view) => ...)` cover cross-cutting view data.

## Vite asset bundling

Laravel 12 uses **Vite** (not Mix — never install Laravel Mix in a 12 app). `vite.config.js` at the project root loads the official Laravel plugin; Blade pages load assets with one directive:

```blade
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

Commands: `npm run dev` starts the HMR dev server; `npm run build` produces hashed production assets to `public/build`. `composer run dev` (the skeleton's Composer script) runs the Laravel server, queue worker, and Vite dev server together — the intended local loop.

Environment variables exposed to JS are prefixed `VITE_` and read via `import.meta.env.VITE_APP_NAME`. On Laravel Cloud/Forge-style deployments, run `npm run build` and `php artisan optimize` during release.

```js
// vite.config.js (default shape)
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```

## JavaScript sprinkles with Alpine

For Blade/Livewire apps, **Alpine.js** is the sanctioned way to add small interactions (dropdowns, modals) without a build-time framework:

```blade
<div x-data="{ open: false }">
    <button @click="open = ! open">Toggle</button>
    <div x-show="open" x-transition>...</div>
</div>
```

Reserve React/Vue-level work for the Inertia starter-kit path — mixing a hand-rolled SPA into a Blade app gives you the complexity of both and the benefits of neither.

## Localizing strings

`__('messages.welcome')` / `trans()` in templates and code; translation files live in `lang/` (publish with `php artisan lang:publish`). In Blade, `{{ __('posts.created') }}` keeps copy out of templates and makes i18n possible later.
