# Laravel 11 Testing

Source: https://laravel.com/framework/docs/11.x/testing (plus queue/HTTP testing sections of
their topic pages).

## Setup and defaults

- `laravel new` prompts for Pest or PHPUnit; both work identically with Laravel's assertions.
- Tests live in `tests/Feature` (booted app, HTTP, DB, everything) and `tests/Unit` (isolated,
  no framework). The Laravel docs' own guidance: **most tests should be feature tests** — they
  give the most confidence the system works.
- Running tests sets the environment to `testing` via `phpunit.xml`; session and cache switch to
  the `array` driver automatically. An optional `.env.testing` overrides `.env` for tests.

```shell
php artisan make:test UserTest              # tests/Feature
php artisan make:test UserTest --unit       # tests/Unit

php artisan test                             # prettier runner
vendor/bin/pest                              # or vendor/bin/phpunit

php artisan test --parallel                  # needs brianium/paratest
php artisan test --coverage --min=80         # needs Xdebug/PCOV
php artisan test --profile                   # list 10 slowest tests
```

## A feature test (Pest style — the 11 default)

```php
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('guests are redirected from the dashboard', function () {
    $response = $this->get('/dashboard');

    $response->assertRedirect('/login');
});

test('users can create posts', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)->post('/posts', [
        'title' => 'My first post',
        'body' => 'Hello world',
    ]);

    $response->assertRedirect(route('posts.index'));
    $this->assertDatabaseHas('posts', ['title' => 'My first post']);
});
```

Same test as a PHPUnit class:

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase;

    public function test_users_can_create_posts(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->post('/posts', [
            'title' => 'My first post',
            'body' => 'Hello world',
        ]);

        $response->assertRedirect(route('posts.index'));
        $this->assertDatabaseHas('posts', ['title' => 'My first post']);
    }
}
```

Call `parent::setUp()` first in custom `setUp()` methods.

## Database in tests

`RefreshDatabase` (the usual choice) runs migrations once then wraps each test in a transaction.
Configure an in-memory SQLite database in `phpunit.xml`:

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

Laravel 11 made in-memory SQLite dramatically faster by reusing the PDO object across
connections — often halving total test time. If tests need features SQLite lacks (e.g. `SKIP
LOCKED`), point `phpunit.xml` at a throwaway MySQL/Postgres database instead.

Factory-based data: `User::factory()->create()`, `->make()` (unsaved),
`->count(20)->create()`, `->state([...])`, and relationship factories like
`Post::factory()->for($user)->create()`.

## HTTP assertions cheat-sheet

```php
$response = $this->get('/api/posts');
$response->assertStatus(200);
$response->assertOk();
$response->assertNotFound();
$response->assertRedirect('/login');
$response->assertJson(['data' => []]);
$response->assertJsonPath('data.0.title', 'First post');
$response->assertJsonCount(3, 'data');
$response->assertViewIs('posts.index');
$response->assertViewHas('posts');
$response->assertSee('Hello world');

// Validation failures...
$response->assertInvalid(['email']);
$response->assertSessionHasErrors(['email']);
```

JSON APIs:

```php
$response = $this->postJson('/api/posts', $payload);
$response->assertCreated();
$response->assertUnprocessable();                  // 422
$response->assertJsonValidationErrors(['email']);  // 422 body shape
```

## Fakes for side effects

```php
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

Queue::fake();           // Queue::assertPushed(ProcessPodcast::class);
Mail::fake();            // Mail::assertSent(WelcomeMail::class);
Mail::assertQueued(...);
Notification::fake();    // Notification::assertSentTo($user, OrderShipped::class);
Event::fake();           // Event::assertDispatched(OrderPlaced::class);
Http::fake([             // HTTP::assertSent(...)
    'example.com/*' => Http::response(['ok' => true], 200),
]);
Storage::fake('public'); // Storage::disk('public')->assertExists("photos/{$id}.jpg");
```

Fake only what the test asserts on — faking everything hides real integration bugs. `Http::fake()`
with no arguments stubs every request, useful to isolate a test from third-party flakiness.

Authentication: `$this->actingAs($user)` (session) or Sanctum (see `auth-basics.md`):

```php
Sanctum::actingAs(User::factory()->create(), ['*']);
```

## Testing Artisan commands

```php
Artisan::shouldReceive('call')->once()->with('mail:send', ['user' => 1]);

// or actually run it:
$this->artisan('mail:send', ['user' => 1])
     ->expectsOutput('Sending email to: 1')
     ->assertSuccessful();
```

## Parallel testing

`php artisan test --parallel` runs suites across processes with auto-created per-process test
databases (`your_db_test_1`, `your_db_test_2`...). Recreate them with `--recreate-databases`.
Hooks are available via the `ParallelTesting` facade in `AppServiceProvider` (e.g. seed each test
database on creation).

## What to actually test

- Feature tests over unit tests for framework-touching code; unit tests for pure helpers.
- Validation: one test per rule boundary (missing, wrong type, too long), not just happy paths.
- Authorization: assert 403 for the wrong user, not only 200 for the right one.
- Jobs: dispatch assertions in feature tests, plus `withFakeQueueInteractions()` for
  release/delete/fail behavior.
- Don't assert on raw log file contents — assert on observable side effects (emails, DB rows,
  queued jobs) instead.
