# CHARTER — Squad 07 · QA

> **Precedência:** o QA conduz a **dupla validação** de todo o projeto (Executor ≠ Validador). Nenhuma execução ou fase conclui sem o "validado" de um agente independente. O **revisor de português** é gate de merge para qualquer texto exibido. Ver [`security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`](../../security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md).

## Missão

Garantir que nada suba sem **prova de qualidade** sob três ângulos independentes — quebra (adversarial), confiabilidade (reliability) e conformidade (structural) — e que **todo texto** do sistema esteja em português impecável. O QA é o dono do processo de **dupla validação** do Projeto Lunar: em contexto governamental de gestão financeira, "parece funcionar" não basta — tem de estar provado, estável, conforme e bem escrito.

## Quando acionar

- Antes de concluir qualquer fase ou execução (a validação é obrigatória, não opcional).
- Em toda mudança de regra de negócio, cálculo financeiro, fluxo de aprovação, RBAC/escopo ou auditoria.
- Sempre que houver texto novo ou alterado destinado ao usuário (interface, mensagem, relatório, PDF, e-mail).
- Quando o `security-master` exigir validação reforçada de um domínio sensível (segurança/financeiro).
- Antes de todo deploy — como parte do Go/No-Go (junto com os squads de segurança e o DevOps).

## Membros (→ agentes)

| Papel | Agente | Foco |
|---|---|---|
| QA Adversarial | [`qa-adversarial`](../../.claude/agents/qa-adversarial.md) | Red-team funcional: casos-limite, entradas maliciosas, bypass de autorização (BOLA/IDOR), race conditions, quebra de invariante financeira; prova o bug com teste. |
| QA de Confiabilidade | [`qa-reliability`](../../.claude/agents/qa-reliability.md) | Cobertura real, determinismo, idempotência, resiliência a falha, regressão, suíte financeira determinística. |
| QA Estrutural | [`qa-structural`](../../.claude/agents/qa-structural.md) | Conformidade com arquitetura/protocolos, camadas, adapter pattern, padrão de código, checklists de gate, doc-first. |
| Revisor de Português ⭐ | [`revisor-portugues`](../../.claude/agents/revisor-portugues.md) | Gate linguístico: zero erro de português e terminologia consistente em todo texto do sistema. Validador textual. |

## O que domina

- **Dupla validação como processo** (QA-12 §1): separar Executor de Validador em toda entrega; registrar o par (executor, validador) e o veredito.
- **Três lentes de teste** (padrão do squad de QA do NexCollabs, com relatórios datados): adversarial, reliability e structural — independentes e complementares.
- **Integridade financeira do lado do teste** (QA-12 §2): garantir suíte determinística e de propriedade, recálculo cruzado e invariantes de conciliação (o dono técnico do cálculo é o squad de Integridade Financeira; o QA garante a disciplina de teste e a validação cruzada).
- **Qualidade textual** (QA-12 §3): revisão linguística obrigatória, strings centralizadas, terminologia do domínio (Empenho, Prestação de Contas…).
- **Rastreabilidade:** achados com ID e severidade (P0–P3), relatórios datados, correção que vira teste de regressão.

## Protocolos que obedece

- [`12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`](../../security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md) — a razão de existir do squad.
- [`00-PRINCIPIOS-CAMADA-0.md`](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md) — fail-closed, SoD, testes obrigatórios recorrentes, gate Go/No-Go.
- [`11-HARDENING-APLICACAO.md`](../../security/protocols/11-HARDENING-APLICACAO.md) — superfícies a testar e o checklist de porte (§9).
- [`02-AUDITORIA-LOGS.md`](../../security/protocols/02-AUDITORIA-LOGS.md) — a integridade da trilha entra na cobertura.
- [`CLAUDE.md`](../../CLAUDE.md) §5 (regras invioláveis) e [`_squads/README.md`](../README.md) (padrões de squad).

## Entregáveis

- **Relatórios datados por lente** (adversarial / reliability / structural / consolidado), no padrão do NexCollabs.
- **Revisão linguística datada** por artefato de texto (APROVADO/REPROVADO com correções).
- **Testes** que provam bug (adversarial), travam regressão (reliability) e checam conformidade (structural).
- **Veredito de dupla validação** por fase: par (executor, validador) + resultado, rastreável no changelog/relatório.

## Regras invioláveis

1. **Executor ≠ Validador** — ninguém valida o próprio trabalho.
2. **Zero erro de cálculo** — divergência de centavo é P0; recálculo cruzado obrigatório.
3. **Zero erro de português** — texto reprovado bloqueia o merge; o gate é o `revisor-portugues`.
4. **Reprovou → volta** — achado do Validador vira tarefa de correção; re-executa e re-valida.
5. **Fail-closed** — na dúvida entre "aceitável" e "defeito", trata como defeito e escala.
6. **Prova ou não vale** — alegação sem teste reproduzível/relatório não é achado nem aprovação.

## Como valida (dupla validação)

1. O **Executor** (o squad que produziu a mudança — Go, Arquitetura, UI/UX etc.) entrega a execução.
2. O QA designa **Validadores independentes**: `qa-adversarial` tenta quebrar, `qa-reliability` prova estabilidade, `qa-structural` confere conformidade, `revisor-portugues` aprova o texto.
3. Domínio sensível recebe **validação reforçada**: cálculo é conferido pelo squad de Integridade Financeira; segurança pelo `security-master`/AppSec; mudança em `internal/security` exige revisão dupla (humano + agente).
4. Qualquer reprova devolve a tarefa ao Executor; a fase só fecha com **todos os vereditos "validado"** e o registro do par (executor, validador) na doc.
