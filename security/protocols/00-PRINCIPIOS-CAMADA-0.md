---
protocolo: PRIN-00
titulo: Princípios da Camada 0 (Segurança)
status: canônico
fase: fundação
contexto: projeto governamental single-tenant
atualizado: 2026-07-21
---

# PRIN-00 — Princípios da Camada 0

> A Camada 0 é a **fundação de segurança**. Nenhuma feature sobe sem ela de pé. Contexto **governamental**: dinheiro e dados públicos → conformidade (TCU/controle interno, LGPD) e trilha inviolável são requisitos legais, não opcionais.

## 1. Os 10 mandamentos

1. **Fail-closed.** Na dúvida, negar. Ausência de permissão = negado. Config de produção incompleta = boot aborta.
2. **Defesa em profundidade.** Auth (Clerk) → Autorização (RBAC) → Escopo de dado → Validação de entrada → Auditoria. Uma camada falhar não pode abrir o sistema.
3. **Least privilege.** Todo perfil, serviço e token recebe o mínimo necessário. Aprovadores não editam; editores não aprovam (segregação de funções — SoD).
4. **Auditoria imutável e total.** Cada ação relevante é registrada (quem, o quê, quando, de onde, antes/depois). Ninguém apaga log. Ver [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md).
5. **Nada de exclusão física.** Dado de negócio só é `inativado`/`cancelado`. Histórico preservado. Ver [`07-SOFT-DELETE-VERSIONAMENTO.md`](07-SOFT-DELETE-VERSIONAMENTO.md).
6. **Registro aprovado é imutável.** Após aprovação de Diretor, trava. Alterar exige reabertura formal + nova versão. Ver [`06-FLUXO-APROVACAO.md`](06-FLUXO-APROVACAO.md).
7. **Segredo nunca exposto.** Nem em git, log, chat, doc versionada ou junto a IP. Ver [`05-GESTAO-SEGREDOS.md`](05-GESTAO-SEGREDOS.md).
8. **Segregação de funções (SoD).** Quem executa ≠ quem aprova ≠ quem audita. Reflete a hierarquia do órgão.
9. **Tudo rastreável e reversível.** Toda mudança tem autor, justificativa e forma de reverter. Migrations empilham, nunca destroem.
10. **Revisão dupla em segurança.** Código sensível (`internal/security`, auth, RBAC, auditoria) só entra com aval humano + agente de segurança.

## 2. Modelo de ameaça (resumo)

Contexto governamental amplia o interesse do atacante. Principais vetores tratados:

| Vetor | Mitigação | Protocolo |
|---|---|---|
| Acesso indevido / escalonamento de privilégio | RBAC com ownership + SoD + 2FA para aprovadores | [01](01-RBAC-PERFIS.md) |
| Adulteração/apagamento de trilha | Auditoria append-only + hash-chain + fora do alcance do usuário | [02](02-AUDITORIA-LOGS.md) |
| Vazamento de dado (BOLA/IDOR) | Validação de escopo em toda leitura, não só na string da permissão | [03](03-ISOLAMENTO-DADOS.md) |
| Comprometimento de credencial | Clerk + fail-closed + rotação + 2FA | [04](04-AUTH-CLERK.md) |
| Vazamento de segredo | sops/age + Swarm secrets + gitleaks no CI | [05](05-GESTAO-SEGREDOS.md) |
| Superfície de infra exposta | Hardening Swarm/Traefik, sem painel público, túnel WireGuard | [08](08-INFRA-HARDENING.md) |
| Não conformidade LGPD/TCU | Base legal, retenção, RIPD, resposta a incidente | [09](09-LGPD-COMPLIANCE.md) |

Threat model completo em [`../threat-model/THREAT-MODEL.md`](../threat-model/THREAT-MODEL.md).

## 3. Gate de segurança (Go / No-Go)

Nenhum deploy público sem o checklist [`../checklists/GO-NO-GO.md`](../checklists/GO-NO-GO.md) 100%. Os squads de segurança têm **poder de veto**: se um agente marca um achado P0/P1 aberto, o deploy não sai.

## 4. Tiers de autonomia dos agentes (AI-SOC)

Herdado do Locus, adaptado para governo:
- **auto-safe** — o agente aplica (formatação, doc, dependência de patch).
- **auto-veto** — o agente bloqueia e reporta (achado P0/P1, segredo detectado, RLS/escopo furado).
- **humano-obrigatório** — schema, deploy em produção, política de RBAC, retenção de dados, exposição de rota. Nunca automático.

## 5. Testes de segurança obrigatórios (por que fazemos todos)

Projeto governamental → bateria completa e recorrente:
- **SAST** (gosec, staticcheck, semgrep) — a cada commit.
- **Dependências** (govulncheck, trivy fs, `go mod` audit) — a cada commit.
- **Segredos** (gitleaks, trufflehog) — pre-commit + CI.
- **Container/infra** (trivy image, docker-bench, CIS) — a cada build.
- **DAST / pentest** (caixa-preta) — por release + externo periódico.
- **Provas de isolamento** (testes adversariais de escopo/BOLA) — suíte dedicada.
- **Runtime** (detecção de anomalia) — contínuo.
- **Revisão noturna** por agente sênior read-only — diária.

> Cada bateria gera relatório datado em `security/reviews/` _(pasta criada no primeiro relatório)_ e, se houver ajuste, entrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md).
