# AI Tooling — Boost, LSP, AI SDK, Vector Search

Laravel 13's headline theme is agent-readiness: the framework itself is designed for AI-assisted development, and it ships first-party AI building blocks. Two distinct concerns live here — (1) tooling that makes *agents building Laravel apps* effective, and (2) the AI SDK for building *AI features into your app*.

## 1. Agent tooling for Laravel development

### Laravel Boost (`laravel/boost`)

Boost is an MCP server bridging coding agents to a Laravel app: 15+ tools (introspection, read-only DB queries, route listing, Artisan discovery, log analysis, browser logs, Tinker execution, docs search), 17,000+ vectorized doc chunks version-filtered to your installed packages, and Laravel-maintained AI guidelines.

```bash
composer require laravel/boost --dev
php artisan boost:install        # interactive; auto-detects IDE/agents, writes .mcp.json etc.
```

- Supports Laravel 10–13 on PHP 8.1+. Generated `.mcp.json`, `CLAUDE.md`, `boost.json` are per-developer — safe to gitignore.
- Guidelines cover Laravel + 16 ecosystem packages (Livewire 2/3/4, Inertia React/Svelte/Vue, Tailwind 3/4, Filament 3/4, Pest, Pint, ...), assembled to match your `composer.json`.
- Agent Skills (agentskills.io) install on-demand knowledge modules for detected packages.
- Project-specific rules: drop `.md` or `.blade.md` files in `.ai/guidelines/*` — they merge into Boost's guidelines.
- Setup for Claude Code, Cursor, Codex, Gemini CLI, Copilot, Junie: https://laravel.com/framework/docs/boost

**When working in a Laravel repo, check whether Boost is installed (`composer.json` / `.mcp.json`) and prefer its tools over ad-hoc file grepping for schema, routes, and docs questions.**

### Laravel LSP

```bash
composer global require laravel/lsp
```

Framework-aware completions, diagnostics, go-to-definition, quick fixes for PHP + Blade. Official editor extensions: VS Code/Cursor (`laravel.vscode-laravel`), Sublime, Zed; Neovim and OpenCode setups in the laravel/lsp repo; PhpStorm has built-in Laravel support.

### The agent playbook — `https://laravel.com/for/agents`

A markdown installation/setup playbook served for agents. When scaffolding a brand-new Laravel app in an agent session, fetch and follow it rather than improvising. The docs' suggested user prompt pattern is exactly this: "Fetch and follow the instructions from https://laravel.com/for/agents."

## 2. Laravel AI SDK (`laravel/ai`) — building AI features

### Install

```bash
composer require laravel/ai
php artisan vendor:publish --provider="Laravel\Ai\AiServiceProvider"
php artisan migrate        # creates agent_conversations + agent_conversation_messages
```

Providers configured via `config/ai.php` / `.env` keys: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `MISTRAL_API_KEY`, `DEEPSEEK_API_KEY`, `OPENROUTER_API_KEY`, `OLLAMA_API_KEY`, `AZURE_OPENAI_API_KEY`, `COHERE_API_KEY`, `ELEVENLABS_API_KEY`, `JINA_API_KEY`, `TYPESAFE_API_KEY`, `VOYAGEAI_API_KEY`, `XAI_API_KEY`, plus generic `OPENAI_COMPATIBLE_API_KEY` + `OPENAI_COMPATIBLE_URL` for any OpenAI-shaped endpoint. Custom base URLs per provider for gateways/proxies.

### Agents

```bash
php artisan make:agent SalesCoach            # app/Ai/Agents/SalesCoach.php
php artisan make:agent SalesCoach --structured
```

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Promptable;
use Stringable;

class SalesCoach implements Agent, HasTools, HasStructuredOutput
{
    use Promptable;

    public function instructions(): Stringable|string
    {
        return 'You are a sales coach analyzing transcripts.';
    }

    public function tools(): iterable
    {
        return [new RetrievePreviousTranscripts];
    }

    public function schema(JsonSchema $schema): array      // structured output
    {
        return [
            'feedback' => $schema->string()->required(),
            'score' => $schema->integer()->min(1)->max(10)->required(),
        ];
    }
}
```

Prompting:

```php
use App\Ai\Agents\SalesCoach;

