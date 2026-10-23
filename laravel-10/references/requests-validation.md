# Requests and Validation

Source: [requests](https://laravel.com/framework/docs/10.x/requests), [validation](https://laravel.com/framework/docs/10.x/validation). On failure, Laravel 10 validation **redirects back with errors** for traditional requests and returns **HTTP 422 JSON** for XHR — automatically, no handling code needed.

## Reading the request

```php
use Illuminate\Http\Request;

public function store(Request $request): RedirectResponse
{
    $request->input('name');                      // body or query string
    $request->query('search');                    // query string only
    $request->input('products.0.name');           // dotted access into arrays
    $request->boolean('subscribed');              // "1", "true", "on" → true
    $request->date('published_at');               // Carbon instance (optional format arg)
    $request->enum('status', ServerStatus::class); // enum instance or null

    $request->all();
    $request->only(['name', 'email']);
    $request->except(['password']);
    $request->has('name') / $request->filled('name') / $request->missing('name');

    $request->header('X-Header-Name');
    $request->ip();
    $request->path();        // e.g. "photos/1" (no leading slash)
    $request->url();         // full URL without query string
    $request->method();      // GET/POST...; $request->isMethod('post')
    $request->user();        // authenticated user model
    $request->user('admin'); // guard-scoped
}
```

Old input for repopulating forms after a validation redirect: `old('name')` / `request()->flash()`.

## Quick validation — `$request->validate()`

The right tool for small, controller-local rules. Works identically in any controller method, including an invokable controller's `__invoke`:

```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'title' => ['required', 'unique:posts', 'max:255'],  // array syntax preferred
            'body' => ['required'],
            'author.name' => ['required'],                       // nested fields via dot syntax
        ]);

        Post::create($validated);

        return redirect('/posts');
    }
}
```

- `$validated` contains only the validated keys — pass it straight to `create()`/`update()`.
- `'bail'` stops validating an attribute after its first failure; validation otherwise runs every rule.
- Escape literal dots in field names: `'v1\.0' => ['required']`.
- Named error bag: `$request->validateWithBag('post', [...])` when handling multiple forms per page.

For one-off error messages in a controller, `Validator::make()` gives manual control — but prefer a Form Request once rules grow beyond a handful.

## Form Requests — the default for anything non-trivial

```bash
php artisan make:request StorePostRequest
```

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Validator;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Post::class);  // false → 403
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'alpha_dash', Rule::unique('posts')],
            'body' => ['required', 'string'],
            'published_at' => ['nullable', 'date'],
            'category_id' => ['required', 'exists:categories,id'],
        ];
    }

    /** Extra messages for specific attribute.rule pairs. */
    public function messages(): array
    {
        return [
            'title.required' => 'A title is required.',
            'body.required' => 'A post body is required.',
        ];
    }

    /** Friendly names in messages instead of field names. */
    public function attributes(): array
    {
        return ['title' => 'post title'];
    }

    /** Cross-field checks that need the whole validated set. */
    public function after(): array
    {
        return [
            function (Validator $validator) {
                if ($this->input('start') >= $this->input('end')) {
                    $validator->errors()->add('start', 'Start must precede end.');
                }
            },
        ];
    }

    /** Normalize input before rules run. */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->input('title')),
        ]);
    }
}
```

Type-hint it in the controller — validation and authorization run before your method body, so the method only ever sees clean, authorized data:

```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use Illuminate\Http\RedirectResponse;

class PostController extends Controller
{
    public function store(StorePostRequest $request): RedirectResponse
    {
        $validated = $request->validated();                 // array of validated fields
        $safe = $request->safe()->only(['title', 'body']);  // subset

        $post = $request->user()->posts()->create($validated);

        return redirect()->route('posts.show', $post);
    }
}
```

Convention: one request class per action when rules differ — `StorePostRequest`, `UpdatePostRequest`. When `update` rules are `store` rules minus `unique`, reuse with `Rule::unique('posts')->ignore($this->route('post'))`.

## Error format and display

Failed validation on XHR/API (automatic):

```json
{
    "message": "The title field is required. (and 1 more error)",
    "errors": {
        "title": ["The title field is required."],
        "body": ["The body field is required."]
    }
}
```

In Blade, `$errors` (an `Illuminate\Support\MessageBag`) is always defined thanks to `ShareErrorsFromSession` in the `web` group:

```blade
@error('title')
    <p class="error">{{ $message }}</p>
@enderror

<input name="title" value="{{ old('title') }}" />
```

## Rule cheat sheet (commonly needed)

`required` `nullable` `sometimes` `string` `integer` `numeric` `boolean` `array` `date` `email` `uuid` `url`
`confirmed` (matches `{field}_confirmation`) `same:field` `different:field` `in:a,b` `not_in:...` `min:3` `max:255` `between:1,10` `size:6`
`unique:table,column` (use `Rule::unique` for ignores) `exists:table,column` `in_array:other.*` `distinct`
`file` `image` `mimes:jpg,png` `max:2048` (kilobytes for files)

Rules with parameters that take user input or model IDs should use the `Rule` object form (`Rule::unique(...)`, `Rule::exists(...)`) rather than string form — string form injects values into a parsed rule string, object form doesn't.

Custom rule objects: `php artisan make:rule Uppercase` → `passes(string $attribute, mixed $value): bool` + `message()`. Laravel 10 also supports invokable rule objects implementing `Illuminate\Contracts\Validation\ValidationRule`, and inline closures inside the rules array:

```php
use Closure;

$rules = [
    'title' => ['required', function (string $attribute, mixed $value, Closure $fail) {
        if ($value !== strtoupper($value)) {
            $fail('The :attribute must be uppercase.');
        }
    }],
];
```
