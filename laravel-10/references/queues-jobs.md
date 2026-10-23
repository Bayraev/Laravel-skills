# Queues and Jobs

Source: [queues](https://laravel.com/framework/docs/10.x/queues). Laravel 10-specific: `Bus::dispatchNow()` / `dispatch_now()` were **removed** — use `Bus::dispatchSync()` / `dispatch_sync()`.

## When to queue

Queue anything a user shouldn't wait for: mail sending, notifications, external API calls, file processing, report generation. Keep requests-and-responses fast; push slow side effects into jobs. Prerequisite: a queue driver — `database` (zero infra: `php artisan queue:table && php artisan migrate`) for small apps, Redis + Horizon for real load. Configure via `QUEUE_CONNECTION` in `.env`.

## Writing a job

```bash
php artisan make:job ProcessPodcast
```

```php
namespace App\Jobs;

use App\Models\Podcast;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Process;

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;          // attempts before failing
    public int $timeout = 300;      // seconds (worker-level too)
    public array $backoff = [10, 30, 60];  // seconds between retries (or `backoff(): array`)

    public function __construct(
        public readonly Podcast $podcast,   // SerializesModels stores the id, re-fetches on run
    ) {}

    public function handle(): void
    {
        $result = Process::timeout(120)->run("transcode {$this->podcast->path}");

        $result->throw();            // ProcessFailedException → job fails → retried
    }
}
```

`handle()` is resolved through the service container — type-hint dependencies there, not in the constructor (constructor is for payload only).

## Dispatching

```php
use App\Jobs\ProcessPodcast;

ProcessPodcast::dispatch($podcast);                    // onto the queue
ProcessPodcast::dispatchSync($podcast);                // run now, synchronously (replaces dispatch_now)
ProcessPodcast::dispatchAfterResponse($podcast);       // after the response is sent, same request

ProcessPodcast::dispatch($podcast)
    ->delay(now()->addMinutes(10))
    ->onQueue('podcasts')          // named queue
    ->onConnection('redis');       // specific connection
```

Helpers: `dispatch(new ProcessPodcast($podcast))` (global function); the `Dispatchable` trait enables the static API.

## Chaining, batching, unique jobs

```php
use App\Jobs\{OptimizePodcast, ProcessPodcast, ReleasePodcast};
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new ProcessPodcast($podcast),
    new OptimizePodcast($podcast),
    new ReleasePodcast($podcast),
])->onConnection('redis')->onQueue('podcasts')->dispatch();
// A failure in any link stops the chain.
```

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;
use Throwable;

$batch = Bus::batch([
    new ProcessPodcast($podcast1),
    new ProcessPodcast($podcast2),
])->then(function (Batch $batch) { /* all succeeded */ })
  ->catch(function (Batch $batch, Throwable $e) { /* first failure */ })
  ->finally(function (Batch $batch) { /* always */ })
  ->allowFailures()
  ->dispatch();

// Progress: $batch->progress(), ->processedJobs(), ->failedJobs(), ->cancel()
```

Batch tracking requires the `job_batches` table (`php artisan queue:batches-table && php artisan migrate`).

Unique — suppressed while an identical job is pending:

```php
use Illuminate\Cache\Repository;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Cache;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    public function uniqueVia(): Repository   // optional: dedicated cache lock store
    {
        return Cache::driver('redis');
    }
}
```

```
Batch tracking requires the `job_batches` table (`php artisan queue:batches-table && php artisan migrate`).

## Rate limiting a job

```php
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Support\Facades\RateLimiter;

public function middleware(): array
{
    return [(new RateLimited('podcasts'))->dontRelease()];   // release() = retry later (default)
}

// RateLimiter::for('podcasts', ...) is defined in AppServiceProvider::boot(), like any other limiter
```

## Workers

```bash
php artisan queue:work                    # long-running; don't restart per deploy — see below
php artisan queue:work --queue=podcasts,default   --tries=3 --timeout=300
php artisan queue:restart                 # gracefully restarts workers after current job
php artisan queue:failed                  # list failures
php artisan queue:retry all               # or specific ids
php artisan queue:flush
php artisan queue:prune-batches
```

Deployment: `queue:restart` after each deploy (Supervisor/systemd handles restarts; Horizon does this for Redis). Job code changes require a restart — a worker running old code is the classic "my fix didn't take" bug.

`failed_jobs` table required (`php artisan queue:failed-table && php artisan migrate`). Custom failure hook: `public function failed(Throwable $e): void` on the job (alerting, cleanup).

Job events (`Queue::before(...)`, `Queue::failing(...)`, etc.) register in `AppServiceProvider::boot()` for cross-cutting logging/timing.

## Schedule frequent jobs via the Console Kernel

On Laravel 10, scheduling lives in `app/Console/Kernel.php` (not `routes/console.php` as in 11):

```php
use App\Jobs\GenerateReport;
use Illuminate\Console\Scheduling\Schedule;

// app/Console/Kernel.php on Laravel 10
class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->job(new GenerateReport)->hourly();
    }
}
```

One cron entry on the server: `* * * * * php /path/to/artisan schedule:run` (or `php artisan schedule:work` locally).
