# Laravel 11 Artisan, Scheduling, and the Health Route

Sources: https://laravel.com/framework/docs/11.x/artisan and
https://laravel.com/framework/docs/11.x/releases.

## Everyday commands

```shell
php artisan list                     # all commands
php artisan help migrate             # one command's options
php artisan about                    # environment/config overview — first stop when diagnosing
php artisan route:list -v            # routes + middleware
php artisan migrate --pretend        # SQL preview
php artisan tinker                   # REPL into the booted app
```

Laravel 11 added plain class generators (no need to hand-write boilerplate):

```shell
php artisan make:class
php artisan make:enum
php artisan make:interface
php artisan make:trait
```

Plus the classics: `make:controller`, `make:model`, `make:middleware`, `make:request`,
`make:job`, `make:command`, `make:event`, `make:listener`, `make:mail`, `make:notification`,
`make:policy`, `make:rule`, `make:seeder`, `make:factory`, `make:test`, `make:migration`.

First-party installers (prefer these over manual wiring):

```shell
php artisan install:api              # routes/api.php + Sanctum + api routing wiring
php artisan install:broadcasting     # routes/channels.php + broadcasting wiring
php artisan lang:publish             # lang/ directory for validation message customization
php artisan config:publish cors      # publish one vendor config file you need to change
php artisan storage:link             # public/storage -> storage/app/public
```

## Writing commands

Class commands live in `app/Console/Commands` (auto-registered):

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Console\Command;

class SendEmails extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--queue}';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Send a marketing email to a user';

    /**
     * Execute the console command.
     */
    public function handle(DripEmailer $drip): void
    {
        $drip->send(User::find($this->argument('user')));
    }
}
```

Signature syntax: `{user}` required, `{user?}` optional, `{user=foo}` default, `{user*}` array,
`{--queue}` boolean switch, `{--queue=}` value option, `{--Q|queue}` shortcut. Keep commands
thin — they parse input and delegate to service classes.

Exit codes: return `int` from `handle()` or call `$this->fail('message')` to exit with 1.

Register extra command directories/classes in `bootstrap/app.php`:

```php
->withCommands([
    __DIR__.'/../app/Domain/Orders/Commands',
])
```

## Closure commands and the schedule — `routes/console.php`

There is no console Kernel in Laravel 11. Closure commands and the schedule both live in
`routes/console.php`:

```php
use Illuminate\Foundation\Inspiring;
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Schedule;

Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
})->purpose('Send a marketing email to a user');

// The Laravel 11 scheduling style — Schedule facade, no Kernel:
Schedule::command('emails:send')->daily();
Schedule::command('sanctum:prune-expired --hours=24')->daily();
Schedule::job(new GenerateReport)->mondays()->at('09:00');
Schedule::call(fn () => DB::table('sessions')->where('last_activity', '<', now()->subDays(7)->timestamp)->delete())->hourly();
```

The scheduler still needs a single cron entry on the server:

```cron
* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1
```

## Command I/O and prompts

```php
$userId = $this->argument('user');
$queue = $this->option('queue');

$this->info('Done.');            // green
$this->error('Broken!');         // red
$this->line('plain');            // no color
$this->newLine(3);

$this->table(['Name', 'Email'], User::all(['name', 'email'])->toArray());

$this->withProgressBar($users, function (User $user) {
    $this->performTask($user);
});
```

Prompting (Laravel Prompts under the hood — placeholders, validation, search):

```php
use function Laravel\Prompts\text;
use function Laravel\Prompts\confirm;
use function Laravel\Prompts\search;

$name = text(label: 'What is your name?', validate: ['name' => 'required|min:3|max:255']);
$ok = confirm(label: 'Proceed?', default: true);
$userId = search(label: 'Search users', options: fn (string $v) => /* [...] */);
```

Prompt validation accepting full Laravel validator rules is new in 11.

Implement `Illuminate\Contracts\Console\PromptsForMissingInput` on a command to auto-prompt for
missing required arguments; customize the questions with `promptForMissingArgumentsUsing(): array`.

## Isolatable commands

```php
class SendEmails extends Command implements Isolatable
{
    //
}
```

Then `php artisan mail:send 1 --isolated` acquires an atomic cache lock so only one instance runs
at a time (needs a lock-capable cache driver). `isolatableId()` customizes the lock key.

## Tinker

```shell
php artisan tinker
```

REPL with full app booted — models, queues, config, container. Caveat: dispatch jobs via
`Bus::dispatch(new Job)` or `Queue::push(...)` inside Tinker, not `dispatch()`/`Job::dispatch()`,
because those rely on garbage collection that behaves differently in the REPL.

## Deployment caching

```shell
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
# or, the standard deploy shortcut that caches them for you:
php artisan optimize

php artisan optimize:clear      # clear all caches
```

Run at deploy time only. After `config:cache`, `env()` returns null outside config files —
another reason to read values via `config()`.

## Health route (`/up`)

Defined by the `health: '/up'` directive in `bootstrap/app.php`'s `withRouting`. Returns 200 when
the app boots; uptime monitors and orchestrators poll it. Add custom checks by listening to the
`DiagnosingHealth` event (fired on each request to the route) — throw in a listener to fail the
check. Don't remove the directive; monitoring platforms and Laravel Cloud conventions expect it.
