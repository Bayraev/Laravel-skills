# Eloquent ORM (Models, Attributes, Resources)

## Model basics — the 13 attribute style

```bash
php artisan make:model Flight
php artisan make:model Flight -mfsc      # + migration, factory, seeder, controller
php artisan model:show Flight            # inspect attributes, relations, casts
```

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Attributes\WithoutTimestamps;
use Illuminate\Database\Eloquent\Model;

#[Table('my_flights')]                       // table name (default: snake plural of class)
#[Table(key: 'flight_id')]                   // primary key (default: id)
#[Table(key: 'uuid', keyType: 'string', incrementing: false)]  // non-incrementing / string keys
#[Table(timestamps: false)]                  // disable created_at/updated_at management
#[Table(dateFormat: 'U')]                    // custom timestamp storage format
class Flight extends Model
{
    //
}
```

Single-purpose attributes: `#[WithoutIncrementing]`, `#[WithoutTimestamps]`, `#[DateFormat('U')]`. The classic protected properties (`$table`, `$primaryKey`, `$keyType`, `$incrementing`, `$timestamps`, `$dateFormat`) still work — attributes are the documented style in 13. **`#[Table(...)]` arguments compose**: use one attribute with named args for multiple settings.

Database connections: `protected $connection = 'pgsql';` (no attribute equivalent needed often).

## Retrieving

```php
use App\Models\Flight;

$flights = Flight::all();
$flights = Flight::where('active', true)->orderBy('name')->limit(10)->get();
$flights = Flight::where('destination', 'Paris')->chunk(200, fn ($flights) => /* ... */);
$active = Flight::lazy()->filter(fn ($f) => $f->active);   // LazyCollection

$flight = Flight::findOrFail(1);                // 404 on miss
$flight = Flight::where('slug', $slug)->firstOrFail();
$flight = Flight::firstOr(fn () => /* default */);
$count = Flight::where('active', true)->count();

Flight::where('active', true)->each(fn ($f) => $f->update(['status' => 'archived']));
Flight::where('active', true)->update(['status' => 'archived']);  // mass update: no model events, no updated_at — loop when events/timestamps matter
```

**N+1 discipline:** always `with()` relationships used in a loop or resource — `Flight::with('airline')->get()`, or `->load()` after the fact; `withCount('passengers')` for counts.

## Inserting / updating

```php
$flight = Flight::create($request->validated());       // mass assignment — needs $fillable
$flight = Flight::createOrFirst(['email' => $e], ['name' => $n]);
$flight = Flight::updateOrCreate(['email' => $e], ['name' => $n]);
$flight->fill($attributes)->save();
$flight->increment('seats_available');
```

```php
class Flight extends Model
{
    protected $fillable = ['name', 'airline'];   // or protected $guarded = ['id'];
    // Guarding rule: list fillable fields explicitly; never use guarded = [] on untrusted input
}
```

## Casts, accessors, mutators

```php
protected function casts(): array
{
    return [
        'options' => AsArrayObject::class,        // array with object semantics
        'delivered_at' => 'datetime',
        'meta' => 'collection',
        'amount' => 'decimal:2',
        'tags' => AsCollection::class,
        'status' => FlightStatus::class,          // enum cast
        'secret' => 'hashed',                     // hashed on set (Laravel 10+)
        'synced_at' => 'immutable_datetime',
    ];
}
```

Attribute accessors/mutators use the single-method syntax:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function name(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => ucfirst($value),
        set: fn (string $value) => strtolower($value),
    );
}
```

## Relationships (stable API)

```php
public function airline()        { return $this->belongsTo(Airline::class); }
public function passengers()     { return $this->hasMany(Passenger::class); }
public function tags()           { return $this->morphToMany(Tag::class, 'taggable'); }
public function latestPassenger(){ return $this->hasOne(Passenger::class)->latestOfMany(); }
```

Query via relationship: `$airline->flights()->where('active', true)->count()`; eager load with constraints: `Flight::with(['passengers' => fn ($q) => $q->where('vip', true)])`. Save through relationships: `$flight->passengers()->create([...])` sets the FK automatically. Pivot data: `->attach($id, ['seat' => '12A'])`, `withPivot()`, `withTimestamps()`.

## Soft deletes

`use Illuminate\Database\Eloquent\SoftDeletes;` on the model + `softDeletes()` in the migration. Query includes trashed-exclusion by default; `withTrashed()`, `onlyTrashed()`, `$model->restore()`, `forceDelete()`.

## Events, scopes, pruning

```php
// Closure observers (lightweight, in a service provider boot)
Flight::created(fn (Flight $flight) => /* ... */);

// Observer classes: php artisan make:observer FlightObserver --model=Flight
class FlightObserver
{
    public function created(Flight $flight): void { /* ... */ }
}

// Local scopes
public function scopeActive(Builder $query): Builder
{
    return $query->where('active', true);
}
Flight::active()->get();
```

Pruning (regular cleanup of stale rows): `Prunable` trait + `prunable()` returning the query to remove; `$model->prunable()` runs via `php artisan model:prune`.

## API resources

```bash
php artisan make:resource UserResource
php artisan make:resource UserCollection
```

```php
class UserResource extends JsonResource
{
    /** @return array<string, mixed> */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'posts' => PostResource::collection($this->whenLoaded('posts')),  // avoid N+1 + conditional
            'secret' => $this->when($request->user()->isAdmin(), 'secret-value'),
            'posts_count' => $this->whenCounted('posts'),
            'created_at' => $this->created_at,
        ];
    }
}
```

The 13 convention shortcuts:

```php
return $user->toResource();                        // finds UserResource by convention
return User::all()->toResourceCollection();        // finds UserCollection
return User::paginate()->toResourceCollection();   // paginated: adds meta/links

// Convention overrides:
#[UseResource(CustomUserResource::class)]              // on the model
#[UseResourceCollection(CustomUserCollection::class)]  // on the model
#[Collects(Member::class)]                             // on a collection resource: what it collects
#[PreserveKeys]                                        // keep collection keys instead of reindexing
```

Data wrapping: outermost resources are wrapped in `data` unless `JsonResource::withoutWrapping()` is called (e.g. in `AppServiceProvider::boot()`).

## JSON:API resources (new in 13)

```bash
php artisan make:resource PostResource --json-api
```

```php
use Illuminate\Http\Resources\JsonApi\JsonApiResource;

class PostResource extends JsonApiResource
{
    public $attributes = ['title', 'body', 'created_at'];          // read straight from the model

    public $relationships = ['author', 'comments'];                // serialized only when ?include=...
}
```

- Override `toAttributes(Request $request): array` for computed attributes (use closures for lazy/ expensive values: `'is_published' => fn () => $this->published_at !== null`).
- Override `toRelationships(Request $request): array` for explicit resource classes or constrained payloads.
- Clients request relations with `GET /api/posts/1?include=author,comments` → identifier objects under `relationships`, full objects in top-level `included`; sparse fieldsets via `fields[posts]=title`.
- Sets `Content-Type: application/vnd.api+json`; handles `id`/`type` automatically.

**Choose plain `JsonResource` for bespoke/internal JSON shapes; choose `JsonApiResource` when clients expect spec-compliant JSON:API payloads.**

## Vector search on Eloquent

Vector query-builder clauses work through Eloquent models too (PostgreSQL + pgvector, or MariaDB 11.7+):

```php
$posts = Post::whereVectorSimilarTo('embedding', 'best wineries in Napa', minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

See `migrations-seeding.md` for `vector()` columns and `ai-tooling.md` for embedding generation.
