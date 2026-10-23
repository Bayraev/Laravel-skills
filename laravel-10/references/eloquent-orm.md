# Eloquent ORM

Source: [eloquent](https://laravel.com/framework/docs/10.x/eloquent), [eloquent-mutators](https://laravel.com/framework/docs/10.x/eloquent-mutators), [eloquent-collections](https://laravel.com/framework/docs/10.x/eloquent-collections). Laravel 10 casting uses the **`$casts` property** — the `casts()` method seen in Laravel 11+ docs does not exist here.

## Model basics

```bash
php artisan make:model Flight
php artisan make:model Flight -m          # + migration
php artisan make:model Flight -a          # all: migration, factory, seeder, policy, controller, form requests
php artisan make:model Membership -p      # pivot model
```

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    use HasFactory;

    protected $fillable = ['name', 'airline'];   // or $guarded (see mass assignment)

    protected $casts = [                         // property, not casts(): method
        'options' => 'array',
        'deployed_at' => 'datetime',
        'metadata' => 'collection',
        'status' => FlightStatus::class,         // enum cast (backed enums)
    ];
}
```

Conventions: table = snake-case plural (`Flight` → `flights`); PK = `id`; timestamps `created_at`/`updated_at` auto-managed. Override via `$table`, `$primaryKey`, `$keyType`, `public $incrementing = false`, `public $timestamps = false`, `const CREATED_AT / UPDATED_AT`.

**UUID/ULID keys** — traits swap auto-increment for ordered UUIDs/ULIDs:

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;   // or HasUlids

class Article extends Model
{
    use HasUuids;
    // migration: $table->uuid('id')->primary();
}
```

## Mass assignment

```php
$flight = Flight::create(['name' => 'BA001', 'airline' => 'BA']);
$flight->fill([...])->save();
```

Use `$fillable` (allow-list) or `$guarded` (deny-list, `['*']` blocks all — safest default). Mass-assignment exceptions are silent by default; surface them in development:

```php
// AppServiceProvider::boot()
use Illuminate\Database\Eloquent\Model;

Model::preventLazyLoading(! $this->app->isProduction());
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

## Querying

```php
use App\Models\Flight;

$flights = Flight::all();
$flights = Flight::where('active', true)->orderBy('name')->get();
$flights = Flight::where('active', true)->cursorPaginate(15);   // efficient on large sets

$flight = Flight::find(1);                          // null if missing
$flight = Flight::findOrFail(1);                    // 404
$flight = Flight::where('slug', $slug)->firstOrFail();

Flight::where('dest', 'LHR')->chunk(200, fn (Collection $flights) => /* ... */);
Flight::where('dest', 'LHR')->lazy()->each(fn (Flight $f) => /* ... */);  // memory-safe iteration

$count = Flight::where('active', true)->count();
Flight::where('active', true)->increment('seats'); // decrement, increment('x', 5, ['field' => val])

// Retrieve or create (returns [model, bool])
[$flight, $created] = Flight::firstOrCreate(['number' => 'BA001'], ['airline' => 'BA']);
[$flight, $created] = Flight::updateOrCreate(['number' => 'BA001'], ['airline' => 'BA']);
```

Load relationships without N+1 — `with()` for eager loading, `load()` after the fact, `withCount()` for counts:

```php
$posts = Post::with('author', 'comments')->latest()->get();
$posts = Post::withCount('comments')->get();   // ->comments_count
$post->loadMissing('author');
```

`Model::preventLazyLoading()` (above) turns accidental lazy loads into exceptions in dev — the cheapest way to catch N+1 before production does.

## Relationships

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;

class Post extends Model
{
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);      // pivot `post_role` by convention
    }

    public function latestComment(): HasOne
    {
        return $this->hasOne(Comment::class)->latestOfMany();
    }
}
```

Relationship querying through dynamic properties:

