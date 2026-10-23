# Process Service (new in Laravel 10)

Source: [processes](https://laravel.com/framework/docs/10.x/processes), [releases](https://laravel.com/framework/docs/10.x/releases). The `Process` facade is Laravel 10's headline feature — a testable abstraction over `symfony/process` for invoking external programs. Before reaching for `exec()`, `shell_exec()`, or `Symfony\Component\Process\Process` directly, use this; it exists precisely to replace those with something mockable.

## Synchronous processes

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

$result->successful();      // exit code === 0
$result->failed();          // exit code > 0
$result->exitCode();        // int
$result->output();          // stdout
$result->errorOutput();     // stderr

// Fail loudly instead of checking manually
$result = Process::run('ls -la')->throw();          // ProcessFailedException when exit code > 0
$result = Process::run('ls -la')->throwIf($condition);
```

Real-time output streaming via closure:

```php
$result = Process::run('import.sh', function (string $type, string $output) {
    // $type: 'stdout' | 'stderr'
    echo $output;
});
```

## Options (chainable before `run`/`start`)

```php
Process::path(base_path())->run('ls -la');                    // working directory
Process::input($payload)->run('cat');                         // stdin
Process::timeout(120)->run('bash import.sh');                 // default: 60s → ProcessTimedOutException
Process::forever()->run('bash long-import.sh');               // no timeout
Process::timeout(60)->idleTimeout(30)->run('streaming.sh');   // max seconds without output
Process::env(['IMPORT_PATH' => __DIR__])->run('bash import.sh');
Process::env(['PASSWORD' => false])->run('deploy.sh');        // false removes an inherited var
Process::forever()->tty()->run('vim');                        // interactive TTY mode
```

The default timeout is 60 seconds; anything longer-lived needs an explicit `timeout()`/`forever()` call.

## Asynchronous processes

```php
$process = Process::start('npm run build');

$process->id();                      // PID
$process->running();
$process->signal(SIGTERM)->stop();   // send a signal

$output = $process->output();        // gathered so far
$error  = $process->errorOutput();
$result = $process->wait();          // block until done → ProcessResult
```

## Pools — concurrent execution

```php
use Illuminate\Process\Pool;
use Illuminate\Support\Facades\Process;

[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->command('cat first.txt');
    $pool->command('cat second.txt');
    $pool->command('cat third.txt');
});

return $first->output();
```

Named pool results (results keyed by name, so ordering doesn't matter):

```php
$results = Process::pool()
    ->name('first')->command('cat first.txt')
    ->name('second')->command('cat second.txt')
    ->start();

$results['first']->wait()->output();
```

Pool options: set once on the pool, inherited by every command — `Process::pool(fn (Pool $pool) => ...)->timeout(60)->start()`. Use pools when shelling out is unavoidable and jobs would be overkill (asset builds, batch file transforms); for anything user-facing, prefer queued jobs — pools still occupy the PHP process.

## Testing processes

```php
use Illuminate\Support\Facades\Process;

Process::fake();

// Run code that shells out...
Process::assertRan('ls -la');
Process::assertRan(fn ($process, $result) => $process->command === 'ls -la');
Process::assertNotRan('rm -rf /');
Process::assertRanTimes('ls -la', 2);

// Queue specific responses for specific commands
Process::fake([
    'git:*' => Process::result(exitCode: 0, output: 'ok'),   // wildcards allowed
    'cat *' => Process::result(errorOutput: 'missing'),
]);
```

`Process::fake()` with no arguments queues a "fake successful result" for anything not explicitly mapped. This is the reason to use the facade over raw Symfony — tests never spawn real processes, so a suite invoking external tools stays fast and deterministic.
