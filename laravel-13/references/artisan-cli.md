# Artisan CLI

## Everyday commands

```bash
# Generators (each creates the file in its conventional location)
php artisan make:model Flight -mfsc        # model + migration + factory + seeder + controller
php artisan make:model Flight --policy
php artisan make:controller PhotoController --model=Photo --resource --requests
php artisan make:request StorePostRequest
php artisan make:middleware EnsureTokenIsValid
php artisan make:job ProcessPodcast
php artisan make:event PodcastProcessed / make:listener SendPodcastNotification --event=PodcastProcessed
php artisan make:command SendDailyDigest
php artisan make:rule Uppercase / make:observer / make:policy / make:resource / make:seeder
php artisan make:component Alert / make:view authors.show
php artisan make:agent SalesCoach / make:tool RandomNumberGenerator   # AI SDK

# Introspection — use these before guessing
php artisan route:list [--except-vendor] [--path=api]
php artisan about                          # env, cache states, drivers at a glance
php artisan model:show Flight              # model columns, relations, casts, observers
php artisan config:show database           # resolved config values
php artisan tinker                         # REPL with the full app booted

# Database
php artisan migrate[:fresh|:rollback|:status] [--pretend] [--seed] [--isolated]
php artisan db:seed / db:show / db:table users / schema:dump --prune
php artisan migrate --force                # production

# Cache & optimization (deploy steps)
php artisan optimize                       # config + routes + views + events cache
php artisan optimize:clear
php artisan config:cache / route:cache / view:clear / event:cache
php artisan storage:link

# Maintenance
php artisan down --secret="maintenance-token" --render="errors::503" --retry=60
php artisan up
php artisan queue:restart / queue:failed / queue:retry all

# Install first-party features
php artisan install:api                    # routes/api.php + Sanctum
php artisan install:broadcasting           # Reverb + channels
php artisan boost:install                  # Laravel Boost (AI tooling)
php artisan lang:publish                   # lang/ directory
php artisan pail                           # tail logs live
```

Maintenance mode: `/up` health route (default in `bootstrap/app.php`) keeps deploy probes working.

## Writing commands

```bash
php artisan make:command SendDailyDigest
```

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;

class SendDailyDigest extends Command
{
    protected $signature = 'app:send-daily-digest
                            {--dry-run : Print what would be sent without sending}
                            {--queue= : Queue to dispatch digests on}';

    protected $description = 'Send the daily digest email to active subscribers';

    public function handle(): int
    {
        $users = User::query()->subscribed()->get();

        if ($this->option('dry-run')) {
            $this->table(['id', 'email'], $users->map(fn ($u) => [$u->id, $u->email])->all());

            return self::SUCCESS;
        }

        $users->each(fn ($user) => $user->notify(new DailyDigest));

        $this->info("Dispatched {$users->count()} digests.");

        return self::SUCCESS;
    }
}
```

Conventions: `app:` namespace prefix for app commands; return `self::SUCCESS`/`self::FAILURE` (int exit codes); use `$this->info/warn/error/line`, `$this->table($headers, $rows)`, `$this->ask(...)`, `$this->confirm(...)`, `$this->secret(...)`, progress bars via `$this->withProgressBar($items, fn ($item) => ...)`; inject services via `handle(Service $service)`.

Closure commands (small, glue-only) go in `routes/console.php` — this file also holds schedule definitions:

```php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Schedule;

Artisan::command('app:ping', function () {
    $this->info('pong');
})->purpose('Test command');

Schedule::command('app:send-daily-digest')->dailyAt('08:00')->timezone('America/Chicago');
Schedule::job(new ProcessPodcast($podcast))->everyMinute();
Schedule::call(fn () => /* ... */)->hourly();
```

Register class-based commands in `bootstrap/app.php` `->withCommands([...])` or under `app/Console/Commands` (auto-discovered).

## Scheduling

Run the scheduler every minute in production (cron: `* * * * * php /path/artisan schedule:run`, or Laravel Cloud/Forge handle it):

```php
Schedule::command('emails:send --force')->daily()->between('08:00', '17:00')->withoutOverlapping();
Schedule::command('report:generate')->weekly()->sundays()->at('02:00')->runInBackground();
Schedule::command('telescope:prune')->daily();
Schedule::command('backup:run')->everyFifteenMinutes()->evenInMaintenanceMode();
```

Modifiers: `->timezone()`, `->when(fn () => ...)`, `->environments(['production'])`, `->onOneServer()`, `->withoutOverlapping()`, `->withOverlappingTime(...)`, `->between()`, `->unlessBetween()`. Frequencies: `everySecond/Minute/FiveMinutes(...)/hourly/hourlyAt(17)/daily/dailyAt('13:00')/twiceDaily(1,13)/weekly/weeklyOn(1,'8:00')/monthly/quarterly/yearly/cron('* * * * *')`.

Test schedules: `php artisan schedule:list` (also `schedule:interrupt`).

## Stub customization

`php artisan stub:publish` copies framework stubs to `stubs/` — edit to change generated code style (e.g. add `#[Tries]` to job stubs for your team's defaults).