```php
$post->comments;                          // Collection (cached)
$post->comments()->where('approved', 1)->get();
$posts = Post::has('comments')->get();    // / orDoesntHave, withCount, whereHas('comments', fn ($q) => ...)
```

N+1 remediation is relationship hygiene: pass `with()` at the query, never loop queries. Collections of models (`Illuminate\Database\Eloquent\Collection`) support `load()` (eager-load on a collection in one query), `contains`, `find`, `fresh`, map/filter with model awareness.

## Insert / update / delete

```php
$flight = new Flight;
$flight->name = 'BA001';
$flight->save();

$flight = Flight::create(['name' => 'BA001']);        // mass create + save
$flight->update(['name' => 'BA002']);                 // mass update on instance
Flight::where('active', false)->update(['archived' => true]);  // mass query update (no events, no timestamps)

$flight->delete();                                    // fires model events
Flight::destroy([1, 2, 3]);
Flight::where('active', false)->delete();             // no model events

// Soft deletes
use Illuminate\Database\Eloquent\SoftDeletes;

class Flight extends Model
{
    use SoftDeletes;   // migration: $table->softDeletes();
}
Flight::withTrashed()->find(1)->restore();
Flight::onlyTrashed()->get();
```

`update()`/`delete()` via query builder skip model events and timestamps — reach for the instance form when hooks matter.

## Accessors / mutators (Attribute API)

Laravel 9+ (and 10) uses one `Attribute` class instead of separate `getFooAttribute` methods:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    protected function name(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => strtoupper($value),
            set: fn (string $value) => strtolower($value),
        );
    }

    // Multi-column value object
    protected function address(): Attribute
    {
        return Attribute::get(fn () => new Address($this->line1, $this->city));
    }
}
```

## Query builder (when not model-shaped)

For aggregate reports, cross-table updates, or tables without models, use `DB` directly — same builder under Eloquent's hood:

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->select('name', 'email')
    ->where('active', true)
    ->whereBetween('created_at', [$start, $end])
    ->whereIn('team_id', $teamIds)
    ->whereNull('deleted_at')
    ->orderBy('name')
    ->limit(50)
    ->get();                       // Collection of stdClass

$user = DB::table('users')->where('email', $email)->first();   // stdClass or null
$count = DB::table('users')->where('active', true)->count();

DB::table('sessions')->where('last_activity', '<', $cutoff)->delete();
DB::table('users')->increment('login_count');

// Joins
$photos = DB::table('photos')
    ->join('users', 'users.id', '=', 'photos.user_id')
    ->select('photos.*', 'users.name as user_name')
    ->get();

// Raw expressions when the builder has no equivalent
DB::table('orders')
    ->select(DB::raw('SUM(total) as revenue'), DB::raw('COUNT(*) as orders'))
    ->whereRaw('created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)')
    ->first();

// Transactions — wrap multi-statement consistency
DB::transaction(function () use ($data) {
    $order = Order::create($data);
    $order->items()->createMany($data['items']);
}, 3);                             // retry 3x on deadlock

DB::beginTransaction();
try {
    /* ... */
    DB::commit();
} catch (Throwable $e) {
    DB::rollBack();
    throw $e;
}
```

Raw SQL when needed: `DB::select('select * from users where active = ?', [1])` — always bindings, never interpolated values (SQL injection).

## Events, observers, scopes

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class Flight extends Model
{
    // Lifecycle hooks on the model
    protected static function booted(): void
    {
        static::creating(function (Flight $flight) {
            $flight->uuid ??= (string) Str::uuid();
        });
    }

    // Local scope — called as Flight::active()
    public function scopeActive(Builder $query): void
    {
        $query->where('active', true);
    }
}
```

```php
// Observer: php artisan make:observer FlightObserver --model=Flight
// Registered automatically by discovery (models/observers convention)
class FlightObserver
{
    public function created(Flight $flight): void
    {
        // post-creation side effects
    }
}
```
