---
name: laravel-worktree
description: Use when creating, inspecting, or cleaning up git worktrees for a Laravel project, across local runtimes — Laravel Herd, DDEV, or Sail. Detects the runtime and branches the setup accordingly (shell, domain, database, container lifecycle). Covers isolated parallel environments end-to-end — base-branch derivation, dedicated database + .env, multi-tenancy/Warden/test-env quirks, baseline tests, and safe cleanup. Triggers: "criar worktree", "branch isolada", "ambiente paralelo", "novo worktree", "listar/inspecionar worktrees", "limpar/remover worktree".
---

# Laravel Worktree (multi-runtime)

Cria e opera git worktrees isolados para projetos Laravel — um ambiente paralelo completo por branch, sem tocar no checkout original. O **runtime local** (Herd / DDEV / Sail) muda só os detalhes de shell, domínio, banco e ciclo de container; o esqueleto (git worktree, base branch, `.env`, baseline) é comum.

Três operações: **criar**, **inspecionar**, **limpar**.

> **Validação:** o branch **Herd** foi exercitado ponta a ponta. Os branches **DDEV** e **Sail** seguem o comportamento documentado dessas ferramentas — confirmar o primeiro uso em cada um.

---

## Passo 0 — Detectar o runtime

Decidir o runtime ativo (primeira condição que casar; em ambiguidade, **perguntar**):
- **DDEV** → existe `.ddev/config.yaml` no repo.
- **Sail** → existe `vendor/bin/sail` (ou `./vendor/bin/sail`) e um `docker-compose.yml`/`compose.yaml` referenciando `laravel/sail`, **e** não há `.ddev/`.
- **Herd** → caso contrário (PHP no host; macOS/Windows com Herd). É o default.

Cada runtime define quatro coisas, usadas pelas operações abaixo:

| | **Herd** | **DDEV** | **Sail** |
|---|---|---|---|
| **Shell p/ php/composer/npm** | PowerShell (Windows) / shell do host. **Não** Git Bash no Windows | `ddev composer`, `ddev artisan`, `ddev exec` | `./vendor/bin/sail composer`, `sail artisan` |
| **Domínio** | `{pasta}.test` | `{pasta}.ddev.site` | `http://localhost:{APP_PORT}` (porta única por worktree) |
| **Servir** | pasta na raiz parkeada do Herd (auto) | `ddev config` + `ddev start` na pasta | `sail up -d` (compose project único) |
| **Banco** | DB dedicado no host (`127.0.0.1`, criar/dropar manualmente) | isolado por projeto DDEV (auto — não criar à mão) | isolado no compose (volume `-v` no teardown) |

Sempre via **Bash**: os comandos `git`.

---

## Conceitos comuns (todos os runtimes)

- **Entradas:** `flag` ∈ { feature, fix, hotfix, chore, docs, test, refactor }; `contexto` em kebab-case; `data` = hoje `AAAAMMDD`; `repo` = nome do diretório do repositório.
- **Pasta:** `{repo}-{flag}-{AAAAMMDD}-{contexto}`. **Branch:** `{flag}/{AAAAMMDD}-{contexto}`.
- **Não aninhar:** se `git rev-parse --git-dir` ≠ `--git-common-dir` e não for submódulo (`git rev-parse --show-superproject-working-tree` vazio), já se está num worktree — parar.
- **Base remota:** `git fetch`; derivar a base na **primeira que existir** em `origin/`: `desenvolvimento` → `dev` → `develop` → `main`. Gravar em `{pasta}/.worktree-base`.
- **`.env` (chaves presentes apenas):** `APP_URL` → domínio do runtime; multi-tenant (detectar `config/tenancy.php` ou `stancl/tenancy`) → `CENTRAL_DOMAIN`/`TENANT_BASE_DOMAIN` = domínio, `SESSION_DOMAIN` = `.{domínio-base}`; prefixos únicos (`REDIS_PREFIX`/`CACHE_PREFIX` ou `REDIS_DB` distinto); Warden child (detectar `config/warden.php` + `WARDEN_MODE=child`) → `WARDEN_ENABLED=false`; **manter `APP_KEY`**.
- **Localização:** worktree sempre **fora** do repo. No Herd, pasta-irmã na raiz parkeada. Nunca usar ferramenta de worktree nativa que ignore o setup do runtime (domínio/`.env`/DB).

---

## Operação A — CRIAR

1. **Detectar runtime** (Passo 0) e **não aninhar**.
2. **Base remota** + `git worktree add <destino> -b {branch} origin/{base}`; gravar `.worktree-base`.
   - Herd: `<destino>` = `<raiz_herd>/{pasta}`.
   - DDEV/Sail: `<destino>` pode ser pasta-irmã qualquer fora do repo.
3. **`.env`:** copiar da origem e aplicar os ajustes comuns. **Banco por runtime:**
   - **Herd:** `DB_DATABASE` (e `TENANCY_DB_DATABASE` se existir) → `{db}_wt_{contexto}`.
   - **DDEV/Sail:** banco é isolado pelo container — **não** apontar para um DB compartilhado; manter o esquema do runtime (DDEV injeta `db`; Sail usa o serviço do compose). Garantir `COMPOSE_PROJECT_NAME`/porta únicos no Sail (`APP_PORT`, `FORWARD_DB_PORT`) para não colidir.
