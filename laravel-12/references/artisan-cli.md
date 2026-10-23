# Artisan, Commands, and Scheduling (Laravel 12)

## Table of contents

- [Everyday commands](#everyday-commands)
- [Writing commands](#writing-commands)
- [Input/output](#inputoutput)
- [Scheduling](#scheduling)
- [Optimization and caches](#optimization-and-caches)
- [Tinker](#tinker)

## Everyday commands

```shell
php artisan list                       # everything
php artisan list make                  # just the generators
php artisan about                      # env, cache states, drivers — first stop when diagnosing
php artisan help migrate               # options for any command
```

The generators that keep code conformant — always prefer them over hand-writing files (stubs match namespaces, naming, and 12.x conventions):

```shell
php artisan make:controller PostController --resource
php artisan make:model Post --all
php artisan make:model Post --migration --factory --seed --policy
php artisan make:request StorePostRequest
php artisan make:middleware EnsureTokenIsValid
php artisan make:command SendDailyDigest
php artisan make:job ProcessPodcast
php artisan make:event PodcastProcessed
php artisan make:listener SendPodcastNotification --event=PodcastProcessed
php artisan make:observer PostObserver --model=Post
php artisan make:policy PostPolicy --model=Post
php artisan make:rule Uppercase
php artisan make:seeder FlightSeeder
php artisan make:factory FlightFactory --model=Flight
php artisan make:cast Money
php artisan make:component Alert
php artisan make:test UserTest [--unit]
```

Customize the stubs once if your team's conventions differ (`php artisan stub:publish` → edit `stubs/`) rather than fixing each generated file by hand.

## Writing commands

Class-based commands for anything with weight (options, DI, tests):

```shell
php artisan make:command SendDailyDigest
```

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Mail;

class SendDailyDigest extends Command
{
    /**
     * The name and signature of the command.
     */
    protected $signature = 'digest:send
                            {--dry-run : Print what would be sent without sending}';

    /**
     * The console command description.
     */
    protected $description = 'Send the daily digest email to active users';

    public function handle(): int
    {
        $users = User::query()->active()->get();

        if ($this->option('dry-run')) {
            $this->table(['id', 'email'], $users->map(fn (User $u) => [$u->id, $u->email])->all());

            return self::SUCCESS;
        }

        $users->each(fn (User $user) => Mail::to($user)->send(new DailyDigest($user)));

        return self::SUCCESS;
    }
}
```

Return `self::SUCCESS` (0) or `self::FAILURE` (1) — exit codes matter in CI and deployment scripts.

Closure commands go straight into `routes/console.php` — the right place for small, app-specific commands (and the default home of the schedule):

```php
use Illuminate\Support\Facades\Artisan;

Artisan::command('reports:count', function (): void {
    $this->info(Report::count().' reports');
})->purpose('Count reports');
```

## Input/output

```php
$this->argument('name');                        // positional arg
$this->option('dry-run');                       // --dry-run flag

$this->info('Saved.');
$this->error('Something broke.');
$this->line('raw output');

$this->ask('What is your name?');
$this->secret('Password?');
$this->confirm('Proceed?');                     // bool
$this->anticipate('Env?', ['local', 'production']);
$this->choice('Driver?', ['mysql', 'sqlite'], 0);

$this->table(['Name', 'Status'], $rows);
$this->withProgressBar($items, fn ($item) => $item->process());

$this->components->info('...');                 // the styled components Artisan itself uses
```

Signature grammar for richer definitions: `'digest:send {user} {--queue=} {--force}'` — `{user}` required, `{user?}` optional, `{user=me}` default, `{--queue=}` option with value.

## Scheduling

In Laravel 11+/12 there is **no `Console/Kernel.php`** — the schedule is defined in `routes/console.php` using the `Schedule` facade:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('digest:send')->dailyAt('08:00')->timezone('America/Chicago');
Schedule::command('sanctum:prune-expired --hours=24')->daily();
Schedule::job(new ProcessPodcast($podcast))->hourly();
Schedule::call(fn () => User::where('stale', true)->update(['stale' => false]))->weeklyOn(1, '02:00');
Schedule::command('reports:generate')->weekdays()->between('07:00', '20:00')->withoutOverlapping();
Schedule::command('model:prune')->daily();
Schedule::command('queue:prune-failed --hours=48')->daily();
```

Constraints chain expressively (`->daily()`, `->hourlyAt(17)`, `->cron('* * * * *')`, `->between()`, `->timezone()`, `->environments('production')`, `->onOneServer()` for Redis/Memcached locks, `->withoutOverlapping()`, `->runInBackground()`).

Run the whole schedule from a single system cron (the standard production setup):

```shell
* * * * * cd /path-to-app && php artisan schedule:run >> /dev/null 2>&1
```

Then `php artisan schedule:list` shows every scheduled task, and `schedule:interrupt` cancels in-progress runs on demand. Testing tip: `Schedule::fake()` (Pest/PHPUnit) plus `Schedule::assertScheduled()` lets you assert scheduling without waiting around.

## Optimization and caches

```shell
php artisan optimize          # config + routes + views + events (the deploy-time command)
php artisan optimize:clear    # clear all of them
php artisan config:cache      # config only
php artisan route:cache       # routes only (controller routes only — closures can't cache)
php artisan view:cache        # precompile Blade
php artisan event:cache
```

Cache **only in production** — locally these caches silently hide your edits (the classic "my route change isn't working" hour). Laravel Cloud/Forge deploy pipelines run `optimize` for you.

Publish vendor assets and config on demand:

```shell
php artisan vendor:publish                     # interactive picker
php artisan vendor:publish --tag=laravel-errors
php artisan config:publish cors
php artisan lang:publish
```

## Tinker

```shell
php artisan tinker
```

REPL with your booted app: `User::count()`, `User::factory()->make()`, `route('posts.show', 1)`, `config('app.url')`. Indispensable for sanity-checking a query or piece of state without writing a scratch route. In Tinker, `dump()`/`dd()` print richly, and multi-line pasting works; exit with `exit`. For scripted use: `php artisan tinker --execute='User::count();'` (careful with quoting) — or write a real command if it's going to run twice.
