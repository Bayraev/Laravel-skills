# Migrations and Seeding (Laravel 12)

## Table of contents

- [Creating and running migrations](#creating-and-running-migrations)
- [Schema builder basics](#schema-builder-basics)
- [Column types you'll actually use](#column-types-youll-actually-use)
- [Indexes and foreign keys](#indexes-and-foreign-keys)
- [Modifying and dropping](#modifying-and-dropping)
- [Inspecting schema (12.x notes)](#inspecting-schema-12x-notes)
- [Seeders](#seeders)
- [Model factories](#model-factories)

## Creating and running migrations

```shell
php artisan make:migration create_flights_table
php artisan make:migration add_pilot_column_to_flights_table
php artisan make:migration create_categories_table --create=categories
php artisan make:migration add_status_to_posts_table --table=posts

php artisan migrate                  # run outstanding migrations
php artisan migrate --pretend        # print SQL without executing — check before you commit
php artisan migrate:rollback         # undo the last batch
php artisan migrate:fresh --seed     # drop everything, rebuild, reseed (dev/test only)
php artisan migrate:status
```

Migrations are **anonymous classes** (the 12.x stub — no class name collisions when two migrations share a name):

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('flights', function (Blueprint $table): void {
            $table->id();
            $table->string('name');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('flights');
    }
};
```

Working habits that save production incidents: make every migration reversible (a real `down()`); keep one logical change per migration; run `--pretend` before running for real against anything shared; and never edit a migration that has shipped — write a new one.

## Schema builder basics

Common patterns with the vocabulary you need:

```php
Schema::create('posts', function (Blueprint $table): void {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->ulid('ulid')->unique();              // or $table->uuid('uuid')->unique();
    $table->string('title', 160);
    $table->string('slug')->unique();
    $table->text('body');
    $table->enum('status', ['draft', 'published', 'archived'])->default('draft');
    $table->json('meta')->nullable();
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
    $table->softDeletes();
});

Schema::create('post_tag', function (Blueprint $table): void {
    $table->id();
    $table->foreignId('post_id')->constrained()->cascadeOnDelete();
    $table->foreignId('tag_id')->constrained()->cascadeOnDelete();
    $table->timestamps();
});
```

Why `foreignId()->constrained()` over the old `unsignedInteger` + `foreign()` pair: it derives the referenced table from the column name, keeps the rule next to the column, and reads as intent. `->cascadeOnDelete()` / `->restrictOnDelete()` / `->nullOnDelete()` state the deletion contract explicitly.

Default `id()` is a big-incrementing integer. For UUID/ULID models (`eloquent-orm.md`), add `$table->uuid('id')` or `$table->ulid('id')` **instead of** `id()` — the `HasUuids`/`HasUlids` trait fills the value, the column just must not auto-increment.

## Column types you'll actually use

`id()`, `foreignId()`, `uuid()`, `ulid()`, `string('name', 255)`, `char()`, `text()`, `longText()`, `integer()`, `tinyInteger()`, `bigInteger()`, `unsignedBigInteger()`, `decimal('amount', 10, 2)`, `boolean()`, `date()`, `dateTime()`, `timestamp()`, `timestamps()`, `softDeletes()`, `json()`, `jsonb()`, `enum()`, `set()`, `ipAddress()`, `macAddress()`, `rememberToken()`, `morphs('taggable')` (adds `taggable_type` + `taggable_id` with index — for polymorphic relations).

Modifiers chain: `->nullable()`, `->default($value)`, `->comment('...')`, `->after('column')`, `->unsigned()`, `->useCurrent()`.

## Indexes and foreign keys

```php
$table->index('email');                       // ->index(['tenant_id', 'status']) for compound
$table->unique('slug');
$table->fullText('body');                     // MySQL 5.7+ / Postgres
$table->spatialIndex('location');

// Detached from the column, when the column predates the index:
$table->foreign('user_id')->references('id')->on('users')->cascadeOnDelete();
```

Naming is automatic (`posts_user_id_index`) and predictable — pass a custom name as the second argument only when collisions force it. Rename indexes with `$table->renameIndex('from', 'to')`.

## Modifying and dropping

```php
Schema::table('flights', function (Blueprint $table): void {
    $table->string('name', 100)->change();      // ->change() needs doctrine/dbal on some drivers? — no:
});
```

Since Laravel 11, `->change()` works natively for most column types without `doctrine/dbal`. Drop with `dropColumn('name')` (or multiple: `dropColumn(['a', 'b'])`), `dropIndex('flights_email_index')`, `dropUnique('flights_slug_unique')`, `dropForeign(['user_id'])`, `dropTimestamps()`, `dropSoftDeletes()`, `renameColumn('from', 'to')`, and `Schema::rename($from, $to)` / `Schema::drop('flights')` / `Schema::dropIfExists('flights')` at the table level.

Caution: dropping or altering columns is driver-sensitive (SQLite historically ignores/limits some operations). Verify with `migrate --pretend` on the production driver, not just locally.

## Inspecting schema (12.x notes)

```php
use Illuminate\Support\Facades\Schema;

Schema::hasTable('flights');
Schema::hasColumn('flights', 'name');

// 12.x: multi-schema by default — scope explicitly when needed:
$tables = Schema::getTables();                        // ALL schemas
$tables = Schema::getTables(schema: 'main');          // one schema
$tables = Schema::getTables(schema: ['main', 'blog']);// several
```

In 12, `Schema::getTables()`, `getViews()`, `getTypes()` return **all schemas** by default, and `Schema::getTableListing()` returns **schema-qualified** names (`main.users`) — pass `schemaQualified: false` for bare names. (Verified: https://laravel.com/docs/12.x/upgrade#multi-schema-database-inspecting.) The `db:show` and `db:table` Artisan commands reflect all schemas on MySQL/MariaDB/SQLite too.

## Seeders

```shell
php artisan make:seeder FlightSeeder
```

```php
<?php

namespace Database\Seeders;

use App\Models\Flight;
use Illuminate\Database\Seeder;

class FlightSeeder extends Seeder
{
    /**
     * Run the database seeds.
     */
    public function run(): void
    {
        Flight::factory()->count(50)->create();

        Flight::create(['name' => 'London to Paris']);
    }
}
```

`DatabaseSeeder` orchestrates; call other seeders with `$this->call(FlightSeeder::class);` or `$this->call([FlightSeeder::class, UserSeeder::class]);`. Run with `php artisan db:seed` (or `php artisan migrate:fresh --seed` for a clean rebuild). Gate environment-specific seeding:

```php
public function run(): void
{
    if (app()->environment('local', 'testing')) {
        $this->call(DemoDataSeeder::class);
    }
}
```

Production seeding should be idempotent or guarded — `db:seed` on prod with destructive demo data is a classic bad afternoon.

## Model factories

```shell
php artisan make:factory FlightFactory --model=Flight
```

```php
<?php

namespace Database\Factories;

use App\Models\Flight;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Flight>
 */
class FlightFactory extends Factory
{
    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'name' => fake()->company(),
            'published_at' => fake()->optional()->dateTimeThisDecade(),
        ];
    }

    public function delayed(): static
    {
        return $this->state(fn (array $attributes) => ['delayed' => true]);
    }
}
```

Usage: `Flight::factory()->count(50)->delayed()->create()`, `->make()` for unsaved instances, `->create(['name' => 'override'])` for targeted overrides, `->for(User::factory())` for parent relations, `->has(Bookings::factory()->count(3))` for children. `fake()` (Faker) is available inside factories and seeders. Models need `HasFactory` to find their factory by convention (`eloquent-orm.md`).
