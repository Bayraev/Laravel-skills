# Queues and Jobs

## Job classes — the 13 shape

```bash
php artisan make:job ProcessPodcast
php artisan make:job-middleware RateLimited
```

```php
<?php

namespace App\Jobs;

use App\Models\Podcast;
use App\Services\AudioProcessor;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Queue\Attributes\Backoff;
use Illuminate\Queue\Attributes\Tries;
use Illuminate\Queue\Attributes\WithoutRelations;

#[Tries(3)]
#[Backoff([10, 30, 60])]          // seconds between attempts
#[WithoutRelations]                // don't serialize loaded model relations (smaller payloads)
class ProcessPodcast implements ShouldQueue
{
    use Queueable;                 // the single trait — covers dispatching, queue interaction, serialization

    public function __construct(
        public Podcast $podcast,   // only the model ID is serialized; re-fetched on handle
    ) {}

    public function handle(AudioProcessor $processor): void
    {
        $processor->process($this->podcast);
    }
}
```

Key points:

- `Queueable` replaces the old four-trait block. Models serialize by identifier and re-retrieve on execution — payload stays small, data stays fresh.
- Prefer `#[WithoutRelations]` on the class (or `$podcast->withoutRelations()` / `->withoutRelation('comments')` in the constructor) over shipping big relation graphs through the queue.
- Binary data must be `base64_encode`d before passing into a job.
- Time-based retry instead of attempt-count: define `retryUntil(): DateTime { return now()->plus(minutes: 30); }`.

Other job attributes: `#[Timeout(120)]`, `#[FailOnTimeout]`, `#[MaxExceptions(3)]`, `#[UniqueFor(3600)]` (with `ShouldBeUnique`).

## Unique and debounced jobs

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Queue\Attributes\UniqueFor;
use Illuminate\Queue\Attributes\DebounceFor;

#[UniqueFor(3600)]
class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    public function uniqueId(): string { return (string) $this->product->id; }
    // uniqueVia(): Repository — use a specific cache store for the lock
}

#[DebounceFor(30, maxWait: 120)]       // rapid re-dispatch: only the latest runs; never deferred > 120s
class RefreshDashboard implements ShouldQueue
{
    public function debounceId(): string { return (string) $this->userId; }
}
```

Unique = "skip if one is already queued/running". Debounce = "collapse bursts, run only the newest" (superseded jobs fire `JobDebounced`). They are mutually exclusive — pick one per job. Both need a lock-capable cache store (redis, memcached, dynamodb, database, file, array) shared by all dispatching servers.

Encrypted jobs: add `ShouldBeEncrypted` to encrypt payloads at rest in the queue.

## Dispatching

```php
ProcessPodcast::dispatch($podcast);                          // queue
ProcessPodcast::dispatch($podcast)->delay(now()->plus(minutes: 10));
ProcessPodcast::dispatchSync($podcast);                      // run now (request lifecycle)
ProcessPodcast::dispatchAfterResponse($podcast);             // after response sent
Bus::dispatchBatch(new BatchCollection([...]));


// Chains and batches
Bus::chain([
    new ProcessPodcast($podcast),
    new PublishPodcast($podcast),
    fn () => $podcast->update(['status' => 'published']),
])->dispatch();

$batch = Bus::batch([
    new ProcessPodcast($podcast),
    new OptimizePodcast($podcast),
])->then(fn (Batch $batch) => /* all succeeded */)
  ->catch(fn (Batch $batch, Throwable $e) => /* first failure */)
  ->finally(fn (Batch $batch) => /* always */)
  ->allowFailures()
  ->onQueue('podcasts')
  ->dispatch();
```

## Queue routing (new in 13) — configure centrally, not at call sites

In a service provider `boot()`:

```php
use Illuminate\Support\Facades\Queue;

Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');
Queue::route(RequiresVideo::class, queue: 'video');              // trait, interface, or parent class
Queue::route([
    ProcessPodcast::class => ['redis', 'podcasts'],               // [connection, queue]
    ProcessVideo::class => 'videos',                              // queue only, default connection
]);

// Move a whole queue to different infrastructure without touching jobs:
Queue::forward('reports', 'reports.fifo', 'sqs');
Queue::forward(['reports' => 'reports.fifo', 'emails' => 'emails.fifo'], connection: 'sqs');
```

Per-dispatch `->onQueue()` / `->onConnection()` still override routing. Use routing as the default mechanism — it keeps queue topology in one auditable place.

## Job middleware

```php
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\Middleware\ThrottlesExceptions;

public function middleware(): array
{
    return [
        (new RateLimited('backups'))->releaseAfter(60),
        (new WithoutOverlapping($this->order->id))->expireAfter(180)->shared(),
        (new ThrottlesExceptions(10, 300))
            ->when(fn (Throwable $e) => $e instanceof HttpException)
            ->backoff(5)
            ->report()
            ->by('payment-gateway'),
    ];
}
```

- `WithoutOverlapping`: prevent concurrent runs by key; `shared()` makes the key span job classes; `dontRelease()` discards overlaps instead of re-queuing.
- `ThrottlesExceptions`: pause a job hammering a flaky third party; `deleteWhen(Exception::class)` deletes it instead.
- Rate limiters are defined with `RateLimiter::for('backups', fn (object $job) => Limit::perHour(1)->by($job->user->id))`.
- Middleware also attach to queued event listeners, mailables, and notifications.

## Running workers

```bash
php artisan queue:work --queue=high,default     # priority order left→right
php artisan queue:work --timeout=120 --tries=3 --max-jobs=1000 --rest=5
php artisan queue:restart                        # graceful restart after deploy (workers exit when idle)
php artisan queue:failed                         # list failed jobs
php artisan queue:retry {uuid}                   # retry one / 'all'
php artisan queue:prune-failed --hours=48
php artisan queue:clear redis --queue=emails
```

Deployment: restart workers after every release (`php artisan queue:restart` + supervisor restarts); `failed_jobs` table required (ships in default migrations). Supervisor config pattern:

```ini
[program:laravel-worker]
command=php /var/www/artisan queue:work --sleep=3 --tries=3 --max-time=3600
numprocs=2
autostart=true
autorestart=true
stopwaitsecs=3600
```

For Redis queues in production use **Laravel Horizon** (dashboard, supervisor tags, metrics): `composer require laravel/horizon && php artisan horizon`.

## Database driver notes

`make:queue-table` + `migrate` if missing. Default migrations already include `jobs`, `job_batches`, `failed_jobs`. SQS: oversized payloads can spill to a cache store via the `overflow` connection config (`'overflow' => ['enabled' => ..., 'store' => ...]`).

## Failed jobs and cleanup

Implement `failed(Throwable $e): void` on the job for cleanup/notification on final failure. `queue:prune-batches`, `queue:prune-failed` for housekeeping. `ShouldBeUniqueUntilProcessing` unlocks a unique job as soon as processing begins rather than after completion.
