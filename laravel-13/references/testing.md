# Testing

## Setup and running

Pest and PHPUnit both ship out of the box (`phpunit.xml` preconfigured; the `testing` environment sets session/cache to `array`).

```bash
php artisan make:test UserTest               # tests/Feature
php artisan make:test UserTest --unit        # tests/Unit
php artisan make:test UserTest --pest        # Pest style

php artisan test                             # preferred runner (pretty output, wraps pest/phpunit)
vendor/bin/pest
php artisan test --testsuite=Feature --stop-on-failure
php artisan test --parallel                  # needs brianium/paratest; auto-creates suffixed test DBs
php artisan test --coverage --min=80.3       # needs Xdebug or PCOV
php artisan test --profile                   # 10 slowest tests
```

Test environment config lives in `phpunit.xml` / `.env.testing`. If you change config values, run `php artisan config:clear` — cached config silently overrides test env.

**Most tests should be Feature tests** (the docs' explicit guidance): they exercise real objects together and give the most confidence. Unit tests don't boot the app. Use `#[UnitTest]` (`Illuminate\Foundation\Testing\Attributes\UnitTest`) on a method inside a TestCase subclass to skip booting for that one test.

Speed up suites with the `WithCachedConfig` trait (build config once per run instead of per test):

```php
// PHPUnit
class ConfigTest extends TestCase
{
    use WithCachedConfig;
}

// Pest — globally
pest()->use(Illuminate\Foundation\Testing\WithCachedConfig::class);
```

## HTTP tests

```php
<?php

namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase;

    public function test_guests_cannot_create_posts(): void
    {
        $this->post('/posts', ['title' => 'Hi'])
             ->assertRedirect('/login');
    }

    public function test_users_can_create_posts(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->post('/posts', [
            'title' => 'My post',
            'body' => 'Content',
        ]);

        $response->assertRedirect(route('posts.index'));
        $this->assertDatabaseHas('posts', ['title' => 'My post', 'user_id' => $user->id]);
        $this->assertModelExists(Post::where('title', 'My post')->first());
    }
}
```

Common assertions: `assertStatus(200)` / `assertOk()` / `assertCreated()` / `assertNotFound()` / `assertForbidden()` / `assertUnauthorized()`, `assertJson([...])`, `assertJsonPath('data.0.title', '...')`, `assertJsonValidationErrors(['title'])`, `assertSee('text')`, `assertViewHas('posts')`, `assertRedirect(route('x'))`, `assertDatabaseHas/DatabaseMissing/DatabaseCount`, `assertModelExists/ModelMissing`, `assertFileExists`.

JSON APIs: `$this->getJson('/api/posts')`, `postJson`, and `assertJsonStructure(['data' => [['id', 'title']]])`.

## Database testing

- `RefreshDatabase` — migrate once per run, wrap each test in a transaction. Default choice.
- `DatabaseMigrations` / `DatabaseTransactions` — the older, heavier variants.
- `LazilyRefreshDatabase` — refresh before first test that hits the DB.
- Parallel runs auto-create per-process databases (`your_db_test_1`, `_2`); recreate with `--recreate-databases`; hooks via `ParallelTesting::setUpTestDatabase(fn ($db, $token) => Artisan::call('db:seed'))`.

## Fakes

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;

Queue::fake();
$this->post('/podcasts', $payload);
Queue::assertPushed(ProcessPodcast::class);
Queue::assertPushed(ProcessPodcast::class, fn ($job) => $job->podcast->id === 1);
Queue::assertNotPushed(AnotherJob::class);
Queue::assertPushedOn('podcasts', ProcessPodcast::class);

Bus::fake();  Bus::assertChained([A::class, B::class]);  Bus::assertBatched(fn ($batch) => $batch->jobs->count() === 2);

Http::fake([
    'example.com/*' => Http::response(['ok' => true], 200),
    'api.fail/*' => Http::response([], 500),
]);
Http::assertSent(fn ($request) => $request->url() === 'example.com/x');
Http::preventStrayRequests();     // fail any request that wasn't faked

Mail::fake();      Mail::assertSent(OrderShipped::class);
Notification::fake();  Notification::assertSentTo($user, OrderPaid::class);
Storage::fake('avatars');  Storage::disk('avatars')->assertExists('a.jpg');

Event::fake();     Event::assertDispatched(OrderPlaced::class);
```

Test queued behavior synchronously: `Event::fake([OnlyThisEvent::class])`, or use `Bus::fake()` to assert dispatch instead of running jobs.

## Authentication shortcuts

```php
$this->actingAs($user);
$this->actingAs($user, 'guard-name');
$user = User::factory()->make();           // not persisted — light auth payload
```

## Testing AI SDK agents (when the app uses laravel/ai)

Never let tests hit real provider APIs. The AI SDK ships its own testing layer — fakes for agents/responses so you can assert prompts, tool calls, and structured output deterministically. The exact fake API is version-sensitive and new in 13: read the "Testing" section of https://laravel.com/framework/docs/13.x/ai-sdk before writing these tests, and pin expected prompt/tool content in assertions.

## Conventions that keep suites healthy

- Name tests `test_what_happens_under_conditions` or Pest `it('creates a post when valid')`.
- One behavior per test; arrange with factories, act, assert.
- Prefer asserting redirects + DB state for web endpoints; JSON structure for APIs.
- Call `parent::setUp()` first / `parent::tearDown()` last when overriding.
- Time-sensitive logic: `Carbon::setTestNow()` / `travel()` — `travel(5)->minutes()`, `travelTo($time)`, `travelBack()`.
