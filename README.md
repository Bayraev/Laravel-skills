# Laravel-skills

Agent skills for Laravel development, one per major version — install only the version your project uses. All content is grounded in the [official Laravel docs](https://laravel.com/framework/docs/) (versioned pages fetched Oct 2026), and each skill's `references/version-notes.md` records the source URL for every version-specific claim.

## Skills

| Skill | For | Key traits |
|---|---|---|
| [`laravel-10`](laravel-10/) | Laravel 10.x (PHP 8.1+) | Classic skeleton: `app/Http/Kernel.php` middleware, exception Handler, Process facade |
| [`laravel-11`](laravel-11/) | Laravel 11.x (PHP 8.2+) | Slim skeleton, fluent middleware API in `bootstrap/app.php`, SQLite default, health route |
| [`laravel-12`](laravel-12/) | Laravel 12.x (PHP 8.2+) | 11's foundation + new React/Vue/Svelte/Livewire starter kits, UUIDv7, Carbon 3 |
| [`laravel-13`](laravel-13/) | Laravel 13.x (PHP 8.3+) | Current major: AI SDK, JSON:API resources, `PreventRequestForgery`, attribute-first declarations, vector search |

Each skill is self-contained:

- `SKILL.md` — hub: version snapshot, idioms to unlearn (from other versions), workflow, decision table
- `references/` — 12–14 topic files loaded on demand: structure, middleware, routing, validation, Eloquent, migrations, Blade, queues, testing, Artisan, errors/logging, version notes (+ `process-service` for 10, `auth-basics` for 11, `starter-kits-frontend` for 12, `ai-tooling` for 13)

## Install (pick your version)

```bash
ln -s "$(pwd)/laravel-12" ~/.claude/skills/laravel-12
```

Each skill's description tells the agent to check `composer.json` (`laravel/framework ^N.0`) before applying conventions, so installing the right version keeps the agent on the correct skeleton/APIs for your app.

All PHP samples pass `php -l`; every skill passes [skill-creator](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator/skills/skill-creator)'s `quick_validate` check.
