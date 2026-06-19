---
description: Remove um worktree e seus recursos (branch + bancos dedicados) com salvaguardas
argument-hint: <pasta-ou-branch>   # ex.: rumly-crm-feature-20260619-teste-worktree
---

Invoque a skill **`laravel-worktree`** e execute a **Operação C — LIMPAR**.

Alvo: `$ARGUMENTS` (pasta do worktree ou nome da branch). Se ausente, primeiro rode a Operação B (inspecionar) e pergunte qual remover.

**Confirmar antes de qualquer passo destrutivo** (remover worktree, dropar bancos, `--force`). Não remover trabalho não commitado/não enviado sem confirmação explícita.
