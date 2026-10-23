# Migrations, Factories, and Seeding

Source: [migrations](https://laravel.com/framework/docs/10.x/migrations), [seeding](https://laravel.com/framework/docs/10.x/seeding), [eloquent-factories](https://laravel.com/framework/docs/10.x/eloquent-factories).

**Version-critical:** on Laravel 10, column modification (`->change()`), `dropColumn`, and `renameIndex` require **`doctrine/dbal`** on SQLite (and on older MySQL/PostgreSQL for renames). Laravel 11 made `change()` native — that does not apply here. `composer require doctrine/dbal` before writing any `->change()` call.

## Generating and structuring migrations

```bash
php artisan make:migration create_flights_table
php artisan make:migration add_status_to_flights_table --table=flights
php artisan make:migration create_users_table --create=users
```

Laravel 10 migrations are anonymous classes with typed methods:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('flights', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('airline');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('flights');
    }
};
```

Rules of the road:

- **Never edit a migration that has run anywhere shared.** Add a new migration instead. In disposable dev databases, editing + `migrate:fresh` is acceptable.
- One logical change per migration; name it after the change (`add_status_to_flights_table`).
- Always write `down()` so `rollback` and `migrate:fresh` stay honest (irreversible drops: `down()` left empty or throwing is a deliberate choice — comment it).

## Schema builder essentials

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();                          // auto-increment bigint PK
    $table->ulid('ref')->nullable();       // uuid('ref') also available
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->foreignId('team_id')->constrained()->cascadeOnDelete();  // FK shorthand
    $table->timestamps();                  // created_at + updated_at
});

Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->string('title', 200);
    $table->text('body');
    $table->enum('status', ['draft', 'published'])->default('draft');
    $table->json('meta')->nullable();
    $table->decimal('price', 10, 2)->default(0);
    $table->boolean('active')->default(true);
    $table->softDeletes();                 // deleted_at for SoftDeletes
    $table->timestamps();

    $table->index(['status', 'published_at']);
    $table->unique(['user_id', 'title']);
});
```

Common column helpers: `$table->foreignId(...)->constrained()`, `->nullable()`, `->default(...)`, `->comment(...)`, `->after('column')` (MySQL), `->first()`. Index methods: `index`, `unique`, `primary` (each with a name argument when needed), `dropIndex`, `spatialIndex`. Foreign keys: `foreignId` + `constrained` (or explicit `foreign('user_id')->references('id')->on('users')`), with `cascadeOnDelete()` / `nullOnDelete()` / `restrictOnDelete()`.

Modify columns (dbal required — see note at top):

```php
use Illuminate\Support\Facades\Schema;

Schema::table('flights', function (Blueprint $table) {
    $table->string('name', 100)->nullable()->change();
    $table->renameColumn('from', 'origin');       // dbal needed on older DBs
    $table->dropColumn('airline');
});
```

Renaming a table: `Schema::rename($from, $to)`. Check first when writing idempotent code: `Schema::hasTable('flights')`, `Schema::hasColumn('flights', 'airline')`.

## Running migrations

```bash
php artisan migrate                       # run outstanding
php artisan migrate --force               # production (bypasses the confirm prompt)
php artisan migrate:status                # what has run
php artisan migrate:rollback              # last batch
php artisan migrate:rollback --step=2
php artisan migrate:fresh                 # drop all tables, re-run everything (DESTRUCTIVE)
php artisan migrate:fresh --seed          # ...and seed
php artisan migrate:refresh               # rollback all + re-run (keep data if no destructive down)
php artisan schema:dump --prune           # squash history into database/schema/*.sql (MySQL/Postgres/SQLite)
```

`migrate:fresh` vs `refresh`: `fresh` drops tables (loses data even without `down()`); `refresh` replays `down()`. In dev, `migrate:fresh --seed` is the standard reset. In production, prefer forward-only migrations and `--pretend` to preview SQL: `php artisan migrate --pretend`.

Squashed schema files live in `database/schema/` and must be committed; they run before remaining migrations on a fresh database (also useful for the test DB — dump it separately with `--database=testing` if tests use another connection).

## Model factories

```bash
php artisan make:factory FlightFactory --model=Flight
```

```php
namespace Database\Factories;

use App\Models\Flight;
use Illuminate\Database\Eloquent\Factories\Factory;

class FlightFactory extends Factory
{
    protected $model = Flight::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->numerify('BA-####'),
            'airline' => fake()->randomElement(['BA', 'LH', 'AF']),
        ];
    }

    public function cancelled(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'cancelled',
        ]);
    }
}
```

Usage (factories create consistent test/seed data; `fake()` is the global Faker helper):

```php
Flight::factory()->create();                     // persisted
Flight::factory()->count(50)->create();
Flight::factory()->cancelled()->create();
Flight::factory()->make();                       // in-memory only
Flight::factory()->for(User::factory())->create();       // belongsTo
Flight::factory()->has(Comment::factory()->count(3))->create();  // hasMany
```

## Seeders

```bash
php artisan make:seeder FlightSeeder
```

```php
namespace Database\Seeders;

use App\Models\Flight;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            UserSeeder::class,
            FlightSeeder::class,
        ]);

        // or inline
        Flight::factory()->count(10)->create();
    }
}
```

```bash
php artisan db:seed                          # DatabaseSeeder
php artisan db:seed --class=FlightSeeder
php artisan db:seed --force                  # production (destructive prompt bypass)
php artisan migrate:fresh --seed             # full rebuild + seed
```

Idempotency matters: guard against double-seeding with `firstOrCreate`/`updateOrCreate` in seeders, and mute model events when a seeder must not trigger hooks (`Model::withoutEvents(fn () => ...)` or the `SilencesModelEvents` trait). Never commit real user data in seeders — factories and fake data only.
