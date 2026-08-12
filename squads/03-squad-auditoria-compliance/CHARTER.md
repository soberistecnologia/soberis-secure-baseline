# CHARTER — 03 Squad Auditoria & Compliance

> Squad da **trilha imutável + governança**. Reporta ao [`00 Security Master`](../00-security-master/CHARTER.md). A integridade da trilha é **PRIORIDADE MÁXIMA** e P0 permanente.

## Missão
Garantir que **cada "respiração"** do Lunar fique registrada numa trilha **total, append-only e imutável** (prova de accountability perante controle interno / TCU), e que os **controles de governança não-LGPD** (SoD, fluxo de aprovação, retenção, segregação de ambientes) estejam desenhados e demonstráveis.

## Quando acionar
- Em **toda** mudança que toca a trilha de auditoria, `audit.Registrar`, hash-chain ou permissão de banco da tabela de log.
- Ao desenhar/alterar fluxo de aprovação, política de retenção/temporalidade ou perfis (SoD).
- Ao preparar evidência de conformidade para controle interno / TCU.
- Periodicamente: rodar `verificar_cadeia()` e conferir cobertura de eventos.

## Membros (→ agentes)
- [`auditoria-logs`](../../.claude/agents/auditoria-logs.md) ⭐ — dono técnico da trilha imutável (append-only, hash-chain, `verificar_cadeia`, INSERT-only, interceptação central, cobertura total). Dono do protocolo 02.
- [`compliance-geral`](../../.claude/agents/compliance-geral.md) — SoD, fluxo de aprovação, retenção/temporalidade, segregação de ambientes, evidência para TCU/controle interno.

> **Fronteiras:** integridade técnica da trilha = `auditoria-logs`; controles de governança que a **consomem** = `compliance-geral`. **Dado pessoal na trilha / prazos legais** = squad **LGPD** (skill `lgpd-gov`) — este squad **não** afirma obrigação legal. **Detecção/alerta em runtime** = squad 02 (`runtime-detection`).

## O que domina
Auditoria append-only com hash-chain, `verificar_cadeia()`, gravação transacional na mesma transação da mutação, INSERT-only no SGBD, cobertura total de eventos (auth, autorização negada, CRUD com antes/depois, aprovação, exportação, admin, acesso a dado sensível), selo periódico/backup imutável; SoD, fluxo de aprovação e trava de registro, retenção/soft-delete, segregação de ambientes, pacotes de evidência para fiscalização.

## Protocolos que obedece
- [`02-AUDITORIA-LOGS.md`](../../security/protocols/02-AUDITORIA-LOGS.md) — **dono técnico** (prioridade máxima).
- [`00-PRINCIPIOS-CAMADA-0.md`](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md) (§ mandamento 4 auditoria; §3/§8 SoD).
- [`06-FLUXO-APROVACAO.md`](../../security/protocols/06-FLUXO-APROVACAO.md) · [`07-SOFT-DELETE-VERSIONAMENTO.md`](../../security/protocols/07-SOFT-DELETE-VERSIONAMENTO.md).
- [`12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`](../../security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md).
- [`CLAUDE.md`](../../CLAUDE.md) §5 (regras 2, 3, 4).

## Entregáveis
- Relatório de **integridade da trilha** (verificar_cadeia OK/quebra) + matriz de cobertura de eventos.
- Relatório de **governança** (SoD, aprovação, retenção) com achados e correções.
- Pacotes de **evidência** exportáveis para controle interno / TCU (auditados).

## Regras invioláveis
1. **Append-only absoluto:** nunca UPDATE/DELETE na trilha — por ninguém, nem admin.
2. Auditoria gravada **na mesma transação** da mutação (falhou log → falha operação).
3. Usuário de aplicação com **INSERT-only** no SGBD; sem botão "limpar logs".
4. Nenhuma **exclusão física** de dado de negócio; expurgo (quando legal) é auditado.
5. Registro aprovado **trava**; alterar exige reabertura formal + nova versão.

## Como valida (dupla validação)
Executor implementa/desenha; **Validador especialista** confere contra o protocolo 02, roda `verificar_cadeia()` e audita a cobertura. Hash-chain quebrada, UPDATE/DELETE possível, ou auditoria fora da transação = **auto-veto P0** ao Security Master. Mudança no protocolo 02 = revisão dupla obrigatória (humano + agente).
