# Testing

Sources: [testing](https://laravel.com/framework/docs/10.x/testing), [http-tests](https://laravel.com/framework/docs/10.x/http-tests). Run with `php artisan test` (wraps phpunit with better output); `--profile` to find slow tests is new in Laravel 10; `--parallel` needs `composer require brianium/paratest --dev`.

## Structure

- `tests/Feature/` — full HTTP/stack flows (the bulk of a good suite).
- `tests/Unit/` — narrow logic tests. Test classes end in `Test`; extend `Tests\TestCase`.
- `phpunit.xml` defines the environment: `APP_ENV=testing`, often `DB_CONNECTION=sqlite` + `:memory:` for speed.
- `.env.testing` overrides `.env` when the suite runs.

```bash
php artisan make:test PhotoTest                 # → tests/Feature/PhotoTest.php
php artisan make:test PhotoTest --unit          # → tests/Unit/

php artisan test                                # whole suite
php artisan test --filter=test_guest_can_view_photos
php artisan test --testsuite=Feature --stop-on-failure
php artisan test --parallel                     # paratest
php artisan test --profile                      # slowest tests first (Laravel 10)
php artisan test --coverage                     # needs a coverage driver (xdebug/pcov)
```

## Database testing

`RefreshDatabase` (the default choice — works with migrations and transactions):

```php
namespace Tests\Feature;

use App\Models\Photo;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PhotoTest extends TestCase
{
    use RefreshDatabase;

    public function test_guest_cannot_create_photos(): void
    {
        $response = $this->post('/photos', ['title' => 'Beach']);

        $response->assertRedirect('/login');
    }

    public function test_user_can_create_photo(): void
    {
        $user = User::factory()->create();
        $photo = Photo::factory()->make()->toArray();

        $response = $this->actingAs($user)->post('/photos', $photo);

        $response->assertRedirect(route('photos.index'));
        $this->assertDatabaseHas('photos', ['title' => $photo['title'], 'user_id' => $user->id]);
    }
}
```

Traits: `RefreshDatabase` (migrate once, wrap each test in a transaction — also handles `:memory:`), `DatabaseMigrations` (full re-migration per test, slower), `DatabaseTransactions` (transactions only, no migrate), `WithoutMiddleware` (escape middleware wholesale — sparingly).

Assertions: `assertDatabaseHas('photos', [...])`, `assertDatabaseMissing(...)`, `assertModelExists($photo)` / `assertModelMissing($photo)`, `assertDatabaseCount('photos', 5)`.

## HTTP test toolkit

```php
$this->get('/photos');
$this->post('/photos', ['title' => 'Beach']);
$this->putJson("/photos/{$photo->id}", ['title' => 'Updated']);   // JSON APIs
$this->deleteJson("/photos/{$photo->id}");

$this->followingRedirects()->post('/photos', $data);
$this->get('/dashboard')->assertRedirect('/login');

// Response assertions
$response->assertStatus(200);
$response->assertOk();  // also assertNotFound(), assertForbidden(), assertUnauthorized()
$response->assertSee('Beach');  assertSeeText('Beach');  assertDontSee('secret');
$response->assertViewIs('photos.index');
$response->assertViewHas('photos', fn ($photos) => $photos->count() === 3);
$response->assertJson(['ok' => true]);
$response->assertJsonPath('data.0.title', 'Beach');
$response->assertJsonCount(3, 'data');
$response->assertJsonStructure(['data' => [['id', 'title']]]);
$response->assertDownload();
$response->assertCookie('theme', 'dark');
$response->assertSessionHas('status', 'Photo created!');
$response->assertInvalid(['title']);            // after a failed validation POST
```

## Authentication helpers

```php
$user = User::factory()->create();
$this->actingAs($user);                         // session auth (web)
$this->actingAs($user, 'sanctum');              // token guard (APIs)

// Or full login flow
$this->post('/login', ['email' => $u->email, 'password' => 'password']);
```

## Mocking and faking

Fake heavy collaborators per-test rather than hitting real services:

```php
use Illuminate\Support\Facades\{Cache, Http, Mail, Notification, Process, Queue, Storage};

Queue::fake();
$this->post('/photos', $data);
Queue::assertPushed(ProcessPodcast::class);
Queue::assertPushed(ProcessPodcast::class, fn ($job) => $job->podcast->id === $photo->id);
Queue::assertNotPushed(AnotherJob::class);

Http::fake([
    'api.stripe.com/*' => Http::response(['ok' => true], 200),
    '*' => Http::response([], 500),
]);
Http::assertSent(fn ($request) => $request->hasHeader('Authorization'));

Mail::fake();                Mail::assertSent(WelcomeMail::class);
Notification::fake();        Notification::assertSentTo($user, PhotoApproved::class);
Storage::fake('public');     Storage::disk('public')->assertExists("photos/{$photo->id}.jpg");
Cache::shouldReceive('get')->andReturn('cached');
Process::fake();             Process::assertRan('ls -la');   // Laravel 10 Process service
```

Time travel (Laravel 10's `Travel::` test helper):

```php
$this->travel(5)->days();
$this->travelTo(now()->addYear());
$this->travelBack();
```

## Writing good Laravel tests

- Test behavior through HTTP where possible (`$this->post(...)`) — it exercises routing, middleware, validation, and controllers as one unit. Drop to unit tests for pure domain logic only.
- Seed the world with factories, never fixtures of real data.
- One behavior per `test_*` method; name it as the behavior it pins (`test_admin_can_delete_any_photo`).
- Assert on visible outcomes (redirects, database rows, queued jobs), not on implementation internals.
- Slow suite? `--profile`, then fix the N+1s and push slow edge cases behind queued jobs rather than weakening assertions.
