# Testing (Laravel 12)

## Table of contents

- [Setup and conventions](#setup-and-conventions)
- [Running tests](#running-tests)
- [Feature tests: HTTP + database](#feature-tests-http--database)
- [Database testing](#database-testing)
- [Fakes: HTTP, mail, queues, notifications](#fakes-http-mail-queues-notifications)
- [Authentication in tests](#authentication-in-tests)
- [Unit tests](#unit-tests)
- [Testing Laravel 12 specifics](#testing-laravel-12-specifics)

## Setup and conventions

Laravel 12 ships with **Pest 3** and **PHPUnit 11** support (the installer asks which you prefer; both run the same tests). Structure:

- `tests/Feature/` — tests that boot the application: HTTP requests, database, services wired through the container. **Most tests belong here** — they exercise the system as it actually runs.
- `tests/Unit/` — tests of isolated classes; the framework is NOT booted, so no facades, no Eloquent, no HTTP helpers.
- `phpunit.xml` sets `APP_ENV=testing` and switches session/cache to the `array` driver automatically; `.env.testing` overrides `.env` for the test run.

Prefer Pest's function style for new tests (it's the 12 default), but match the existing file's style when extending:

```php
<?php

test('guests can view the pricing page', function () {
    $response = $this->get('/pricing');

    $response->assertOk();
});
```

## Running tests

```shell
php artisan test                     # Pest/PHPUnit with a readable report
php artisan test --testsuite=Feature
php artisan test --parallel          # multiple processes (needs brianium/paratest)
php artisan test --coverage          # Xdebug/PCOV required
php artisan test --coverage --min=80 # enforce a threshold
php artisan test --profile           # list the 10 slowest tests
vendor/bin/pest                      # direct Pest
vendor/bin/phpunit                   # direct PHPUnit
```

Parallel test runs auto-create per-process test databases (`your_db_test_1`, `your_db_test_2`, ...); re-create them with `--recreate-databases`. With a cached config (`php artisan config:cache`), run `config:clear` before testing or config changes won't apply.

## Feature tests: HTTP + database

```php
<?php

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('a user can create a post', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->post(route('posts.store'), [
            'title' => 'My first post',
            'body' => 'Hello world',
        ]);

    $response->assertRedirectToRoute('posts.show', Post::latest('id')->first());

    expect(Post::where('title', 'My first post')->exists())->toBeTrue();
});

test('validation rejects a missing title', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->from(route('posts.create'))
        ->post(route('posts.store'), ['body' => 'no title'])
        ->assertRedirect(route('posts.create'))
        ->assertSessionHasErrors('title');
});
```

The core HTTP vocabulary — use the narrowest assertion that proves the point:

```php
$this->get('/api/posts')->assertOk()->assertJsonCount(3, 'data');
$this->getJson('/api/posts/999')->assertNotFound();
$this->postJson('/api/posts', $payload)
    ->assertCreated()
    ->assertJsonValidationErrors(['title']);
$this->patchJson("/api/posts/{$post->id}", ['title' => ''])
    ->assertUnprocessable()                        // 422 — validation
    ->assertJsonValidationErrorFor('title');
$this->deleteJson("/api/posts/{$post->id}")->assertNoContent();
```

Both redirects **and** JSON validation errors are assertable (`assertSessionHasErrors` after a redirect, `assertJsonValidationErrors` for XHR) — test whichever contract the client uses.

## Database testing

```php
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);
```

`RefreshDatabase` runs migrations (once per process when parallel) and wraps each test in a transaction — fast, and safe with SQLite in-memory or a dedicated test database. `DatabaseTransactions` is the lighter variant when migrations are too slow to re-run. Assert state with Eloquent, and seed exactly what the scenario needs:

```php
test('drafts are hidden from guests', function () {
    Post::factory()->published()->count(2)->create();
    Post::factory()->draft()->create();

    $this->get('/posts')->assertOk()
        ->assertSeeInOrder(Post::published()->pluck('title')->all());
});
```

## Fakes: HTTP, mail, queues, notifications

Assert that side effects happen without causing them:

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;

test('creating a podcast queues processing', function () {
    Bus::fake();
    Http::fake(['*.service.dev/*' => Http::response(['ok' => true])]);

    $this->post(route('podcasts.store'), $this->podcastAttributes());

    Bus::assertDispatched(ProcessPodcast::class);
    Http::assertSent(fn ($request) => $request->url() === 'https://api.service.dev/check');
});
```

- `Bus::fake()` + `Bus::assertDispatched` / `assertNotDispatched` / `assertDispatchedTimes` — jobs.
- `Queue::fake()` when you care about the queue connection specifically.
- `Mail::fake()` + `Mail::assertSent(WelcomeEmail::class, fn ($mailable) => $mailable->hasTo($user->email))`.
- `Notification::fake()` + `Notification::assertSentTo($user, PodcastPublished::class)`.
- `Http::fake([...])` stubs outbound HTTP; `Http::preventStrayRequests()` makes un-stubbed calls throw — turn it on in tests to catch hidden integrations.
- `Storage::fake('public')` + `$this->post(...)->assertFileExists(Storage::disk('public')->path($path))` for uploads.

## Authentication in tests

```php
$user = User::factory()->create();
$this->actingAs($user);                       // web guard
$this->actingAs($user, 'sanctum');            // API token guard

// Sanctum tokens with abilities:
use Laravel\Sanctum\Sanctum;

Sanctum::actingAs(User::factory()->create(), ['posts:view']);
```

For unauthenticated API behavior, test the 401 shape explicitly: `$this->getJson('/api/user')->assertUnauthorized();`.

## Unit tests

`php artisan make:test MoneyTest --unit` writes to `tests/Unit` extending `PHPUnit\Framework\TestCase` — no app boot. Unit-test pure objects: value objects, calculators, enum logic, collection macros. If a "unit" test starts needing facades or models, it's a feature test wearing a costume — move it and let `RefreshDatabase` do the work.

## Testing Laravel 12 specifics

- **Pest 3 / PHPUnit 11**: assertion names largely match older docs; a few PHPUnit assertions renamed in 11 (`assertFileDoesNotExist` etc.) — if a copied test fails on a missing method, check the PHPUnit 11 changelog, not Laravel.
- **`WithCachedConfig`** trait (`Illuminate\Foundation\Testing\WithCachedConfig` or `pest()->use(...)`) builds config once per run for large suites.
- **Parallel databases**: code that hardcodes the test database name breaks under `--parallel`; read the connection config instead, or use `ParallelTesting::token()` when a shared resource needs per-process isolation.
- Test what the framework gives you for free rather than re-testing it (don't test that `validate` validates; test that YOUR rule set rejects the inputs your domain cares about).
