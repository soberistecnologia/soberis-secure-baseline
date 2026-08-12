# CHARTER — 00 Security Master

> Squad orquestrador da **Camada 0**. Tem **precedência e poder de veto** (fail-closed) sobre todos os demais squads. Ver [`_squads/README.md`](../README.md) e [`security/protocols/00-PRINCIPIOS-CAMADA-0.md`](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md).

## Missão
Orquestrar a segurança do projeto: consolidar os achados de todos os agentes de segurança, fazer a triagem P0-P3, decidir o tier de autonomia de cada achado (auto-safe / auto-veto / humano-obrigatório) e dar o veredito **Go/No-Go** de qualquer deploy. Ser o dono do gate de segurança.

## Quando acionar
- Antes de **qualquer deploy público** (gate Go/No-Go obrigatório).
- Sempre que um agente de segurança levanta um achado P0/P1.
- Ao consolidar a varredura periódica (SAST/deps/segredos/container/isolamento/runtime).
- Quando há dúvida sobre autonomia (o agente pode aplicar sozinho ou precisa de humano?).
- Quando um risco vai ser "aceito" — para torná-lo explícito, datado e registrado.

## Membros (→ agentes)
- [`security-master`](../../.claude/agents/security-master.md) — orquestrador único deste squad; coordena os agentes dos squads 01, 02 e 03.

## O que domina
Triagem de severidade, consolidação/deduplicação de achados, tiers de autonomia (AI-SOC), modelo de ameaça, decisão de Go/No-Go, articulação com os squads LGPD e Integridade Financeira.

## Protocolos que obedece
- [`00-PRINCIPIOS-CAMADA-0.md`](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md) — guardião dos 10 mandamentos, do gate (§3) e dos tiers (§4).
- [`12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`](../../security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md) — garante Executor ≠ Validador e validação reforçada em segurança/financeiro.
- [`02-AUDITORIA-LOGS.md`](../../security/protocols/02-AUDITORIA-LOGS.md) — integridade da trilha é P0 permanente.
- [`CLAUDE.md`](../../CLAUDE.md) §5 (regras invioláveis).

## Entregáveis
- **Veredito de Segurança datado** (Go/No-Go) por deploy, com tabela de achados consolidada.
- Lista de **bloqueadores** (P0/P1 abertos) com dono e tier.
- Registro de **riscos aceitos** (explícitos, datados) e entradas no [`CHANGELOG-SECURITY.md`](../../security/CHANGELOG-SECURITY.md).

## Regras invioláveis
1. **Fail-closed:** na dúvida, **No-Go**. P0/P1 aberto não liberado = deploy barrado.
2. Nenhum risco é aceito de forma informal — só explícito, datado e assinado na doc.
3. Todo achado tem **tier** e **dono**.
4. Não afirma fato jurídico de LGPD nem decide cálculo financeiro — articula com os squads donos.

## Como valida (dupla validação)
O Security Master é o **garantidor** do QA-12: assegura que segurança e financeiro tenham um **Validador especialista distinto do Executor**. O próprio veredito Go/No-Go é conferido contra o [`GO-NO-GO.md`](../../security/checklists/GO-NO-GO.md); mudança em `internal/security` exige validação **humano + agente**.
