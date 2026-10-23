# Laravel 11 Migrations and Seeding

Source: https://laravel.com/framework/docs/11.x/migrations, verified against the official
`laravel/laravel` 11.x skeleton (https://github.com/laravel/laravel/tree/11.x/database/migrations).

## The consolidated default migrations

A Laravel 11 app ships exactly THREE migration files (Laravel 10 shipped nine):

- `0001_01_01_000000_create_users_table.php` — `users`, `password_reset_tokens`, `sessions`
- `0001_01_01_000001_create_cache_table.php` — `cache`, `cache_locks`
- `0001_01_01_000002_create_jobs_table.php` — `jobs`, `job_batches`, `failed_jobs`

The `0001_01_01` date prefix orders them before any app migration. These tables back the 11
defaults: SQLite database, `database` session/cache/queue drivers. Add tables by appending new
migrations — never edit the defaults in place once other environments have run them.

## Generating and running migrations

```shell
php artisan make:migration create_flights_table
php artisan make:migration add_status_to_flights_table --table=flights
php artisan make:migration create_flights_table --create=flights

php artisan migrate
php artisan migrate --pretend        # print SQL without running
php artisan migrate --force          # production, skip confirmation
php artisan migrate:status

php artisan migrate:rollback         # last batch
php artisan migrate:rollback --step=5
php artisan migrate:refresh --seed   # rollback all + re-migrate + seed
php artisan migrate:fresh --seed     # DROP everything + re-migrate + seed (careful!)
php artisan migrate --isolated       # atomic lock; safe for multi-server deploys
```

Use `migrate:fresh` in development and `migrate --force` in production deploy scripts.

## Migration structure

Anonymous class migrations (standard since Laravel 9, universal in 11):

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
        Schema::create('flights', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('airline');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::drop('flights');
    }
};
```

## Common column types and modifiers

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('title', 100);
    $table->text('body');
    $table->boolean('published')->default(false);
    $table->integer('votes')->unsigned()->default(0);
    $table->decimal('amount', total: 8, places: 2);
    $table->json('options')->nullable();
    $table->timestamp('published_at')->nullable();
    $table->enum('status', ['draft', 'published']);   // consider a string + enum cast instead
    $table->uuid('uuid')->unique();                   // for HasUuids models
    $table->ulid('ulid')->unique();                   // for HasUlids models
    $table->softDeletes();
    $table->timestamps();
});
```

`foreignId('user_id')->constrained()` beats the verbose
`$table->foreign('user_id')->references('id')->on('users')` — it derives the referenced table by
convention. If the table name doesn't match: `->constrained(table: 'users', indexName: 'posts_user_id')`.

Use `foreignIdFor(User::class)` to respect the model's key type (BIGINT vs UUID vs ULID).

Modifiers worth remembering: `->nullable()`, `->default($value)`, `->comment('...')`,
`->after('column')` (MySQL/MariaDB), `->useCurrent()`, `->storedAs($expr)`, `->virtualAs($expr)`.
Default expressions wrap in `new Expression('(JSON_ARRAY())')` so they aren't quoted as strings.

## Altering tables

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('phone')->nullable();                 // add
    $table->string('name', 50)->change();                // modify
    $table->renameColumn('from', 'to');                  // rename
    $table->dropColumn('votes');                         // drop (array for multiple)
    $table->index('email');
    $table->unique(['account_id', 'created_at']);        // composite index
    $table->fullText('body');                            // MySQL/PostgreSQL
});
```

`change()` requires restating every modifier you want to keep — attributes not repeated are
dropped. It does not touch indexes; manage those separately.

Existence guards for shared/legacy databases: `Schema::hasTable('users')`,
`Schema::hasColumn('users', 'email')`, `Schema::hasIndex('users', ['email'], 'unique')`.

## Squashing migrations

Once the migration folder grows unwieldy:

```shell
php artisan schema:dump
php artisan schema:dump --prune                 # dump + delete the squashed migrations
php artisan schema:dump --database=testing --prune   # also dump for the test connection
```

Writes `database/schema/*.sql`, executed before remaining migrations on fresh installs. Commit it.
MariaDB/MySQL/PostgreSQL/SQLite only (uses each database's CLI client).

## Seeding

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
    public function run(): void
    {
        Flight::factory()->count(50)->create();
    }
}
```

The entry point is `DatabaseSeeder`:

```php
public function run(): void
{
    $this->call([
                UserSeeder::class,
                FlightSeeder::class,
            ]);
}
```

Run with `php artisan db:seed`, or `--class=FlightSeeder` for one seeder.
`php artisan migrate:fresh --seed` is the standard reset-and-seed loop.

## Factories

```shell
php artisan make:factory FlightFactory --model=Flight
```

```php
public function definition(): array
{
    return [
        'name' => fake()->company(),
        'airline' => fake()->randomElement(['Delta', 'United', 'Southwest']),
    ];
}
```

Usage: `Flight::factory()->create(['name' => 'Overridden'])`, `->count(50)->create()`,
`->make()` (unsaved), `->createQuietly()` (no model events). Factories double as test data
builders — see `testing.md`.

## Practical rules

- One concern per migration; name it after what it does (`add_soft_deletes_to_comments_table`).
- Always write `down()` for reversible operations; irreversible drops can throw in `down()`.
- Never edit a migration that has run in a shared environment — append a new one.
- In tests use `RefreshDatabase` (see `testing.md`); it migrates an in-memory SQLite database,
  which Laravel 11 made substantially faster by reusing the PDO connection.
