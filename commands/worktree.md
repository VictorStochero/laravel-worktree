---
description: Provisiona rápido um worktree Laravel/Herd isolado (ambiente paralelo completo)
argument-hint: <flag> <contexto>   # ex.: feature pagamentos-pix
---

Invoque a skill **`laravel-worktree`** e execute a **Operação A — CRIAR** seguindo-a à risca (detecte o runtime no Passo 0).

Argumentos: `$ARGUMENTS`
- `$1` = flag (feature|fix|hotfix|chore|docs|test|refactor) — validar; se inválida/ausente, parar e perguntar.
- restante = contexto (normalizar para kebab-case) — se ausente, parar e perguntar.

A skill é a fonte da verdade do procedimento (detecção, base remota, `.env`, banco, verificação da suíte). O objetivo é provisionamento rápido: **não** rodar a suíte inteira — apenas verificar que ela está descobrível/saudável. Não duplicar as etapas aqui.
