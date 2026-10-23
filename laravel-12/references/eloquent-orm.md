# Eloquent ORM (Laravel 12)

## Table of contents

- [Model conventions](#model-conventions)
- [UUID, ULID, and key types (12.x: UUIDv7)](#uuid-ulid-and-key-types-12x-uuidv7)
- [Attribute casting](#attribute-casting)
- [Relationships](#relationships)
- [Querying and eager loading](#querying-and-eager-loading)
- [Inserts, updates, mass assignment](#inserts-updates-mass-assignment)
- [Events, observers, and hooks](#events-observers-and-hooks)
- [Soft deletes and pruning](#soft-deletes-and-pruning)
- [Strictness and performance](#strictness-and-performance)

## Model conventions

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    //
}
```

Conventions you get for free (and should not fight): snake-case plural table names (`AirTrafficController` → `air_traffic_controllers`), `id` auto-incrementing integer primary key, `created_at`/`updated_at` timestamps. Override only when the database forces it:

```php
protected $table = 'my_flights';
protected $primaryKey = 'flight_id';
public $incrementing = false;
protected $keyType = 'string';
public $timestamps = false;
protected $connection = 'mysql';
public const CREATED_AT = 'creation_date';
public const UPDATED_AT = 'updated_date';
```

Composite primary keys are **not** supported — model one unique identifying key, and enforce additional uniqueness with unique indexes in migrations.

Generate with everything you'll need in one shot:

```shell
php artisan make:model Flight --all    # model + migration + factory + seeder + policy + controller + form requests
php artisan make:model Flight -mfsc    # the same, shorthand
php artisan model:show Flight          # inspect attributes, casts, relations
```

## UUID, ULID, and key types (12.x: UUIDv7)

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class Article extends Model
{
    use HasUuids;
}

$article = Article::create(['title' => 'Traveling to Europe']);
$article->id; // "018f2b5c-6a7f-7b12-9d6f-2f8a4e0c9c11" — UUIDv7
```

**In Laravel 12, `HasUuids` generates UUIDv7** (time-ordered, lexicographically sortable, index-friendly) — verified at https://laravel.com/docs/12.x/eloquent#uuid-and-ulid-keys. Migrating a model that relied on 11's UUIDv4? Use `Illuminate\Database\Eloquent\Concerns\HasVersion4Uuids`. For 26-character ULIDs use `HasUlids`. Both traits require a matching `uuid()`/`ulid()` primary key column in the migration. Customize generation per model:

```php
use Ramsey\Uuid\Uuid;

public function newUniqueId(): string
{
    return (string) Uuid::uuid4();
}

public function uniqueIds(): array
{
    return ['id', 'discount_code'];   // more columns can get unique IDs
}
```

## Attribute casting

Declare casts; never juggle raw strings in app code:

```php
use Illuminate\Database\Eloquent\Casts\AsEnumCollection;
use App\Enums\Status;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    protected function casts(): array
    {
        return [
            'status' => Status::class,
            'tags' => 'array',
            'published_at' => 'datetime',
            'is_featured' => 'boolean',
            'meta' => 'collection',
            'settings' => AsCollection::class,
            'options' => AsArrayObject::class,
            'statuses' => AsEnumCollection::class.':'.Status::class,
        ];
    }
}
```

Laravel 11+ uses a `casts()` **method** (the old `$casts` property still works; prefer the method in new code). Casts run both directions: `'published_at' => 'datetime'` yields a Carbon on read and serializes correctly on write. Custom casts: `php artisan make:cast Money`, implementing `CastsAttributes` (get/set), then reference by class name.

## Relationships

```php
class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    // Custom foreign key / local key as 2nd/3rd args:
    public function phone(): HasOne
    {
        return $this->hasOne(Phone::class, 'user_id', 'id');
    }
}

class Post extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);       // pivot: post_tag
    }

    public function latestComment(): HasOne
    {
        return $this->hasOne(Comment::class)->latestOfMany();
    }
}
```

Patterns that keep relations predictable:

- Default foreign keys (`user_id`) are conventions — omit them until the schema disagrees.
- Name `HasOneOfMany` accessors by what they return (`latestComment`, `oldestOrder`) and constrain explicitly (`latestOfMany()`, `ofMany('price', 'max')`).
- Pivot data: `->withPivot('role')`, `->withTimestamps()`, and `using()` a custom pivot model when the pivot deserves logic.
- Eager-load the inverse side of `BelongsToMany` deliberately; taxonomies accidentally loading N×M rows is a classic production incident.

## Querying and eager loading

```php
use App\Models\Post;

$posts = Post::query()
    ->with(['user', 'comments' => fn ($query) => $query->latest()->limit(5)])
    ->where('published_at', '<=', now())
    ->latest()
    ->paginate(15);

$post = Post::findOrFail($id);          // 404 on miss — right default for routes
$post = Post::firstOrCreate(['slug' => $slug], ['title' => $title]);
$post = Post::updateOrCreate(['slug' => $slug], ['title' => $newTitle]);
```

`findOrFail` in a route/controller context converts `ModelNotFoundException` into a clean 404 — prefer it over null-check-and-abort gymnastics. For large datasets, stream instead of loading everything:

```php
Flight::chunk(200, function (Collection $flights): void {
    $flights->each->process();
});
Flight::chunkById(200, function (Collection $flights): void {  // when also updating the filtered column
    $flights->each->update(['processed' => true]);
});
Flight::lazy()->each(fn (Flight $flight) => $flight->process()); // LazyCollection, one model in memory
```

## Inserts, updates, mass assignment

```php
$flight = Flight::create(['name' => 'London to Paris']);   // create + save, needs $fillable
$flight = new Flight(['name' => '...']);
$flight->save();

$flight->update(['name' => 'Paris to London']);
Flight::where('active', 1)->update(['delayed' => 1]);      // mass update — no model events
$flight->increment('reads');
Flight::destroy([1, 2, 3]);
```

`$fillable` is the allowlist for mass assignment; fill it thoughtfully, and let `preventSilentlyDiscardingAttributes()` (below) catch typos in development. JSON column keys need explicit entries: `$fillable = ['options->enabled']`. `updateOrCreate` returns the model — check `$model->wasRecentlyCreated` when the new-vs-existing distinction matters. Inspect changes with `isDirty()` / `wasChanged()` / `getOriginal()` / `getChanges()`.

## Events, observers, and hooks

Model lifecycle events: `retrieved`, `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored`, `replicating`. Attach listeners in the model's `booted()` closure, or centralize per-model logic in an observer:

```shell
php artisan make:observer PostObserver --model=Post
```

```php
class PostObserver
{
    public function created(Post $post): void
    {
        $post->searchable();   // e.g. sync a search index
    }
}
```

Observers are auto-discovered by convention (`App\Observers\PostObserver` for `App\Models\Post`) — no registration needed in 12. Inside a transaction boundary, `Saved` events may fire before commit; use `Illuminate\Database\TransactionsManager` (`DB::afterCommit(fn () => ...)`) when listeners must observe committed state.

## Soft deletes and pruning

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    protected $fillable = ['title', 'body'];
}
```

Soft-deleted rows are hidden from queries by default; `withTrashed()` includes them, `onlyTrashed()` limits to them, `restore()` revives, `forceDelete()` is permanent. Route binding follows the same rule — chain `->withTrashed()` on the route to bind trashed models.

Pruning (for periodically purging old models) pairs a `prunable()` query with a scheduled `model:prune`:

```php
use Illuminate\Database\Eloquent\Prunable;

class OldPost extends Model
{
    use Prunable;

    public function prunable(): Builder
    {
        return static::where('created_at', '<=', now()->subYears(2));
    }
}
```

Then in `routes/console.php`: `Schedule::command('model:prune')->daily();` (scheduling details in `artisan-cli.md`).

## Strictness and performance

Enable strict mode everywhere except production so mistakes throw during development instead of leaking query load to prod — a high-leverage three lines in `AppServiceProvider::boot()`:

```php
use Illuminate\Database\Eloquent\Model;

Model::preventLazyLoading(! $this->app->isProduction());
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
Model::preventAccessingMissingAttributes(! $this->app->isProduction());
```

- **Lazy loading** throwing forces `with()` at query time — the difference between 101 queries and 1.
- **Silently discarded attributes** throwing turns "why isn't my field saving?" into an immediate exception.
- **Missing attributes** throwing kills `$model->typo_column` bugs.

Scopes keep query intent reusable and reviewable:

```php
public function scopePublished(Builder $query): void
{
    $query->whereNotNull('published_at')
          ->where('published_at', '<=', now());
}

// Usage:
Post::published()->latest()->paginate(15);
```

Global scopes (`protected static function booted() { static::addGlobalScope('tenant', fn (Builder $q) => $q->where('tenant_id', tenant()->id)); }`) apply automatically — reserve them for cross-cutting concerns (multi-tenancy), because hidden query behavior is a debugging tax.
