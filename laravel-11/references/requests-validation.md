# Laravel 11 Requests and Validation

Sources: https://laravel.com/framework/docs/11.x/validation and
https://laravel.com/framework/docs/11.x/requests.

## Reading request input

Type-hint `Illuminate\Http\Request` anywhere the container resolves (controllers, route closures):

```php
use Illuminate\Http\Request;

public function store(Request $request): RedirectResponse
{
    $name = $request->input('name');
    $name = $request->input('name', 'Sally');        // default
    $name = $request->input('products.0.name');      // dot notation for arrays
    $name = $request->query('name');                 // query string only

    $input = $request->all();                        // everything
    $input = $request->only(['username', 'password']);
}
```

Typed accessors (use these instead of casting in application code):

```php
$name     = $request->string('name')->trim();            // Stringable
$perPage  = $request->integer('per_page');               // int, safe default
$archived = $request->boolean('archived');               // "1"/"on"/"yes" => true
$birthday = $request->date('birthday');                  // Carbon instance
$status   = $request->enum('status', Status::class);     // backed enum or null
```

Presence checks: `$request->has('name')`, `$request->filled('name')` (present AND not empty),
`$request->missing('name')`. JSON bodies are accessible via `input()` when the request sends
`Content-Type: application/json`.

Remember: the global `TrimStrings` and `ConvertEmptyStringsToNull` middleware normalize input, so
optional fields arrive as `null`, not `''` — validate them as `nullable`.

## Quick validation: `$request->validate()`

```php
public function store(Request $request): RedirectResponse
{
    $validated = $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    // $validated contains only the validated keys/values...
    $post = Post::create($validated);

    return to_route('posts.show', $post);
}
```

Behavior on failure: traditional requests redirect back with errors flashed to the session; XHR
requests get a 422 JSON response:

```json
{
    "message": "The team name must be a string. (and 4 more errors)",
    "errors": {
        "team_name": ["The team name must be a string."],
        "users.0.email": ["The users.0.email field is required."]
    }
}
```

Array syntax for rules is preferred in real code (easier to append rules dynamically):

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
]);
```

## Form requests: the default for real endpoints

```shell
php artisan make:request StorePostRequest
```

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        return true;   // or $this->user()->can('create', Post::class)
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ];
    }
}
```

Type-hint it on the controller — validation and authorization run before the method body:

```php
public function store(StorePostRequest $request): RedirectResponse
{
    $validated = $request->validated();

    // Or a subset...
    $validated = $request->safe()->only(['title', 'body']);

    $post = Post::create($validated);

    return to_route('posts.show', $post);
}
```

Useful form request hooks:

```php
// Custom messages / attribute names...
public function messages(): array
{
    return ['title.required' => 'A title is required'];
}

public function attributes(): array
{
    return ['email' => 'email address'];
}

// Normalize input BEFORE validation...
protected function prepareForValidation(): void
{
    $this->merge(['slug' => Str::slug($this->slug)]);
}

// Extra checks AFTER the base rules pass...
public function after(): array
{
    return [
        function (Validator $validator) {
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add('field', 'Something is wrong with this field!');
            }
        },
    ];
}

/** Stop on the first failure. */
protected $stopOnFirstFailure = true;
```

`authorize()` returning false produces a 403 before validation runs.

## Manual validators

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
], [
    'required' => 'The :attribute field is required.',
]);

if ($validator->fails()) {
    return redirect('/post/create')
        ->withErrors($validator)
        ->withInput();
}

$validated = $validator->validated();
```

`->validate()` on a manual validator re-enables the automatic redirect/422 behavior;
`validateWithBag('post', [...])` targets a named error bag (needed when a page has multiple forms).

## Validating arrays and nested data

```php
$request->validate([
    'title' => 'required',
    'author.name' => 'required',                 // nested
    'users.*.email' => 'required|email',         // array of objects
    'users.0.email' => 'required|email',         // specific index
]);
```

Field names containing a literal dot are escaped: `'v1\.0' => 'required'`.

## Custom rule objects

```shell
php artisan make:rule Uppercase
```

```php
use Illuminate\Contracts\Validation\ValidationRule;

class Uppercase implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (strtoupper($value) !== $value) {
            $fail('The :attribute must be uppercase.');
        }
    }
}
```

## Displaying errors in Blade

```blade
@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

<input type="text" name="title" value="{{ old('title') }}" class="@error('title') is-invalid @enderror">

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

`$errors` is always defined in `web` views (shared by `ShareErrorsFromSession` middleware).
`old('field')` repopulates the form from the flashed input.

## Choosing an approach

- Controller action with 1–3 simple rules → `$request->validate([...])`.
- Anything reused, complex, nested, or needing authorization → form request.
- Validating non-request data (imports, DTOs) → `Validator::make()`.
- Validating CLI prompt input → Laravel Prompts' `validate:` now accepts full validator rule
  arrays: `text('What is your name?', validate: ['name' => 'required|min:3|max:255'])`.
