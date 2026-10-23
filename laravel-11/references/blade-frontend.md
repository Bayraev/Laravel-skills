# Laravel 11 Blade and Frontend

Source: https://laravel.com/framework/docs/11.x/blade (directives verified across the 11.x
routing/validation docs; Vite integration is the 11 default).

## Blade fundamentals

Views live in `resources/views` as `dot.nested.name.blade.php` —
`view('posts.show', ['post' => $post])` renders `resources/views/posts/show.blade.php`.

```blade
{{-- Escaped output (always default to this) --}}
{{ $post->title }}

{{-- Raw output — only for HTML you generated or sanitized yourself --}}
{!! $post->body_html !!}

@if ($post->published)
    Published
@elseif ($post->scheduled)
    Scheduled
@else
    Draft
@endif

@foreach ($posts as $post)
    <article>{{ $post->title }}</article>
@endforeach

@forelse ($posts as $post)
    <article>{{ $post->title }}</article>
@empty
    <p>No posts yet.</p>
@endforelse
```

`{{ }}` escapes via `htmlspecialchars` — it is what prevents stored XSS, so raw `{!! !!}` is a
deliberate, audited choice each time.

## Forms: CSRF and method spoofing

Every `POST`/`PUT`/`PATCH`/`DELETE` form under the `web` middleware group needs the CSRF token:

```blade
<form method="POST" action="/profile">
    @csrf

    <input type="hidden" name="_method" value="PUT">  {{-- or just: @method('PUT') --}}
    ...
</form>
```

`@method('PUT')` generates the hidden `_method` field (HTML forms only support GET/POST).

## Validation errors and old input

```blade
<input
    type="text"
    name="title"
    value="{{ old('title') }}"
    class="@error('title') is-invalid @enderror"
/>

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror

@error('title', 'post')
    {{-- from a named error bag: $errors->post->first('title') --}}
@enderror
```

`old('title', 'default')` pulls flashed input; `$errors` is always defined in web views.

## Layouts

Two supported styles. Use component layouts for new work:

**Component layout** — a layout component with a named slot:

```blade
{{-- resources/views/components/layouts/app.blade.php --}}
<html>
    <body>
        <main>
            {{ $slot }}
        </main>
    </body>
</html>

{{-- any view --}}
<x-layouts.app>
    <h1>{{ $post->title }}</h1>
</x-layouts.app>
```

**Template inheritance** (still common in older codebases):

```blade
{{-- resources/views/layouts/app.blade.php --}}
<html><body>@yield('content')</body></html>

@extends('layouts.app')

@section('content')
    <h1>{{ $post->title }}</h1>
@endsection
```

## Components

`php artisan make:component Alert` creates `app/View/Components/Alert.php` +
`resources/views/components/alert.blade.php`:

```blade
{{-- usage --}}
<x-alert type="error" :message="$errorMessage"/>

{{-- resources/views/components/alert.blade.php --}}
@props(['type' => 'info', 'message' => null])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

- `@props` declares the component's public API; everything else lands in `$attributes`.
- `:message="$var"` passes PHP values; `message="literal"` passes strings.
- `$attributes->merge(['class' => '...'])` combines consumer classes with your defaults.
- `$slot` renders the body; named slots via `<x-slot:title>`.
- Components under `resources/views/components/` (no class) are anonymous — use them for pure
  markup.

## Including and reuse

```blade
@include('partials.header', ['title' => 'Home'])   {{-- inherits parent data + overrides --}}
@each('partials.post-row', $posts, 'post')         {{-- renders per item, view when empty --}}
```

## Conditional classes and misc directives

```blade
@class(['p-4', 'font-bold' => $isFeatured])

@selected($option->id === $currentId)      {{-- selected="selected" --}}
@checked($user->active)                    {{-- checked="checked" --}}
@disabled($formLocked)

@csrf
@json($options)                            {{-- <script>const o = @js($options);</script> --}}
```

## Vite integration

Laravel 11 uses Vite (not Mix) with an `@vite` Blade directive:

```blade
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

`npm run dev` starts the dev server with hot reload; `npm run build` produces hashed production
assets. Custom build steps (Tailwind, Vue, React) edit `vite.config.js`. The `composer run dev`
script (new in 11) runs the server, queue worker, and Vite together.

## Frontend approaches at a glance

| Need | Use |
| --- | --- |
| Server-rendered pages | Blade + Vite |
| Livewire components | Livewire package, Blade-based reactivity |
| SPA | Inertia (Vue/React) rendering through Blade, or separate SPA + Sanctum (see `auth-basics.md`) |
| API only | No Blade — JSON responses; see `auth-basics.md` |

## Practical rules

- Prefer components over `@include` for anything with its own props/state.
- Never put business logic in Blade; format-only helpers are fine (`$post->title`, `Str::of(...)`).
- Escape by default; document every `{!! !!}` with a reason.
- Keep `web.php` routes returning views and `api.php` routes returning JSON/DTOs — don't mix.
