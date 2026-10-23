# Blade Templates and Frontend

Sources: [views](https://laravel.com/framework/docs/10.x/views), [blade](https://laravel.com/framework/docs/10.x/blade), [vite](https://laravel.com/framework/docs/10.x/vite).

## Views

```php
return view('gallery.photos', ['photos' => $photos]);   // → resources/views/gallery/photos.blade.php

view()->exists('gallery.photos');                        // check before rendering
return view('profile')->with('user', $user);             // alternative passing style
return response()->view('pdf', $data, 200, ['X-Header' => 'value']);
```

Share data with every view in a service provider: `View::share('key', $value)`. View composers (logic per-view) register via `View::composer('profile*', fn ($view) => ...)` in a provider's `boot()`.

## Template basics

```blade
{{ $name }}                          {{-- escaped output (HTML entities) --}}
{!! $html !!}                        {{-- raw output — only for trusted HTML --}}
{{ $title ?? 'Fallback' }}           {{-- null coalescing --}}
{{ strtoupper($name) }}              {{-- plain PHP works everywhere --}}

@if (count($photos) > 0)
    I have {{ count($photos) }} photos.
@elseif ($pending)
    Pending...
@else
    None yet.
@endif

@isset($record) @endisset
@empty($records) @endempty

@foreach ($photos as $photo)
    <p>{{ $loop->iteration }}/{{ $loop->count }}: {{ $photo->title }}</p>
    @if ($loop->first) first! @endif
    @if ($loop->last) last! @endif
@endforeach

@forelse ($photos as $photo)
    <p>{{ $photo->title }}</p>
@empty
    <p>No photos.</p>
@endforelse

{{ $user->name }} @if ($user->is_admin) (admin) @endif

@unless ($paid) Pay up. @endunless
@checked($active) @selected($country === 'UK') @disabled($locked)   {{-- boolean attributes --}}

{{ old('title', $post->title) }}     {{-- repopulate after validation redirect --}}

@json($data)                         {{-- inline JSON (e.g. props for a Vue component) --}}
```

## Layouts: components over `@extends`

The Laravel 10 idiom is component-based layouts (the old `@extends/@yield/@section` still works, but modern code — and Breeze — uses components):

```blade
{{-- resources/views/components/layouts/app.blade.php --}}
<html>
<body>
    {{ $slot }}
    @isset($header) {{ $header }} @endisset
</body>
</html>
```

```blade
{{-- resources/views/dashboard.blade.php --}}
<x-layouts.app>
    <x-slot name="header">
        <h1>Dashboard</h1>
    </x-slot>

    <main>
        {{-- content goes to {{ $slot }} --}}
    </main>
</x-layouts.app>
```

Anonymous components live in `resources/views/components/` (`<x-alert>` → `components/alert.blade.php`); `$attributes` carries extras:

```blade
{{-- components/alert.blade.php --}}
@props(['type' => 'info'])
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $slot }}
</div>
```

```blade
<x-alert type="error" class="mb-4" :message="$errorText" />   {{-- :prop = PHP value passthrough --}}
```

Class-based components (`php artisan make:component Alert` → `app/View/Components/Alert.php` + view) carry real logic: constructor promotion for props, an `execute`-style render method:

```php
namespace App\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    public function __construct(
        public string $type = 'info',
    ) {}

    public function render()
    {
        return view('components.alert');
    }
}
```

## Forms and validation display

```blade
<form method="POST" action="{{ route('photos.store') }}">
    @csrf

    <label for="title">Title</label>
    <input id="title"
           name="title"
           value="{{ old('title') }}"
           class="form-control @error('title') is-invalid @enderror" />

    @error('title')
        <div class="invalid-feedback">{{ $message }}</div>
    @enderror

    <button type="submit">Save</button>
</form>
```

`@csrf` is mandatory for every POST/PUT/PATCH/DELETE form in the `web` middleware group — omitting it produces a 419. `$errors` is always defined (no `isset` dance needed).

## Includes, conditionals, misc directives

```blade
@include('partials.photo-card', ['photo' => $photo])
@includeWhen($showCard, 'partials.photo-card')
@each('partials.photo-card', $photos, 'photo', 'partials.empty')   {{-- list rendering --}}

@auth I'm signed in. @endauth
@guest Please log in. @endguest
@can('update', $photo) <a href="...">Edit</a> @endcan
@role('admin') ... @endrole           {{-- requires a roles package (spatie) — not core --}}
@production Only in production. @endproduction
@once @endonce                        {{-- render once per request --}}
@push('scripts') <script>...</script> @endpush   +  @stack('scripts') in the layout
@vite('resources/js/app.js')          {{-- asset entry point --}}
```

## Vite assets (Laravel 10 default)

`resources/css/app.css` and `resources/js/app.js` are entry points wired through `vite.config.js`. Include in a layout with `@vite`:

```blade
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

```bash
npm install
npm run dev      # dev server (hot reload) — leave running while building UI
npm run build    # production bundles → public/build
```

Asset helpers for non-Vite files: `asset('img/logo.png')`, `secure_asset(...)`, `route('name')` for URLs. Uploaded/public storage files: `Storage::url('avatars/1.jpg')` after `php artisan storage:link`.

## Compilation and caching

Blade compiles automatically; stale compiled templates (`storage/framework/views`) usually clear themselves, but `php artisan view:clear` is the remedy when template edits seem invisible. `view:cache` pre-warms all views in production deploys.
