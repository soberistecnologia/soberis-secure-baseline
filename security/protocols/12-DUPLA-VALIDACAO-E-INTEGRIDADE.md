---
protocolo: QA-12
titulo: Dupla Validação, Integridade Financeira e Qualidade Textual
status: canônico
prioridade: MÁXIMA
atualizado: 2026-07-21
---

# QA-12 — Dupla Validação, Integridade Financeira e Qualidade Textual

> Diretriz do dono, textual: *"todas as fases precisam ter double check, cada execução deve ser revisada e validada por dois agentes"*, *"NÃO PODE EM HIPÓTESE ALGUMA CONTER ERROS DE CÁLCULO"*, *"O SISTEMA NÃO PODE CONTER ERROS DE PORTUGUÊS"*.

## 1. Dupla validação (double-check por 2 agentes)

**Regra:** nenhuma execução ou fase é considerada concluída sem passar por **dois agentes independentes**: o **Executor** (quem faz) e o **Validador** (quem revisa e valida). O Validador **não pode** ser o mesmo agente que executou.

| Etapa | Papel A (Executor) | Papel B (Validador) |
|---|---|---|
| Produz | implementa/escreve | — |
| Revisa | — | audita contra requisito + protocolo |
| Aprova | — | dá o "validado" (ou reprova com achados) |

- **Reprovou → volta.** Achado do Validador vira tarefa de correção; re-executa e re-valida.
- **Segurança e financeiro exigem validação reforçada** (ver §2 e §3): o Validador é um agente **especialista naquele domínio** (ex.: cálculo financeiro validado por agente de integridade financeira; crypto validada por agente AppSec).
- **Rastreável:** o par (executor, validador) e o veredito ficam registrados (changelog/relatório). Mudança em `internal/security` = validador **humano + agente** (revisão dupla do PRIN-00).

## 2. Integridade financeira absoluta (zero erro de cálculo)

Contexto: recursos públicos (empenhos, pagamentos, DDF, passagens). **Um erro de cálculo é falha crítica** — pode virar dano ao erário.

**Regras de engenharia financeira:**
1. **Nada de `float`/`float64` para dinheiro.** Usar tipo de precisão fixa (decimal — ex.: `int64` em centavos ou `shopspring/decimal`), com regra de arredondamento explícita e documentada.
2. **Uma única fonte de verdade de cálculo** (biblioteca/serviço central), nunca cálculo duplicado espalhado.
3. **Testes determinísticos e de propriedade** cobrindo casos-limite (zero, negativo, arredondamento, soma de muitos itens, moeda, percentuais).
4. **Validação cruzada:** todo resultado financeiro relevante é conferido por um **segundo método/segundo agente** (recálculo independente) antes de persistir/exibir.
5. **Conciliação:** somatórios batem com a soma das partes (invariantes checadas em runtime; divergência → bloqueio + auditoria).
6. **Rastro:** todo valor calculado é auditável (entrada → fórmula → saída), coerente com [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md).
7. **Dono técnico:** Squad de Integridade Financeira (futuro) + Validador financeiro obrigatório em qualquer PR que toque cálculo.

## 3. Qualidade textual (zero erro de português)

Todo texto exibido — UI, mensagens de erro, relatórios, e-mails, notificações, PDFs — **não pode conter erro de português**.

1. **Revisão linguística obrigatória** por um agente/rotina revisora antes de qualquer texto ir para produção.
2. **Centralizar strings** (i18n/catálogo de mensagens) para revisão e consistência — nada de texto solto e não revisado no código.
3. **Padrão:** português formal, claro, correto (concordância, crase, acentuação, pontuação). Terminologia consistente (ex.: "Empenho", "Prestação de Contas").
4. **Gate:** texto novo/alterado passa pelo revisor; reprova bloqueia o merge.

## 4. Onde isso se conecta

- **Squad QA** conduz a dupla validação como processo padrão.
- **Squad de Integridade Financeira** (a criar) é o Validador de tudo que envolve cálculo/dinheiro.
- **Revisor de Português** (papel dentro de QA/UX) é o gate textual.
- **Security Master** garante que segurança e financeiro tenham validação reforçada e poder de veto.

> Mudança neste protocolo → entrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md).