$response = SalesCoach::make(user: $user)->prompt('Analyze this sales transcript...');

return (string) $response;                                  // text
$structured = $response->json();                            // when schema defined
$response->raw->header('X-RateLimit-Remaining-Requests');   // raw provider HTTP response

// Overrides per call:
SalesCoach::make()->prompt('...', provider: Lab::Anthropic, model: 'claude-sonnet-5', timeout: 120);
```

Implement `Conversational` (+ `messages(): iterable`) for stored conversation context; the SDK's migrated `agent_conversations` tables power it. Also supported: streaming, broadcasting (Laravel Echo), queueing (`->queue()`), sub-agents, middleware, anonymous agents (fluent `Agent::new()->withInstructions(...)`), prompt caching, deferred tool loading, file-storage tools, **MCP tools** (mount external MCP servers), and human tool approval for risky tools.

### Tools

```bash
php artisan make:tool RandomNumberGenerator     # app/Ai/Tools
```

```php
use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Tool;
use Laravel\Ai\Tools\Request;
use Stringable;

class RandomNumberGenerator implements Tool
{
    public function description(): Stringable|string
    {
        return 'Generate cryptographically secure random numbers.';
    }

    public function handle(Request $request): Stringable|string
    {
        return (string) random_int($request['min'], $request['max']);
    }

    public function schema(JsonSchema $schema): array
    {
        return [
            'min' => $schema->integer()->min(0)->required(),
            'max' => $schema->integer()->required(),
        ];
    }
}
```

### Media, embeddings, classification

```php
use Illuminate\Support\Str;
use Laravel\Ai\Audio;
use Laravel\Ai\Embeddings;
use Laravel\Ai\Image;

$image = Image::of('A donut on a kitchen counter')->generate();   // (string) $image = raw bytes
$audio = Audio::of('I love coding with Laravel.')->generate();

$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings();
$response = Embeddings::for(['text one', 'text two'])
    ->dimensions(1536)
    ->generate(Lab::OpenAI, 'text-embedding-3-small');
$response->embeddings;        // [[0.123, ...], [0.789, ...]]
```

Reranking, classification (yes/no, choose-from-collection, image classification), transcriptions, vector stores (add files, query), usage tracking, and provider failover all ship in the SDK — see https://laravel.com/framework/docs/13.x/ai-sdk for the full API. **Do not guess provider model IDs or method signatures from memory — the SDK is new; check the page.**

## 3. Semantic / vector search (DB-native)

Prereqs: PostgreSQL + `pgvector` (or MariaDB 11.7+), and the AI SDK for string auto-embedding.

```php
// Migration
Schema::ensureVectorExtensionExists();
$table->vector('embedding', dimensions: 1536)->index();

// Query — string input is embedded automatically:
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley', minSimilarity: 0.4)
    ->limit(10)
    ->get();

// Manual control:
DB::table('documents')
    ->selectVectorDistance('embedding', $queryEmbedding, as: 'distance')
    ->whereVectorDistanceLessThan('embedding', $queryEmbedding, maxDistance: 0.3)
    ->orderByVectorDistance('embedding', $queryEmbedding);
```

Search strategy ladder (all built-in, no external services needed until you need typo tolerance/facets/geo at scale):
1. `LIKE` — substring match, no ranking.
2. `whereFullText(...)` — keyword relevance via native full-text indexes (MariaDB/MySQL/PostgreSQL); understands stemming.
3. `whereVectorSimilarTo(...)` — semantic match by meaning (needs pgvector + AI SDK).
4. AI SDK **reranking** — second stage over full-text candidates for semantic precision.
5. Laravel Scout (`Searchable` models, database engine or Algolia/Meilisearch/Typesense/Turbopuffer) — auto-synced indexes.

## Practices for agent-driven Laravel work

- Convention over configuration — place files where the framework expects (see `structure-conventions.md`); agents and tools both rely on it.
- Verify, don't assume: use `route:list`, `model:show`, `migrate --pretend`, and Tinker to confirm reality before writing code against it.
- Prefer the official docs (17k indexed chunks via Boost, or laravel.com/framework/docs/13.x) over training-data recall for anything attribute- or AI-SDK-related — these APIs are new in 13 and evolve.
- Keep AI keys in `.env`; never hardcode provider keys in `config/ai.php` commits.