4. **Subir o ambiente do runtime:**
   - **Herd:** nada a subir (a pasta já é servida). Criar o DB dedicado (PDO via `.php` temporário → `CREATE DATABASE IF NOT EXISTS`, apagar o temp).
   - **DDEV:** `ddev config --project-name={pasta} --project-type=laravel --docroot=public` (ou copiar `.ddev/config.yaml` e trocar o `name`) → `ddev start`.
   - **Sail:** garantir `vendor/` (passo 5) e `sail up -d`.
5. **Dependências:**
   - **Herd:** `composer install`. Em Windows, se falhar por `ext-pcntl`/`ext-posix` (Horizon e libs Unix-only), repetir com `--ignore-platform-req=ext-pcntl --ignore-platform-req=ext-posix` (+ as extensões reportadas).
   - **DDEV:** `ddev composer install` (container tem pcntl/posix — sem flags).
   - **Sail:** `composer install` no host para obter `vendor/bin/sail`, depois `sail composer install` se preferir paridade de container.
6. **Migrar:** `migrate --seed` pelo shell do runtime (**limpo + seed — não clonar dados**). Herd: `php artisan migrate --seed`. DDEV: `ddev artisan migrate --seed`. Sail: `sail artisan migrate --seed`.
7. **`storage:link`** pelo shell do runtime.
8. **Front-end:** se houver `package.json`, `npm install` (Herd) / `ddev npm install` / `sail npm install` (+ build se necessário).
9. **Ambiente de teste:** se houver `.env.testing` versionado, alinhar o host de DB ao runtime ativo. Um `.env.testing` apontando para `db`/`mysql` (DDEV/Docker) **quebra no Herd** (`getaddrinfo for db failed`) e vice-versa. Se existir um exemplo do runtime (`.env.testing.example-herd`, `.env.testing.example-ddev`, etc.), sobrepor o correto como `.env.testing` e usar um banco de teste isolado (`{db}_wt_{contexto}_test` no Herd; o do container em DDEV/Sail); migrar com `migrate --env=testing`.
10. **Baseline verde:** rodar a suíte (`... artisan test --compact`) antes de iniciar. Se já parte vermelha, reportar e perguntar — **não confundir regressão nova com falha pré-existente** (URL/host hardcoded, ou teste que depende da suíte inteira ter migrado o banco de teste).

### Reportar (criação)
Pasta + **URL**; branch + base (`.worktree-base`); banco/projeto do runtime; status do baseline (verde / vermelho + causa).

---

## Operação B — INSPECIONAR
1. `git worktree list --porcelain` → pasta, branch, HEAD.
2. Enriquecer (best-effort): URL (pelo runtime + nome da pasta), base (`.worktree-base`), banco/projeto, e estado sujo (`git -C <pasta> status -sb`).
3. Tabela: pasta | branch | base | URL | banco/projeto | sujo?. Sinalizar órfãos (pasta sumiu, registro existe) → sugerir `git worktree prune`. Para DDEV, complementar com `ddev list`.

---

## Operação C — LIMPAR
**Confirmar antes de qualquer passo destrutivo.**
1. Identificar alvo (pasta/branch). Ler `{pasta}/.env` (banco) e `.worktree-base` **antes** de remover.
2. **Salvaguarda:** `git -C <pasta> status -sb` — havendo mudanças não commitadas/não enviadas, avisar e exigir confirmação explícita.
3. **Teardown do runtime (libera o banco isolado):**
   - **DDEV:** `ddev delete -O {pasta}` (remove projeto + volume do DB).
   - **Sail:** `sail down -v` na pasta (remove containers + volumes).
   - **Herd:** dropar os bancos dedicados (`_wt_` de dev e `_wt_..._test`) via `.php` temporário (`DROP DATABASE IF EXISTS`, apagar o temp).
4. `git worktree remove <pasta>` (`--force` só se necessário e confirmado). **No Windows**, o `remove` pode esvaziar o registro mas **deixar a pasta** (lock de arquivo) — conferir e apagar o residual com `Remove-Item -Recurse -Force <pasta>`.
5. `git branch -d <branch>` (`-D` só se não mergeada e confirmado).
6. `git worktree prune`.

### Reportar (limpeza)
O que foi removido (pasta, branch, banco/projeto) e o que foi preservado/abortado por salvaguarda.

---

## Pull Request
PR sempre com alvo na base de `.worktree-base` (nunca `main` se a origem foi `develop`/`dev`/`desenvolvimento`): `gh pr create --base <base> --head <branch>`.

## Red flags
- Worktree dentro do repo ou aninhado.
- Misturar shells/domínios de runtimes (ex.: `php artisan` no host num projeto DDEV; `.test` no Sail).
- Clonar dados da origem em vez de `migrate --seed` limpo.
- Declarar baseline verde sem rodar testes, ou tratar falha pré-existente como bloqueio.
- Dropar banco / `ddev delete` / `sail down -v` / `--force` sem confirmar mudanças não salvas.
