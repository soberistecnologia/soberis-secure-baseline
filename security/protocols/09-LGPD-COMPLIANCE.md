---
protocolo: LGPD-09
titulo: Conformidade LGPD (setor público)
status: canônico (base verificada; itens 🔲 pendentes de 2ª rodada)
fase: fundação
base_conhecimento: .claude/skills/lgpd-gov/references/base-lgpd-setor-publico.md
skill: lgpd-gov
atualizado: 2026-07-21
---

# LGPD-09 — Conformidade LGPD (Setor Público)

> O sistema é sistema de um **órgão governamental** → LGPD com **regime do Poder Público** (Cap. IV). Este protocolo fixa a postura de conformidade; o detalhe verificado e citado está em [`../../.claude/skills/lgpd-gov/references/base-lgpd-setor-publico.md`](../../.claude/skills/lgpd-gov/references/base-lgpd-setor-publico.md). Ao trabalhar com dado pessoal, **use a skill `lgpd-gov`**.

## 1. Postura de conformidade do projeto

1. **Base legal:** todo tratamento se ancora em **obrigação legal (art. 7º,II)** ou **política pública/uso compartilhado (art. 7º,III)** + **finalidade/interesse público (art. 23)**. **Não** implementar fluxo de "consentimento" como base para tratamento compulsório.
2. **Minimização (art. 6):** coletar só o necessário; nunca PII sensível excessiva em logs; redação de dado sensível na trilha conforme [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md).
3. **Registro das operações (art. 37 / ROPA):** inventário de tratamentos + trilha de auditoria = evidência de conformidade.
4. **Segurança (art. 46):** satisfeita pelos protocolos [01](01-RBAC-PERFIS.md) (RBAC+SoD), [11](11-HARDENING-APLICACAO.md) (cripto/hardening), [02](02-AUDITORIA-LOGS.md) (auditoria).
5. **Encarregado (DPO):** papel previsto no RBAC + canal do titular (Res. CD/ANPD 18/2024).

## 2. Runbook de incidente de segurança (art. 48 + Res. CD/ANPD 15/2024)

Fluxo obrigatório quando houver incidente com dados pessoais:

1. **Detectar & conter** (Squad InfraSec/Runtime + Security Master).
2. **Avaliar o limiar (art. 5 da Res. 15/2024):** afeta significativamente direitos **E** envolve ≥1 de: dados sensíveis, de crianças/adolescentes/idosos, financeiros, de autenticação, sob sigilo, ou em larga escala? *(Dados financeiros e de autenticação são o caso típico do projeto.)* Criptografia é atenuante, **não** isenção.
3. **Se aciona:** comunicar à **ANPD em 3 dias úteis** do conhecimento (art. 6; 6 d.ú. se pequeno porte), via **formulário eletrônico** com os **12 itens**; **comunicação preliminar** admite complementação em **20 dias úteis**.
4. **Comunicar o titular** (art. 9): linguagem simples, individualizada quando possível; se inviável, divulgação pública ≥ 3 meses.
5. **Registrar o incidente e guardar por 5 anos.**
6. **Pós-incidente:** entrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md) + relatório na pasta `security/incidents/` _(criada no primeiro incidente)_.

## 3. Retenção × guarda pública × LAI ✅ (verificado — Rodada 2)

- **Regra:** encerrado o tratamento, dado pessoal **em regra é eliminado** — condicionado às normas de gestão documental/**tabelas de temporalidade (CONARQ, Lei 8.159/1991)**. *(Guia ANPD Poder Público.)*
- **Retenção autorizada (art. 16):** (I) **cumprimento de obrigação legal** — ancora a guarda **contábil/TCU/Lei 4.320/1964**; (IV) **uso exclusivo do controlador se anonimizado**. Implementação: **soft-delete/temporalidade** ([07](07-SOFT-DELETE-VERSIONAMENTO.md)), nunca exclusão física; fim do prazo sem base → eliminar ou anonimizar.
- **LGPD × LAI:** **publicidade é a regra, sigilo é exceção**; divulgação de PII observa princípios LGPD. Recursos de negativa: **CGU** (5 dias) e **CMRI** (Dec. 7.724/2012, arts. 23-24). Direitos no setor público observam Habeas Data, Lei 9.784/1999 e LAI (art. 23 §3 LGPD). Não usar LGPD como pretexto para reduzir transparência.
- 🔲 **Ainda aberto:** prazos numéricos por tipo documental (TCU/temporalidade) para parametrização.

## 3.1 Framework de segurança gov (verificado — Rodada 2)

- **E-Ciber vigente: Decreto 12.573/2025** (executa a **PNCiber**, Decreto 11.856/2023). Governança GSI/PR + CNCiber.
- **IN GSI/PR 1/2020:** POSIC obrigatória, revisão ≤4 anos; **art. 19, X** liga SI à LGPD (Gestor de SI ↔ Encarregado/DPO). Nosso [`08-INFRA-HARDENING.md`](08-INFRA-HARDENING.md) + [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md) já cobrem Controles de Acesso, Gestão de Incidentes e Auditoria (diretrizes mínimas da POSIC).
- **Dados sensíveis de servidor (art. 11):** base do art. 11 (não art. 7), mais restrita (art. 11, II, "b" = só leis/regulamentos). **Anonimização (art. 12)** tira do escopo; **pseudonimização (art. 13 §4º)** continua dado pessoal (medida de segurança).

## 4. Sanções (art. 52) — expectativa realista

Entes públicos: **multa em geral não incide** (art. 52 §3); sanções aplicáveis = advertência, publicização, bloqueio/eliminação. **Sem imunidade:** responsabilização de agentes públicos e atuação da ANPD permanecem. Conformidade não é opcional.

## 5. Pendências (após Rodada 2)

✅ **Resolvido na Rodada 2:** retenção arts. 15-16 × TCU/LAI · direitos do titular arts. 17-22 · RIPD art. 38 · framework vigente (E-Ciber 2025/PNCiber/IN GSI 1/2020) · art. 11 (sensíveis) · anonimização.

🔲 **Ainda aberto (3ª rodada, se necessário):** checklist item-a-item do **art. 46** (Guia de Segurança ANPD); **prazos numéricos** de guarda TCU/temporalidade CONARQ; **P-Ciber** detalhado; **crosswalk ISO 27001:2022/27701**. Ver base de conhecimento da skill (seção "Lacunas ainda abertas").

## 6. Responsáveis

**Squad LGPD** (dono) + **Squad de Auditoria & Compliance** + **Security Master** (incidente). Skill: `lgpd-gov`.
