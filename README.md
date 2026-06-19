# laravel-worktree

A [Claude Code](https://claude.com/claude-code) plugin that creates, inspects, and cleans up **isolated git worktrees for Laravel projects** — each one a complete parallel environment (dedicated database, its own `.env`, its own local domain) so you can work on several branches at once without disturbing your main checkout.

It detects your local runtime — **Laravel Herd**, **DDEV**, or **Sail** — and adapts the shell, domain scheme, database handling, and container lifecycle accordingly.

> Status: the **Herd** path is validated end-to-end. The **DDEV** and **Sail** paths are written from each tool's documented behavior — confirm them on first use and please open an issue with any fixes.

## Why

Spinning up a throwaway environment for a second branch is fiddly: you clone or stash, copy `.env`, point it at a separate database, fix the domain, run migrations and seeders, wire up the test environment, and remember to tear it all down later. This plugin turns that into three commands and a single source of truth (the skill), with the Laravel/runtime-specific gotchas already handled.

## What it does

- Derives the base branch from your remote in order: `desenvolvimento` → `dev` → `develop` → `main`.
- Creates the worktree **outside** the repo, with a predictable folder/branch naming scheme.
- Copies and rewrites `.env`: app URL/domain, a **dedicated database**, unique queue/cache prefixes, multi-tenant domains (auto-detected), and Warden child telemetry off (auto-detected) — while keeping `APP_KEY`.
- Installs dependencies (with Windows `ext-pcntl`/`ext-posix` fallback for Horizon), runs a **clean `migrate --seed`** (never clones data), links storage, installs front-end deps.
- Aligns a committed `.env.testing` to the active runtime (a DDEV `DB_HOST=db` breaks on Herd, and vice-versa) and runs a **green baseline** before you start.
- Inspects all worktrees and **cleans them up safely** (drops the dedicated DB / `ddev delete` / `sail down -v`, removes the worktree, branch, and stray folder), with guards against discarding uncommitted work.

## Supported runtimes

| Runtime | Shell | Domain | Database | Status |
|---|---|---|---|---|
| **Herd** | host (`php artisan`, PowerShell on Windows) | `{folder}.test` | dedicated DB on host, created/dropped explicitly | ✅ validated |
| **DDEV** | `ddev artisan` / `ddev composer` | `{folder}.ddev.site` | isolated per DDEV project | 📝 documented |
| **Sail** | `./vendor/bin/sail` | `http://localhost:{APP_PORT}` | isolated compose volume | 📝 documented |

## Install

### As a plugin (recommended)

In Claude Code, add the marketplace once and install:

```
/plugin marketplace add VictorStochero/claude-plugins
/plugin install laravel-worktree@stochero
```

This installs the `laravel-worktree` skill (auto-activates on relevant requests) and the three slash commands. The same marketplace also serves any future plugins.

### Manual

Copy the pieces into your Claude config:

- `skills/laravel-worktree/` → `~/.claude/skills/laravel-worktree/`
- `commands/*.md` → `~/.claude/commands/`

## Commands

| Command | What it does |
|---|---|
| `/worktree <flag> <context>` | Create a worktree + full environment. `flag` ∈ `feature\|fix\|hotfix\|chore\|docs\|test\|refactor`. Example: `/worktree feature payments-pix` |
| `/worktree-list` | List worktrees with branch, base, URL, database, and dirty state |
| `/worktree-clean <folder-or-branch>` | Tear a worktree down (environment + branch + database) with safety prompts |

You can also just ask in natural language ("create an isolated branch to test X") — the skill activates on its own.

## Conventions

For `/worktree feature payments-pix` in a repo named `acme`, on `2026-06-19`:

- **Folder:** `acme-feature-20260619-payments-pix` → served at `acme-feature-20260619-payments-pix.test` (Herd)
- **Branch:** `feature/20260619-payments-pix`, created from `origin/<derived-base>`
- **Base** recorded in `.worktree-base` (used as the PR target)

## How it works

The plugin's behavior lives in a single skill (`skills/laravel-worktree/SKILL.md`) with three operations — **Create**, **Inspect**, **Clean** — and a runtime-detection step that branches the runtime-specific details. The slash commands are thin entry points that invoke the skill, so there is no duplicated procedure to keep in sync.

## Caveats

- DDEV/Sail paths are unverified on real environments yet — treat the first run as a dry run and report issues.
- The skill prompts are written in Brazilian Portuguese (the author's working language); the workflow itself is language-agnostic.
- Designed around a `dev`-style integration branch and `.test`/`.ddev.site` local domains; adjust the skill if your team's flow differs.

## Contributing

Issues and PRs welcome — especially real-world fixes for the DDEV and Sail paths, and additional runtime support.

## License

MIT © Victor Stochero
