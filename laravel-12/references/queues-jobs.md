# Queues and Jobs (Laravel 12)

Queues move slow work (mail, API calls, PDFs, imports) out of the request cycle. Use this reference whenever a task takes more than a few hundred milliseconds or tolerates retry-with-delay.

## Creating a job

```shell
php artisan make:job ProcessPodcast
```

```php
<?php

namespace App\Jobs;

use App\Models\Podcast;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * The number of times the job may be attempted.
     */
    public int $tries = 3;

    /**
     * Seconds to wait before retrying a failed attempt.
     */
    public int $backoff = 60;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Podcast $podcast,      // SerializesModels stores the id, refetches on run
    ) {}

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // Dependencies type-hinted here are resolved from the container,
        // so the job class stays testable and stateless.
        Log::info('Processing podcast', ['id' => $this->podcast->id]);
    }
}
```

Why this shape: `ShouldQueue` is what makes it queued (drop the interface and `dispatch()` runs it synchronously — a deliberate escape hatch). `SerializesModels` only serializes the primary key, so you never persist a stale model snapshot — the fresh row is re-fetched when the job runs.

## Dispatching

```php
use App\Jobs\ProcessPodcast;
use Illuminate\Support\Facades\Bus;

ProcessPodcast::dispatch($podcast);                          // default queue
ProcessPodcast::dispatch($podcast)->onQueue('podcasts');     // named queue
ProcessPodcast::dispatch($podcast)->delay(now()->addMinutes(5));

dispatch(function () use ($podcast) {                        // closure jobs (serialize with care)
    $podcast->process();
});

Bus::chain([                                                // ordered pipeline, stops on failure
    new OptimizePodcast($podcast),
    new ProcessPodcast($podcast),
    new ReleasePodcast($podcast),
])->dispatch();

Bus::batch([                                                // parallel fan-out with progress tracking
    new ProcessPodcast($podcastA),
    new ProcessPodcast($podcastB),
])->then(fn (Batch $batch) => Log::info('Batch done'))
  ->finally(fn (Batch $batch) => /* ... */)
  ->allowFailures()
  ->dispatch();
```

Dispatch after the DB transaction commits, or the worker may pick up a job whose row isn't committed yet: put the dispatch inside `DB::transaction()` and Laravel 12 handles it automatically via transaction-aware dispatching (`$afterCommit` defaults sensibly); be explicit with `->afterCommit()` / `->beforeCommit()` if behavior surprises you.

## Running workers

```shell
php artisan queue:work               # daemon — production worker (run under Supervisor / Horizon)
php artisan queue:work --queue=high,default
php artisan queue:listen             # re-reads code per job (slow, dev only)
php artisan queue:restart            # gracefully restart workers after deploy (signal via cache)
php artisan queue:failed             # list failures
php artisan queue:retry all          # re-queue failed jobs
```

The `queue:work` daemon loads the framework **once** — code changes require a restart, which is why deploy scripts run `php artisan queue:restart` (workers pick it up and exit; the Supervisor/Horizon config restarts them). Default `QUEUE_CONNECTION=database` in a fresh 12 app (a `jobs` table migration ships in the skeleton); switch via `QUEUE_CONNECTION` in `.env` (`redis` for throughput, `sync` to run inline, `sqs` on AWS).

For Redis-backed queues at scale, **Laravel Horizon** (separate package) gives a dashboard, supervisor config, and metrics — the standard answer once queue volume matters.

## Failure handling

```php
public function failed(?Throwable $e): void
{
    // Called after all attempts are exhausted — alert, compensate, clean up.
    report($e);
    $this->podcast->markFailed();
}
```

Control retries per job: `public int $tries = 3;` (or `tries()` method), `public int $backoff = 60;` (or `backoff()` / `retryUntil()` for a deadline), and `public int $timeout = 120;` for the per-job wall-clock limit. Exponential backoff: `public function backoff(): array { return [10, 30, 120]; }` (one value per attempt).

Failures land in the `failed_jobs` table; retry with `queue:retry {id|all}`, delete with `queue:forget {id}`, and prune old records with `Schedule::command('queue:prune-failed --hours=48')->daily();` in `routes/console.php`.

**Job middleware** keep cross-cutting concerns out of `handle()`:

```php
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\Middleware\WithoutOverlapping;

public function middleware(): array
{
    return [
        (new WithoutOverlapping($this->podcast->id))->expireAfter(300),
        new RateLimited('podcasts'),
    ];
}
```

## When NOT to queue

- The caller needs the result now → it's a synchronous operation; use a service class.
- The task is trivially fast → the queue round-trip costs more than it saves.
- The data must commit atomically with a request → run in-request or in the same transaction.

Queues buy resilience (retries, isolation from request timeouts), not speed of the individual task — design jobs idempotent, because they *will* occasionally run twice (retries, at-least-once delivery).
