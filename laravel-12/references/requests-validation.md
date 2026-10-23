# Requests and Validation (Laravel 12)

## Table of contents

- [Reading request input](#reading-request-input)
- [Quick validation in controllers](#quick-validation-in-controllers)
- [Form requests (the default choice)](#form-requests-the-default-choice)
- [Validated input](#validated-input)
- [Error responses and Blade display](#error-responses-and-blade-display)
- [Rule highlights and Laravel 12 notes](#rule-highlights-and-laravel-12-notes)
- [Custom rules](#custom-rules)
- [Validating arrays and nested data](#validating-arrays-and-nested-data)

## Reading request input

```php
use Illuminate\Http\Request;

public function store(Request $request): RedirectResponse
{
    $value = $request->input('name');
    $value = $request->input('products.0.name');          // dot notation
    $value = $request->query('search');                    // query string only
    $value = $request->input('name', 'default');           // with default

    $all = $request->all();
    $few  = $request->only(['name', 'email']);
    $rest = $request->except(['password']);

    if ($request->has('terms')) { /* key present */ }
    if ($request->filled('email')) { /* present AND non-empty */ }
    if ($request->missing('coupon')) { /* absent */ }
}
```

Remember that the global middleware `TrimStrings` + `ConvertEmptyStringsToNull` normalize input before you see it: empty string fields arrive as `null`. That's why optional fields carry `nullable` in rules.

## Quick validation in controllers

For prototypes and trivial endpoints, `$request->validate()` is honest and compact:

```php
$validated = $request->validate([
    'title' => 'required|string|max:255',
    'body' => 'required|string',
    'publish_at' => 'nullable|date',
]);
```

On failure: web requests redirect back with errors flashed to session; XHR requests get a **422 JSON** body (`{"message": "...", "errors": {"field": ["..."]}}`). No controller code needed for either.

Prefer **array syntax** for rules (`['required', 'string']`) — it's easier to diff, comment, and extend than pipe strings.

## Form requests (the default choice)

Once more than one rule or any authorization is involved, generate a form request:

```shell
php artisan make:request StorePostRequest
```

```php
<?php

namespace App\Http\Requests;

use App\Models\Post;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        return $this->user()->can('create', Post::class);
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'alpha_dash', Rule::unique('posts')],
            'body' => ['required', 'string'],
            'publish_at' => ['nullable', 'date', 'after:now'],
        ];
    }
}
```

Then the controller just type-hints it — validation and authorization run before the controller method executes:

```php
public function store(StorePostRequest $request): RedirectResponse
{
    $post = $request->user()->posts()->create($request->validated());

    return to_route('posts.show', $post);
}
```

Why form requests win: rules are testable and reusable, authorization is colocated with the data it guards, and fat controllers stay slim. `authorize()` returning `false` yields a 403 automatically; return `true` (or omit) when authorization happens elsewhere (policies + `can` middleware).

Lifecycle hooks worth knowing:

```php
// Normalize input BEFORE rules run:
protected function prepareForValidation(): void
{
    $this->merge(['slug' => Str::slug($this->input('title'))]);
}

// Extra cross-field checks AFTER rules pass:
public function after(): array
{
    return [
        function (Validator $validator): void {
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add('field', 'Something is wrong with this field!');
            }
        },
    ];
}
```

Also available: `messages()` (custom messages per `field.rule`), `attributes()` (friendlier `:attribute` names), `protected bool $stopOnFirstFailure = true;`, `protected $redirect` / `$redirectRoute` to override the failure redirect.

## Validated input

Only ever persist validated data — `$request->all()` invites mass-assignment surprises:

```php
$validated = $request->validated();                    // array of validated fields
$validated = $request->safe()->only(['name', 'email']);
$validated = $request->safe()->except(['internal_flag']);
```

## Error responses and Blade display

Blade always has `$errors` (a `MessageBag`) thanks to `ShareErrorsFromSession`:

```blade
<input type="text" name="title" value="{{ old('title') }}"
       class="@error('title') is-invalid @enderror" />

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

The JSON error contract (422) with nested keys flattened to dot notation — match this shape in frontend error handling:

```json
{
    "message": "The team name must be a string. (and 4 more errors)",
    "errors": {
        "team_name": ["The team name must be a string."],
        "authorization.role": ["The selected authorization.role is invalid."],
        "users.0.email": ["The users.0.email field is required."]
    }
}
```

## Rule highlights and Laravel 12 notes

- `nullable` — required for any optional field (because of `ConvertEmptyStringsToNull`).
- `bail` — stop validating an attribute after its first failure.
- `Rule::unique('users')->ignore($user)` and `Rule::exists('roles', 'id')` — parameterize the database rules instead of string concatenation.
- `enum:App\Enums\Category` or `Rule::enum(Category::class)` — validate backed enums; combine with `->only([...])` to restrict cases.
- `password` — validates against the current user's password (`PasswordValidationRules` in starter kits composes it).
- `File::image()->types(['jpg', 'png'])->max(2048)` / `File::image(allowSvg: true)` — object rules for uploads.
- **12.x gotcha**: the `image` rule **rejects SVGs by default** now. `image:allow_svg` (string) or `File::image(allowSvg: true)` (object) re-enables them — only where SVG uploads are actually safe.
- `prohibited_if:status,draft` family — reject fields that shouldn't be sent in a given state.

## Custom rules

**Rule objects** (`php artisan make:rule Uppercase`) — for reusable logic:

```php
<?php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class Uppercase implements ValidationRule
{
    /**
     * Run the validation rule.
     */
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (mb_strtoupper($value) !== $value) {
            $fail('The :attribute must be uppercase.');
        }
    }
}
```

Usage: `'name' => ['required', new Uppercase]`. Rule objects are resolved from the container, so they can take constructor dependencies.

**Closures** — for one-off checks (inside rules arrays):

```php
$request->validate([
    'title' => [
        'required',
        'string',
        fn (string $attribute, mixed $value, Closure $fail) =>
            $value === 'Laravel'
                ? $fail("{$attribute} may not be 'Laravel'.")
                : null,
    ],
]);
```

**Implicit rules** (run even when the field is empty — implement `Illuminate\Contracts\Validation\ImplicitRule` or use the `--implicit` flag on `make:rule`) are for rules like "required if X" whose contract includes emptiness.

## Validating arrays and nested data

```php
$request->validate([
    'title' => ['required', 'string'],
    'authors.*.name' => ['required', 'string'],
    'authors.*.email' => ['required', 'email'],
    'tags' => ['array'],           // and/or 'list' for sequential arrays
]);

$request->validate([
    'user.name' => ['required', 'string'],      // nested object input
]);
```

Error keys mirror the input shape (`authors.0.name`), so frontend code can map errors onto the right repeating row. To prepare nested defaults, `mergeIfMissing()` supports dot notation in 12: `$request->mergeIfMissing(['user.last_name' => 'Doe'])`.

For manual validators (non-HTTP contexts), use `Validator::make($data, $rules)` with the same rule syntax, then `->validate()` to throw, or `->fails()` / `->valid()` / `->errors()` to branch yourself.
