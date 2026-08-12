---
description: Estabelece/mantém a camada de segurança ISOLADA (pasta security/ separada) — histórico, testes e auditorias fora do resto do projeto.
---

Você vai estabelecer (ou manter) a **camada de segurança isolada** deste projeto. Princípio inegociável: **tudo de segurança vive numa pasta própria, separada do código da app** — histórico, testes e auditorias **não se misturam** com o resto. É a Camada 0 com casa própria e história independente.

## Estrutura canônica (crie o que faltar; cada pasta com um README curto do propósito)
```
security/
├── CHANGELOG-SECURITY.md   # APPEND-ONLY: toda mudança de segurança, datada
├── protocols/              # a doutrina (vem do baseline)
├── threat-model/           # THREAT-MODEL.md deste projeto
├── reviews/                # cada revisão/auditoria, datada e arquivada (nunca sobrescreve)
├── relatorios/             # relatórios de pentest/auditoria (datados)
├── tests/                  # ⭐ testes de SEGURANÇA — suíte SEPARADA da app
├── playbooks/              # auth-attack-defense, runbooks
├── checklists/             # GO-NO-GO, gate de PR
├── incidents/              # runbook DFIR + registro de incidentes (datado)
└── evidencias/             # evidências preservadas (cadeia de custódia; hash)
```
O time (`.claude/agents/`) e as skills forjadas (`.claude/skills/`) também são isolados — config do Claude, não código da app.

## Regras que você mantém SEMPRE
1. **Nada de segurança fora de `security/`.** Não espalhe relatório, teste, evidência ou runbook de segurança na árvore da app. Se achar algum solto, mova pra cá.
2. **Testes de segurança ISOLADOS.** A suíte de segurança roda em alvo/config próprio (ex.: `security/tests/` com seu runner), **nunca** misturada à suíte da app — pra o histórico de segurança ser independente e auditável sozinho.
3. **Histórico APPEND-ONLY.** `CHANGELOG-SECURITY.md` só cresce: toda mudança de segurança gera entrada datada. Toda auditoria/pentest/review vai pra `reviews/` ou `relatorios/` **com data no nome**, nunca sobrescrevendo a anterior.
4. **Uma fonte da verdade.** `security/` é a casa canônica. Se algo de segurança existir em dois lugares, consolide aqui.
5. **Ao aplicar/atualizar o baseline:** copie `protocols/`, `playbooks/`, `checklists/` e o time pra cá, crie as pastas vazias com README, e **registre a adoção** no `CHANGELOG-SECURITY.md` (data + o que foi ligado + o tier).

Ao terminar: rode `git status` e liste o que criou. Confirme que **nenhum** artefato de segurança ficou fora de `security/`.

$ARGUMENTS
