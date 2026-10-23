# Migrations, Seeding, and Factories

## Creating and running migrations

```bash
php artisan make:migration create_flights_table          # name drives table-name guessing
php artisan make:migration add_status_to_flights_table
php artisan make:migration create_users_table --create=users
php artisan make:migration add_votes_to_users_table --table=users

php artisan migrate
php artisan migrate --pretend          # print SQL without running
php artisan migrate --step             # each migration its own batch (fine-grained rollback)
php artisan migrate --isolated         # atomic lock — multi-server deploy safety
php artisan migrate --force            # production, skip confirmation
php artisan migrate:status
php artisan migrate:rollback --step=5
php artisan migrate:refresh --seed     # rollback all + re-migrate + seed
php artisan migrate:fresh --seed       # DROP all tables + re-migrate + seed (dev/CI staple)
```

**Naming and order:** migrations run in filename timestamp order; never edit a migration that has shipped to other environments — add a new one. Squash bloat with `php artisan schema:dump --prune` (writes `database/schema/*.sql`, MariaDB/MySQL/PostgreSQL/SQLite only; commit the dump).

## Anatomy

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('flights', function (Blueprint $table) {
            $table->id();
            $table->foreignId('airline_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->text('description')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('flights');
    }
};
```

Use a non-default connection via `protected $connection = 'pgsql';`. Skip a migration conditionally with a `shouldRun(): bool` method (e.g. behind a feature flag).

## Column types you'll actually use

`id()` (auto-increment PK), `foreignId('user_id')` / `foreignIdFor(User::class)` / `foreignUuidFor(...)`, `string('name', length: 100)`, `text()`, `longText()`, `boolean()`, `integer()` / `bigInteger()` / `unsignedBigInteger()`, `decimal('amount', total: 8, places: 2)`, `float()`, `date()` / `dateTime()` / `timestamp()`, `timestamps()` / `softDeletes()`, `json()` / `jsonb()`, `uuid()` / `ulid()`, `enum('status', ['a', 'b'])` (or `Enum::cases()`), `ipAddress()`, `rememberToken()`, `morphs('taggable')`, `vector('embedding', dimensions: 1536)`.

Modifiers: `->nullable()`, `->default($value)` (or `new Expression('(JSON_ARRAY())')` for DB functions), `->comment()`, `->after('col')` (MySQL/MariaDB), `->storedAs()` / `->virtualAs()`, `->useCurrent()`, `->unsigned()`, MySQL-specific `->instant()` and `->lock('none'|'shared'|'exclusive')` for fast/low-lock DDL.

## Foreign keys and indexes

```php
$table->foreignId('user_id')->constrained();                       // users.id
$table->foreignId('user_id')->constrained('users')->cascadeOnDelete();
$table->string('email')->unique();
$table->index(['account_id', 'created_at']);
$table->fullText('body');                                          // full-text index (MariaDB/MySQL/PG)
$table->vector('embedding', dimensions: 1536)->index();            // cosine-distance vector index

$table->renameColumn('from', 'to');
$table->dropColumn('votes');
$table->dropIndex(['email']);            // or by name
$table->dropForeign(['user_id']);
```

Name foreign keys explicitly when you plan to rename tables (convention-based names reference the old table). `change()` rewrites a column — specify **every modifier you want to keep**; missing ones are dropped:

```php
$table->string('name', 50)->change();
$table->integer('votes')->unsigned()->default(1)->comment('...')->change();
$table->date('birthday')->using('birthday::date')->change();       // PostgreSQL type cast
```

## Vector columns (semantic search, 13)

```php
use Illuminate\Support\Facades\Schema;

public function up(): void
{
    Schema::ensureVectorExtensionExists();      // loads pgvector (PostgreSQL) — call before creating vector columns

    Schema::create('documents', function (Blueprint $table) {
        $table->id();
        $table->vector('embedding', dimensions: 1536)->index();
        $table->timestamps();
    });
}
```

Supported on PostgreSQL + `pgvector` and MariaDB 11.7+. Query with `whereVectorSimilarTo()` (see `eloquent-orm.md`).

## Seeding

```bash
php artisan make:seeder FlightSeeder
php artisan db:seed                       # runs DatabaseSeeder
php artisan db:seed --class=FlightSeeder
php artisan migrate:fresh --seed          # rebuild + seed (dev)
```

```php
// database/seeders/DatabaseSeeder.php
public function run(): void
{
    $this->call([
                FlightSeeder::class,
                PassengerSeeder::class,
            ]);
}

// database/seeders/FlightSeeder.php
public function run(): void
{
    Flight::factory()->count(50)->create();
}
```

Production caution: guard seeders with `if (app()->environment('production')) { return; }` when they contain test data; seeders are also used for legitimate reference data (roles, countries) — keep the two kinds in separate seeders.

## Factories

```bash
php artisan make:factory FlightFactory --model=Flight
```

```php
class FlightFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name' => fake()->company(),
            'airline_id' => Airline::factory(),
            'departed_at' => now(),
            'status' => 'scheduled',
        ];
    }

    public function cancelled(): static
    {
        return $this->state(fn (array $attributes) => ['status' => 'cancelled']);
    }
}
```

```php
Flight::factory()->create();                       // persist + return model
Flight::factory()->count(50)->create();            // many
Flight::factory()->cancelled()->create();          // named state
Flight::factory()->has(Passenger::factory()->count(3))->create();   // relationships
Flight::factory()->create(['name' => 'override']); // attribute override
Flight::factory()->make();                         // in-memory only (tests)
Flight::factory()->recycle($airline)->create();    // reuse an existing related model
```

Factory states beat ad-hoc `create([...])` overrides for anything reused across tests. In tests, combine with `RefreshDatabase` (see `testing.md`).
