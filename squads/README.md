# Squads do Projeto Lunar — Organograma

> Os agentes que constroem e defendem o Lunar. Cada squad tem **CHARTER** (aqui), **agente(s) funcional(is)** em [`.claude/agents/`](../.claude/agents/) e, quando há conhecimento especializado, **skill(s)** em [`.claude/skills/`](../.claude/skills/).
> **Precedência:** squads de **segurança têm poder de veto** (fail-closed). Nada conclui sem **dupla validação** (Executor + Validador) — ver [`security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`](../security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md).

## Organograma

```
                        ┌─────────────────────┐
                        │  00 SECURITY MASTER  │  orquestra segurança, triagem P0-P3, veto
                        └──────────┬───────────┘
        ┌──────────────┬──────────┼───────────┬──────────────┐
   ┌────┴────┐   ┌─────┴─────┐ ┌──┴──────────┐ ┌────────────┐ ┌──────────────┐
   │01 APPSEC│   │02 INFRASEC│ │03 AUDITORIA │ │  10 LGPD   │ │11 INTEGRIDADE│
   │         │   │           │ │ &COMPLIANCE │ │            │ │  FINANCEIRA  │
   └─────────┘   └───────────┘ └─────────────┘ └────────────┘ └──────────────┘

   ─── Squads de produto/engenharia ───
   ┌─────────────┐ ┌────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐ ┌──────────┐
   │04 ARQUITETURA│ │ 05 GO  │ │06 UI/UX  │ │ 07 QA  │ │08 DEVOPS │ │09 MOBILE │
   └─────────────┘ └────────┘ └──────────┘ └────────┘ └──────────┘ └──────────┘
```

## Roster

| # | Squad | Missão (1 linha) | Agentes | Skills |
|---|---|---|---|---|
| 00 | **Security Master** | Orquestra a segurança, triagem P0-P3, poder de veto | `security-master` | — |
| 01 | **AppSec** | Segurança de aplicação Go: injeção, IDOR/BOLA, auth, segredos | `appsec-go`, `sec-isolamento-acesso`, `sec-auth-webhooks`, `sec-segredos` | `appsec-go` |
| 02 | **InfraSec** | Hardening de infra (Swarm/Traefik/CIS) + detecção em runtime | `infra-hardening`, `runtime-detection` | `infra-hardening` |
| 03 | **Auditoria & Compliance** | Integridade da trilha imutável + controles de governança | `auditoria-logs`, `compliance-geral` | — |
| 04 | **Arquitetura** | Arquitetura de software, camadas, ADRs, trade-offs | `arquiteto-software` | — |
| 05 | **Go** | Especialista Go — docs na ponta da língua, idiomático, seguro | `go-expert` | `go-reference` |
| 06 | **UI/UX** | Design de interface/experiência, acessibilidade, anti-slop | `uiux-designer` | (usa `refero-design`) |
| 07 | **QA** | Dupla validação, testes adversarial/reliability/structural + **revisor de português** | `qa-adversarial`, `qa-reliability`, `qa-structural`, `revisor-portugues` | — |
| 08 | **DevOps** | Docker Swarm + Portainer + Traefik, CI/CD, deploy seguro | `devops-swarm` | — |
| 09 | **Mobile** | Estrutura e boas práticas mobile (indispensável desde o início) | `mobile-expert` | — |
| 10 | **LGPD** | Conformidade LGPD setor público, incidentes, RIPD | `lgpd-compliance` | `lgpd-gov` ✅ |
| 11 | **Integridade Financeira** | Zero erro de cálculo; validação cruzada de tudo que é dinheiro | `integridade-financeira` | (futuro) |

## Padrão de um AGENTE (`.claude/agents/<nome>.md`)

```markdown
---
name: <nome-kebab>
description: <quando acionar este agente — 1-2 frases, com gatilhos>
tools: <opcional: lista; omitir = todas>
model: <opcional: sonnet|opus|haiku>
---

<System prompt do agente: quem é, o que domina, como trabalha, o que NUNCA faz,
a quais protocolos obedece, e o formato de entrega.>
```

## Padrão de um CHARTER (`_squads/<NN-squad>/CHARTER.md`)

Missão · Quando acionar · Membros (→ agentes) · O que domina · Protocolos que obedece · Entregáveis · Regras invioláveis · Como valida (dupla validação).

## Princípios comuns a todos os squads

1. **Doc-first** — decisão fora da doc não existe.
2. **Dupla validação** — Executor ≠ Validador.
3. **Fail-closed** — na dúvida, negar/parar e escalar.
4. **Sabe exatamente o que é** — cada agente conhece seu escopo e seus limites.
5. **Obedece à Camada 0** — [`security/protocols/`](../security/protocols/) é lei.
