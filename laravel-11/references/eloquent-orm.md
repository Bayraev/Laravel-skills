# Laravel 11 Eloquent ORM

Source: https://laravel.com/framework/docs/11.x/eloquent.

## Model conventions

```shell
php artisan make:model Flight
php artisan make:model Flight --migration      # or -m
php artisan make:model Flight -mfsc            # + factory, seeder, controller
php artisan make:model Flight --all            # everything incl. policy + form requests
php artisan model:show Flight                  # inspect columns, casts, relations
```

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    //
}
```

Defaults: snake_case plural table (`Flight` → `flights`), `id` primary key, `created_at` /
`updated_at` timestamps maintained automatically. Override only when you deviate:
`protected $table`, `protected $primaryKey`, `public $timestamps = false`,
`protected $connection`.

## The Laravel 11 `casts()` method

New in 11 — casts as a method instead of a property, which makes casts with arguments clean:

```php
use Illuminate\Database\Eloquent\Casts\AsCollection;

protected function casts(): array
{
    return [
        'options' => AsCollection::using(OptionCollection::class),
        'published_at' => 'datetime',
        'is_active' => 'boolean',
        'tags' => 'array',
        'status' => FlightStatus::class,           // enum cast
        'secret' => 'encrypted',                   // encrypted cast
    ];
}
```

The `$casts` property still works (upgraded apps keep it); prefer the method in new code.
`php artisan model:show` displays resolved casts.

## UUID and ULID keys

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Concerns\HasUlids;

class Article extends Model
{
    use HasUuids;    // ordered UUIDs; pair with $table->uuid('id') in the migration
    // use HasUlids; // 26-char sortable id; pair with $table->ulid('id')
}
```

## Mass assignment

```php
class Flight extends Model
{
    protected $fillable = ['name', 'airline'];

    // or block-list style:
    // protected $guarded = [];
}
```

`$guarded = []` allows mass-assigning anything — only use it when every array passed to
`create`/`fill`/`update` is hand-built in your code. Never pass `$request->all()` to a model with
an empty `$guarded`. JSON columns need each key listed: `'options->enabled'`.

Make silent discarding loud during development — put this in `AppServiceProvider::boot()`:

```php
Model::preventLazyLoading(! $this->app->isProduction());
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

## Retrieving models

```php
use App\Models\Flight;

$flights = Flight::all();
$flights = Flight::where('active', 1)->orderBy('name')->take(10)->get();

$flight = Flight::find(1);
$flight = Flight::findOrFail(1);                    // 404 on miss — use in routes
$flight = Flight::where('active', 1)->firstOrFail();

// First-or-create...
$flight = Flight::firstOrCreate(['name' => 'London to Paris']);
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1],
);

// Aggregates...
$count = Flight::where('active', 1)->count();
$max = Flight::where('active', 1)->max('price');
```

Large datasets — avoid `all()`/`get()` memory blowups:

```php
Flight::chunk(200, fn (Collection $flights) => /* ... */);

// Use chunkById when filtering by a column you also update:
Flight::where('departed', true)
    ->chunkById(200, fn (Collection $flights) => $flights->each->update(['departed' => false]));

foreach (Flight::lazy() as $flight) { /* streams chunk-by-chunk */ }
foreach (Flight::cursor() as $flight) { /* one query, hydrate-on-iterate */ }
```

## Inserting / updating / deleting

```php
$flight = Flight::create(['name' => 'London to Paris']);   // requires $fillable

$flight = new Flight;
$flight->name = 'Paris to London';
$flight->save();

Flight::where('active', 1)->update(['delayed' => 1]);      // mass update: no model events

$flight->delete();
Flight::destroy(1, 2, 3);
```

Change detection: `isDirty()`, `isClean()`, `wasChanged()`, `getOriginal('name')`.
Mass updates skip `saving`/`saved`/`updating`/`updated` events — if events must fire, retrieve
then save each model.

Upserts in one atomic query:

```php
Flight::upsert(
    [
        ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
        ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150],
    ],
    uniqueBy: ['departure', 'destination'],
    update: ['price'],
);
```

## Soft deletes

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Flight extends Model
{
    use SoftDeletes;    // needs a deleted_at column: $table->softDeletes();
}
```

`restore()` brings records back; `forceDelete()` removes permanently; queries exclude trashed
records unless you use `withTrashed()` / `onlyTrashed()`.

## Relationships (definition cheat-sheet)

```php
// One to many / inverse
public function comments(): HasMany
{
    return $this->hasMany(Comment::class);
}

public function post(): BelongsTo
{
    return $this->belongsTo(Post::class);
}

// Many to many
public function roles(): BelongsToMany
{
    return $this->belongsToMany(Role::class);
}

// Has one of many
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

Always eager load in loops — the N+1 problem is the #1 Eloquent performance bug:

```php
// BAD: one query per book for ->author
foreach (Book::all() as $book) { echo $book->author->name; }

// GOOD
$books = Book::with('author')->get();
$books = Book::with(['author', 'comments' => fn ($q) => $q->where('approved', true)])->get();
```

`Model::preventLazyLoading(! $this->app->isProduction())` turns accidental lazy loads into
exceptions so they surface in dev, not in production load.

## Query scopes

```php
// Local scope on the model...
public function scopeActive(Builder $query): void
{
    $query->where('active', 1);
}

Flight::active()->orderBy('name')->get();
```

## Events and observers

```php
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy(FlightObserver::class)]
class Flight extends Model
{
    //
}

class FlightObserver
{
    public function created(Flight $flight): void
    {
        // ...
    }
}
```

Lifecycle hooks: `retrieved`, `creating`, `created`, `updating`, `updated`, `saving`, `saved`,
`deleting`, `deleted`, `restoring`, `restored`, `forceDeleted`. Event discovery is automatic in
Laravel 11 — `Event::listen(FlightCreated::class, ...)` needs no manual registration, and
observers like the above work without listing them in a provider.

## Schema inspection (new in 11)

```php
use Illuminate\Support\Facades\Schema;

$tables = Schema::getTables();
$columns = Schema::getColumns('users');
$indexes = Schema::getIndexes('users');
$foreignKeys = Schema::getForeignKeys('users');
```

Useful for admin tooling, drift checks, and writing introspection-based features without parsing
raw database information_schema queries.
