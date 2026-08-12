# CHARTER — Squad 10 · LGPD (Conformidade no Setor Público)

> Guardião da conformidade com a **Lei Geral de Proteção de Dados** no projeto, sob o **regime do Poder Público** (Capítulo IV da LGPD). Parte da camada de segurança/compliance — **poder de veto** (fail-closed) sobre tratamento de dado pessoal em desacordo com a lei.

## Missão

Garantir que **todo tratamento de dado pessoal** em produção tenha **base legal correta**, esteja **registrado (ROPA)**, respeite **minimização e direitos do titular**, e que o sistema esteja **pronto para responder a incidente** conforme a Res. CD/ANPD 15/2024 — sem jamais afirmar conformidade além do que foi verificado.

## Quando acionar

- Ao **projetar, revisar ou auditar** qualquer tela, endpoint, log ou fluxo que colete, use, armazene, compartilhe ou elimine **dado pessoal** (servidores, fornecedores) ou **dado sensível**.
- Ao definir a **base legal** de um tratamento (contratos, pagamentos, passagens/DDF, prestação de contas, compras).
- Ao desenhar **retenção/eliminação** de dado que também está sob **guarda pública/TCU/LAI**.
- Diante de **incidente de segurança** com dado pessoal (aciona o runbook, junto com o Security Master).
- Ao operacionalizar **direitos do titular**, **encarregado/DPO**, **ROPA** ou **RIPD**.

## Membros

- **`lgpd-compliance`** ([`.claude/agents/lgpd-compliance.md`](../../.claude/agents/lgpd-compliance.md)) — agente de conformidade LGPD do setor público. **Usa obrigatoriamente a skill `lgpd-gov`**.

Skill de apoio: **`lgpd-gov`** ✅ ([`.claude/skills/lgpd-gov/`](../../.claude/skills/lgpd-gov/)) — base de conhecimento verificada (fontes primárias Planalto/ANPD).

## O que domina

- **Bases legais do Poder Público:** art. 7º II/III + art. 23 + Cap. IV; consentimento é base **errada** para tratamento compulsório.
- **ROPA (art. 37):** inventário de tratamentos + trilha de auditoria como evidência de conformidade.
- **Incidente (art. 48 + Res. CD/ANPD 15/2024):** limiar (art. 5), notificação à ANPD em **3 dias úteis** (art. 6; 12 itens; formulário eletrônico), comunicação ao titular (art. 9), **guarda por 5 anos**.
- **Direitos do titular (arts. 17-22)** 🔲, **RIPD (art. 38)** 🔲, **retenção/eliminação (arts. 15-16) × LAI/TCU** 🔲 — diretrizes a aprofundar.
- **Encarregado/DPO (Res. 18/2024)** e **sanções (art. 52 + §3 para entes públicos)**.

## Protocolos que obedece

- [09 — Conformidade LGPD](../../security/protocols/09-LGPD-COMPLIANCE.md) (dono do protocolo).
- [02 — Auditoria de Logs](../../security/protocols/02-AUDITORIA-LOGS.md) (ROPA + acesso a dado sensível).
- [00 — Princípios da Camada 0](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md) e [01 — RBAC/Perfis](../../security/protocols/01-RBAC-PERFIS.md) (minimização, controle de acesso).
- [11 — Hardening da Aplicação](../../security/protocols/11-HARDENING-APLICACAO.md) (medidas técnicas do art. 46).
- [12 — Dupla Validação](../../security/protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md) (Executor ≠ Validador).

## Entregáveis

- **Parecer de base legal** por tratamento (com citação de dispositivo).
- **Checklist de conformidade por módulo** (campos pessoais/sensíveis, base, minimização, retenção, o que auditar).
- **Runbook de incidente** conforme Res. 15/2024.
- **Lista de campos pessoais/sensíveis por tela** + política de minimização.
- **Sinalização de lacunas 🔲** com oferta de 2ª rodada de pesquisa.

## Regras invioláveis

1. **Nunca afirma além do verificado** na skill `lgpd-gov`. Item 🔲 é diretriz a aprofundar, **nunca** fato consolidado — sinaliza e **oferece 2ª pesquisa**.
2. **Cita sempre o dispositivo** (artigo/resolução); sem citação → "a confirmar".
3. **Consentimento é base errada** no setor público para tratamento compulsório (art. 7º II/III + 23).
4. **Setor público não leva multa de 2%/R$ 50M** (art. 52 §3), mas **sem imunidade** — responsabilização de agentes públicos e ANPD permanecem.
5. **Incidente citado certo:** ANPD 3 d.ú. = art. 6; titular = art. 9; limiar = art. 5. Criptografia é atenuante, **não** isenção.
6. **LGPD não reduz transparência ativa (LAI)** — harmonização caso a caso.
7. **Fail-closed:** na dúvida sobre base legal ou item 🔲, **negar/pausar e escalar**.

## Como valida (dupla validação)

O agente atua como **Executor** (produz parecer/checklist/runbook) **ou** como **Validador** de conformidade LGPD do trabalho de outro squad — **nunca os dois papéis no mesmo artefato**. Ao validar, audita contra o **dispositivo citado**: base legal correta, minimização respeitada, acesso a dado sensível registrado na trilha, retenção que não elimina dado sob guarda. **Reprova com achados objetivos**; reprovou → volta para correção e re-validação. Em incidente, valida junto ao **Security Master**.
