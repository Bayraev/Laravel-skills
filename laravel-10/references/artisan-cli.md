# Artisan CLI

Source: [artisan](https://laravel.com/framework/docs/10.x/artisan). Laravel 10 niceties: every `make:` command prompts interactively when invoked without arguments, and `php artisan about` summarizes the whole environment.

## Daily drivers

```bash
php artisan about                     # environment overview: PHP version, cache states, drivers
php artisan about --only=drivers      # narrow the table
php artisan help migrate              # signature + flags for any command
php artisan list                      # everything available (incl. app commands)
```

## Generators (`make:*`)

| Command | Creates |
| --- | --- |
| `make:controller PhotoController --resource --model=Photo` | Controller (+typed model hints) |
| `make:controller ProvisionServer --invokable` | Single-action controller |
| `make:model Flight -a` (or `-m`, `-f`, `-s`, `-c`, `--policy`, `-p`) | Model + selected companions |
| `make:request StorePhotoRequest` | Form request |
| `make:middleware EnsureTokenIsValid` | Middleware class |
| `make:job ProcessPodcast` | Queue job |
| `make:command SendEmails` | Console command (prompts for name/signature in 10) |
| `make:migration add_status_to_photos_table --table=photos` | Migration |
| `make:seeder PhotoSeeder` / `make:factory` / `make:policy` / `make:rule` / `make:event` / `make:listener` / `make:provider` / `make:component` / `make:cast` / `make:channel` | The rest |

Flags compose: `make:model Photo -mfsc` = migration + factory + seeder + controller.

## Custom commands

```php
namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;

class SendEmails extends Command
{
    protected $signature = 'mail:send {user} {--queue}';   // {user} required, {user?} optional, {user=default}
    protected $description = 'Send a marketing email to a user';

    public function handle(): void
    {
        $user = User::find($this->argument('user'));

        if ($this->option('queue')) {
            // queue the mail...
        }

        $this->info("Email queued for {$user->email}.");
        $this->fail('Something went wrong.');   // exit code 1
    }
}
```

Commands auto-register from `app/Console/Commands` (10.x scans the directory). Interactive I/O inside commands: `$this->ask()`, `$this->anticipate()`, `$this->choice()`, `$this->confirm()`, `$this->table($headers, $rows)`, `$this->withProgressBar($items, fn ($item) => ...)`.

Closure commands belong in `routes/console.php`:

```php
use Illuminate\Foundation\Inspiring;

Artisan::command('inspire', function () {
    $this->comment(Inspiring::quote());
})->purpose('Display an inspiring quote');
```

## Scheduling (Laravel 10 location: Console Kernel)

`app/Console/Kernel.php` — the 10.x home for schedules (Laravel 11 moved this; keep it here):

```php
use App\Console\Kernel;
use App\Jobs\GenerateReport;
use Illuminate\Console\Scheduling\Schedule;

class MyKernel extends Kernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->command('emails:send --force')->dailyAt('08:00');
        $schedule->job(new GenerateReport)->hourly();
        $schedule->command('telescope:prune')->daily();
    }
}
```

Server cron: `* * * * * php /path/to/artisan schedule:run` (local alternative: `php artisan schedule:work`). Inspect with `php artisan schedule:list`.

## Cache and optimization

```bash
php artisan optimize                  # config + routes + views + events cached (production)
php artisan optimize:clear            # clears all caches — first move when deployed behavior looks stale
php artisan config:cache / config:clear
php artisan route:cache / route:clear      # route:cache fails on closure routes
php artisan view:cache / view:clear
php artisan event:cache / event:clear
```

During development, caches usually off; a stale route/view cache is the classic cause of "my change isn't showing."

## Maintenance and ops

```bash
php artisan down --retry=60                 # 503 maintenance page
php artisan down --secret="alpha-137"       # bypass at /alpha-137 (cookie unlock)
php artisan up
php artisan migrate --pretend               # print SQL without running
php artisan storage:link                    # public/storage → storage/app/public
php artisan queue:restart                   # after every deploy (workers pick up new code)
php artisan route:list                      # route table — run after any route change
php artisan route:list --path=api --except-vendor
php artisan db:show / db:table users        # introspection
```

## Tinker (REPL)

```bash
php artisan tinker
```

```php
$user = App\Models\User::first();
$user->update(['name' => 'Ali']);
factory(App\Models\User::class)->count(3)->create();   // or User::factory()->count(3)->create() in 10
dispatch(new App\Jobs\ProcessPodcast($podcast));
```

Tinker allow-lists commands (`config/tinker.php`: `clear-compiled`, `down`, `env`, `inspire`, `migrate`, `optimize`, `up` by default). Prefer `php artisan tinker --execute="User::count();"` for one-shot scripts in automation.
