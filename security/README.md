# `security/` — a Camada 0, isolada

Esta pasta é a **casa canônica da segurança** do projeto. Princípio: **segurança vive separada do resto** — histórico, testes e auditorias não se misturam com o código da app, para poderem ser lidos, versionados e auditados por conta própria.

> Para estabelecer/manter esta camada num projeto, rode o comando **`/security-layer`**.

## O que mora aqui
| Pasta | Propósito |
|---|---|
| `CHANGELOG-SECURITY.md` | **Append-only** — toda mudança de segurança, datada |
| `protocols/` | A doutrina (princípios + regras invioláveis) |
| `threat-model/` | O modelo de ameaça deste projeto |
| `reviews/` | Cada revisão/auditoria, datada e arquivada |
| `relatorios/` | Relatórios de pentest/auditoria |
| `tests/` | ⭐ Testes de segurança — **suíte separada da app** |
| `playbooks/` | Playbooks e runbooks acionáveis |
| `checklists/` | GO-NO-GO, gate de PR |
| `incidents/` | Runbook DFIR + registro de incidentes |
| `evidencias/` | Evidências preservadas (hash + cadeia de custódia) |

## As 3 regras de isolamento
1. **Nada de segurança fora daqui.** Relatório, teste, evidência, runbook de segurança → sempre em `security/`.
2. **Testes de segurança rodam separados** da suíte da app (alvo/config próprio), para o resultado de segurança ser independente.
3. **Histórico append-only.** O `CHANGELOG-SECURITY.md` e as pastas `reviews/`/`relatorios/` só crescem — cada item datado, nunca sobrescrito.

O **time de agentes** (`.claude/agents/`) e as **skills** (`.claude/skills/`) também são isolados — são a config do Claude que opera esta camada, não código da aplicação.
