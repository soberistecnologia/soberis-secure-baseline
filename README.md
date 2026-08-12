# 🛡️ Soberis Secure Baseline

Base de segurança **modular e ligável por pergunta** para começar qualquer projeto da Soberis já com a Camada 0 de pé — sem virar *canhão para matar formiga* 🐜.

Destilado do sistema **Lunar** (gestão financeira governamental, auditado e testado com carga real), com os **erros já corrigidos** de saída (observabilidade dia 1, segredo nunca em disco, backstop anti-DDoS global).

---

## Como usar (o Claude é o maestro — não é questionário de terminal)

Este template foi feito pra ser rodado **pelo Claude Code**, não digitando num prompt TTY.

```
Você:  "Claudião, clona o soberis-secure-baseline e configura pra esse projeto"
         (ou, dentro do repo já clonado:  /setup-baseline)
   ↓
Claude lê o CLAUDE.md deste repo (automático) e:
   1. INFERE o que der do projeto (tem banco? auth? mexe com dinheiro? Compose/Swarm?)
   2. PERGUNTA a você só o que faltar (porte + gatilhos), via a caixa de pergunta do Claude Code
   3. escreve baseline.answers.json e te mostra o que vai ligar/desligar (você confirma)
   4. roda `node setup.mjs` (NÃO-interativo — lê o answers.json, sem TTY)
   ↓
gerador filtra módulos + agentes + skills → projeto sob medida → se auto-remove
```

> **Por que assim?** O Claude roda comandos de forma não-interativa — um questionário que
> espera stdin travaria. Aqui as perguntas continuam, mas quem conduz é o Claude (e ele
> infere metade sozinho). Fallback humano: `node setup.mjs --interativo`.

O gerador produz: `docker-compose.yml` só com os serviços do tier · `.claude/agents`+`skills`
só do time filtrado · `SECURITY-BASELINE.md` sob medida · `CHECKLIST-PR.md` (gate) · CI
(gitleaks+trivy) — e some com o andaime (`setup.mjs`, `presets/`, `baseline.config.json`, `CLAUDE.md`).

## O que tem dentro

| Pasta | Conteúdo |
|---|---|
| `baseline.config.json` | O **cérebro do filtro** — módulos, tiers, presets, e qual agente/skill cada módulo puxa |
| `modules/` | Cada camada de segurança como **módulo independente** (compose + config + middleware + migration + doc) |
| `presets/` | 🐜 `formiga` · 🔧 `padrao` · 💥 `canhao` |
| `.claude/agents/` | O **time de agentes de segurança** (security-master + 11 especialistas) |
| `.claude/skills/` | Skills na ponta da língua (appsec-go, go-reference, infra-hardening, lgpd-gov) |
| `squads/` | Charters dos squads (missão, quando acionar) |
| `security/protocols/` | A **doutrina** — 13 protocolos (RBAC, auditoria, segredos, hardening…) |
| `docs/SECURITY-BASELINE.md` | As 11 camadas + o checklist por tier |

## Os 3 presets

- **🐜 formiga** — perímetro, borda (TLS+headers+rate-limit), secrets, container-hardening. Time: `infra-hardening`, `sec-segredos`, `security-master`.
- **🔧 padrão** — tudo do formiga **+** auth (Clerk), RBAC+ownership (anti-IDOR, 404 anti-enum), banco least-privilege, **observabilidade/alerta**. Time: `+ sec-auth-webhooks`, `sec-isolamento-acesso`, `appsec-go`, `runtime-detection`, `qa-adversarial`.
- **💥 canhão** — tudo do padrão **+** auditoria hash-chain, step-up 2FA, RBAC granular, integridade financeira, LGPD. Time: **completo**.

## Regras invioláveis (valem em todo tier)

1. **Segredo nunca** em git/log/chat/junto a IP.
2. **Fail-closed** — na dúvida, negar; o boot aborta sem hardening.
3. **Toda rota** valida auth **e** authz **e** escopo de dado.
4. **Dupla validação** — Executor + Validador independentes.

> ⚠️ **Este repo pratica o que ensina:** `.env*` e chaves estão no `.gitignore`, o CI roda `gitleaks`, e nenhum segredo mora aqui. Se você colocar um token no `.env.local`, ele **não** vai pro git — mas trate-o como o que é: um segredo.
