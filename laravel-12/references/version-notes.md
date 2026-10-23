# Laravel 12 Version Notes

Verified, version-specific facts for Laravel 12 with primary sources. Consult this before claiming anything is "new" or "changed" in 12, and before advising an upgrade.

## Table of contents

- [Release and support timeline](#release-and-support-timeline)
- [Platform requirements](#platform-requirements)
- [What Laravel 12 is (and is not)](#what-laravel-12-is-and-is-not)
- [New in 12: starter kits](#new-in-12-starter-kits)
- [Breaking changes upgrading 11 → 12](#breaking-changes-upgrading-11--12)
- [12 vs 13: what NOT to use](#12-vs-13-what-not-to-use)
- [Sources](#sources)

## Release and support timeline

| Version | PHP | Released | Bug fixes until | Security fixes until |
|---|---|---|---|---|
| 10 | 8.1 – 8.3 | Feb 14, 2023 | Aug 6, 2024 | Feb 4, 2025 |
| 11 | 8.2 – 8.4 | Mar 12, 2024 | Sep 3, 2025 | Mar 12, 2026 |
| **12** | **8.2 – 8.5** | **Feb 24, 2025** | **Aug 13, 2026** | **Feb 24, 2027** |
| 13 | 8.3 – 8.5 | Q1 2026 | Q3 2027 | Q1 2028 |

Policy: 18 months of bug fixes and 2 years of security fixes for every major. As of October 2026, Laravel 12 receives security fixes only; Laravel 13 is current. Source: https://laravel.com/docs/12.x/releases

## Platform requirements

- PHP 8.2 – 8.5 (Laravel 11 topped out at 8.4; Laravel 13 requires 8.3+).
- Composer dependencies for a 12 app: `laravel/framework: ^12.0`, `phpunit/phpunit: ^11.0`, `pestphp/pest: ^3.0`, **Carbon 3 required** (Carbon 2 support removed).
- SQLite is the default database in a fresh app (`database/database.sqlite` is created and migrated during `laravel new`).

## What Laravel 12 is (and is not)

Laravel 12 is a **maintenance release**: upstream dependency updates plus quality-of-life improvements shipped throughout the year, with breaking changes deliberately minimized. Most Laravel 11 applications upgrade **without changing application code**; the official estimate is a 5-minute upgrade. The 11.x streamlined skeleton (slim config, `bootstrap/app.php` fluent configuration, no HTTP Kernel, no Console Kernel) carries over unchanged — so nearly all Laravel 11 guidance about structure and configuration also applies to 12.

Practical consequences when working in a 12 codebase:

- Do not introduce `app/Http/Kernel.php`-style patterns from Laravel ≤10 tutorials.
- Quality-of-life features shipped *during* the 11.x and 12.x cycles (not at the 12.0 boundary) are generally present — e.g. response-based rate limiting (`Limit::after()`), `Lottery`, exception throttling. Verify against 12.x docs if unsure.

## New in 12: starter kits

The headline feature. New application starter kits replace Breeze/Jetstream (both now frozen — no further updates):

| Kit | Stack |
|---|---|
| React | Inertia 2, React 19, TypeScript, Tailwind (v4), shadcn/ui |
| Svelte | Inertia 2, Svelte 5, TypeScript, Tailwind, shadcn-svelte |
| Vue | Inertia 2, Vue 3 Composition API, TypeScript, Tailwind, shadcn-vue |
| Livewire | Livewire (v4 line), Tailwind, Flux UI component library, Volt |

- All kits use **Laravel Fortify** for authentication (login, registration, password reset, email verification, 2FA enabled by default) and ship all source in the app for full ownership — there is no "upgrade the kit" path, you own the code.
- **WorkOS AuthKit** variants of each kit add social auth (Google, Microsoft, GitHub, Apple), passkeys, Magic Auth, and SSO; free up to 1M monthly active users. Chosen at `laravel new` time; configured via `WORKOS_CLIENT_ID`, `WORKOS_API_KEY`, `WORKOS_REDIRECT_URL`.
- JS kits use **Wayfinder** for type-safe, generated route definitions — removing references to routes you disabled breaks the build (expected behavior).

Details and customization: `references/starter-kits-frontend.md` and https://laravel.com/docs/12.x/starter-kits

## Breaking changes upgrading 11 → 12

Full guide: https://laravel.com/docs/12.x/upgrade — summary by impact:

**Medium**

- `HasUuids` now generates **UUIDv7** (lexicographically sortable, index-friendly). To keep old behavior use `Illuminate\Database\Eloquent\Concerns\HasVersion4Uuids` (alias it as `HasUuids` if you want a one-line swap). `HasVersion7Uuids` was removed — use `HasUuids`.

**Low**

- **Carbon 3 required.**
- **`image` validation rule no longer accepts SVGs.** Re-enable per-field with `'photo' => 'required|image:allow_svg'` or `File::image(allowSvg: true)`.
- **Local filesystem disk root** defaults to `storage/app/private` (was `storage/app`) when no explicit `local` disk is configured. `Storage::disk('local')` paths change accordingly; define the disk manually to restore the old root.
- **Container now respects constructor property defaults**: `resolve(Example::class)` for `public ?Carbon $date = null` now yields `null`, not an injected `Carbon`.
- **`Schema::getTables()/getViews()/getTypes()`** return all schemas by default; pass `schema: 'main'` (string or array) to scope. `Schema::getTableListing()` returns schema-qualified names unless `schemaQualified: false`.
- **Route name collisions**: uncached routing now matches the **first** route registered with a given name (previously the last) — unified with cached behavior.
- **`Concurrency::run`** with an associative array now returns results keyed by those names.
- **`mergeIfMissing()`** supports nested dot-notation keys (e.g. `'user.last_name'`).
- **`Blueprint` and `Grammar` constructors** now require an `Illuminate\Database\Connection` instance (package-maintainer concern). `Grammar::setConnection()` and `Connection::withTablePrefix()` were removed; table prefixes come from `$connection->getTablePrefix()`.
- **`DatabaseTokenRepository`** (password resets) `$expires` constructor param is now in **seconds**, not minutes.
- PHPUnit ^11 and Pest ^3 are required (mostly transparent; some assertion/test-case renames apply when upgrading very old suites).

## 12 vs 13: what NOT to use

Laravel 13 (Q1 2026) requires PHP 8.3+ and is a larger release. When working in a 12 codebase, do not pull in 13-only APIs or suggest `composer require laravel/framework:^13`. If the user asks about upgrading, the 12→13 constraint is the PHP 8.3 floor plus whatever the 13.x upgrade guide lists (https://laravel.com/docs/13.x/upgrade) — recommend reading it before committing to a plan.

## Sources

- Release notes & support policy: https://laravel.com/docs/12.x/releases
- Upgrade guide 11 → 12: https://laravel.com/docs/12.x/upgrade
- Installation (PHP support, SQLite default, Boost): https://laravel.com/docs/12.x/installation
- Starter kits: https://laravel.com/docs/12.x/starter-kits
- Frontend: https://laravel.com/docs/12.x/frontend
- Eloquent (UUIDv7 default): https://laravel.com/docs/12.x/eloquent
- Middleware (fluent API): https://laravel.com/docs/12.x/middleware
- Routing: https://laravel.com/docs/12.x/routing
- Testing: https://laravel.com/docs/12.x/testing
- Error handling: https://laravel.com/docs/12.x/errors
- Sanctum: https://laravel.com/docs/12.x/sanctum
