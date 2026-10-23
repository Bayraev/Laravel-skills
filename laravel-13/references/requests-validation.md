# Requests and Validation

## Reading input

```php
public function store(Request $request): RedirectResponse
{
    $name = $request->input('name', 'default');
    $nested = $request->input('products.0.name');
    $boolean = $request->boolean('subscribed');        // "1", "true", "on" → true
    $all = $request->all();
    $only = $request->only(['name', 'email']);
    $except = $request->except(['password']);

    if ($request->filled('name')) { /* non-empty present */ }
    if ($request->has('name')) { /* key present */ }
    if ($request->missing('coupon')) { /* absent */ }

    $request->merge(['slug' => Str::slug($request->title)]);
    $request->whenHas('coupon', fn ($value) => /* ... */);
}
```

The global `TrimStrings` + `ConvertEmptyStringsToNull` middleware run by default — optional fields arrive as `null`, so mark them `nullable` in rules. Uploads: `$request->file('avatar')` (an `UploadedFile`), validate then `$request->file('avatar')->store('avatars')`.

## Quick validation (in-controller)

```php
$validated = $request->validate([
    'title' => ['required', 'string', 'max:255'],   // array syntax — the current documented style
    'body' => ['required', 'string'],
    'publish_at' => ['nullable', 'date'],
]);
```

On failure: web requests redirect back with errors flashed (Blade reads `$errors`); XHR/JSON requests get **422** with `{ message, errors: { field: [messages] } }` (nested keys flattened to dot notation like `users.0.email`). Escape literal dots in field names with a backslash: `'v1\.0'`.

## Form requests — the default for anything non-trivial

```bash
php artisan make:request StorePostRequest
```

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\Attributes\FailOnUnknownFields;
use Illuminate\Foundation\Http\FormRequest;

#[FailOnUnknownFields]                    // reject payload keys not present in rules()
class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Post::class);   // or just `return true;` if authorized elsewhere
    }

    /** @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string> */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255', 'unique:posts'],
            'body' => ['required', 'string'],
            'publish_at' => ['nullable', 'date'],
        ];
    }

    public function messages(): array
    {
        return ['title.required' => 'A title is required'];
    }

    public function attributes(): array
    {
        return ['email' => 'email address'];
    }

    protected function prepareForValidation(): void
    {
        $this->merge(['slug' => Str::slug($this->title)]);
    }

    public function after(): array           // extra checks post-validation
    {
        return [
            fn (Validator $validator) => $validator->errors()->add('field', '...') ,
        ];
    }
}
```

Controller usage — validation runs before the method body:

```php
public function store(StorePostRequest $request): RedirectResponse
{
    $validated = $request->validated();
    $partial = $request->safe()->only(['title', 'body']);

    $post = $request->user()->posts()->create($validated);

    return to_route('posts.show', $post);
}
```

### Form request attributes (13)

| Attribute | Effect |
|---|---|
| `#[StopOnFirstFailure]` | Stop validating all attributes after first failure |
| `#[FailOnUnknownFields]` | Reject unexpected input keys (opt out per-class with `#[FailOnUnknownFields(false)]`; enable globally with `FormRequest::failOnUnknownFields()` in a provider `boot()`) |
| `#[RedirectTo('/dashboard')]` / `#[RedirectToRoute('dashboard')]` | Custom failure redirect target |
| `#[ErrorBag('login')]` | Put errors in a named bag (multiple forms per page) |

`#[FailOnUnknownFields]` hardens APIs against mass-assignment-style leaks — but it complements, not replaces, `$fillable` on models.

## Common rules worth memorizing

`required`, `nullable`, `sometimes`, `string`, `integer`, `numeric`, `boolean`, `array`, `date`, `email`, `url`, `uuid`, `ulid`, `min:/max:` (length for strings, value for numbers), `between`, `in:a,b`, `not_in:`, `confirmed` (pairs with `field_confirmation`), `same:field`, `different:field`, `unique:table,column` (with `->ignore($id)` object form), `exists:table,column`, `regex:`, `alpha_dash`, `starts_with:`, `ends_with:`, `digits:4`, `image`, `file`, `mimes:jpg,png`, `max:2048` (KB for files), `accepted`, `declined`, `prohibited`, `required_if:field,value`, `required_unless:`, `required_with:`, `exclude_if:field,value` (removes key from validated output), `bail` (stop this attribute's rules on first failure), `distinct`, `in_array:field.*`.

Conditional rules at runtime:

```php
Validator::make($data, [
    'plan' => ['required', Rule::in(['free', 'pro'])],
    'company' => [Rule::requiredIf($data['plan'] === 'pro')],
    'email' => [Rule::unique('users')->ignore($user->id)],
]);
```

## Validating nested arrays

```php
$request->validate([
    'title' => ['required', 'string'],
    'authors.*.name' => ['required', 'string'],
    'authors.*.email' => ['required', 'email'],
    'tags' => ['array', Rule::in(Tag::pluck('name'))],
]);
```

## Custom rule objects

```bash
php artisan make:rule Uppercase
```

```php
use Illuminate\Contracts\Validation\ValidationRule;

class Uppercase implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (mb_strtoupper($value) !== $value) {
            $fail('The :attribute must be uppercase.');
        }
    }
}
```

Usage: `'name' => [new Uppercase]`. Use `InvokableRule` with `ValidationRule` for implicit rules (that run even when the field is empty/absent). For one-off checks, a closure works inline: `'name' => [fn (string $attribute, mixed $value, Closure $fail) => $value === 'x' || $fail("{$attribute} must be x")]`.

## Manual validators

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), $rules, $messages, $attributes);

if ($validator->fails()) {
    return redirect('/post/create')->withErrors($validator)->withInput();
}

$validator->stopOnFirstFailure()->validate();         // validate() re-enables auto-redirect/422
$validator->after(fn ($v) => /* add errors */);
$errors = $validator->errors();                        // MessageBag: ->first('email'), ->get('email'), ->any(), ->has()
```

Named error bag: `->validateWithBag('post', [...])`; read in Blade `@error('title', 'post')`.

## Language files

The app skeleton ships without `lang/`; publish with `php artisan lang:publish` to customize `lang/en/validation.php` messages (`custom`, `attributes`, `values` arrays).
