# CHARTER — 01 Squad AppSec

> Squad de **segurança de aplicação** (backend Go). Reporta ao [`00 Security Master`](../00-security-master/CHARTER.md). Precedência de segurança / poder de veto (fail-closed).

## Missão
Garantir que o código de aplicação do Lunar seja seguro em **quatro frentes**: injeção e higiene de código Go, isolamento/controle de acesso (anti-IDOR), autenticação/webhooks (Clerk) e gestão de segredos. É a linha de frente contra os erros que derrubaram os sistemas de referência (IDOR do One Nexus, segredos expostos do Locus, ausência de MFA do NexCollabs).

## Quando acionar
- Em **todo PR/diff** que toca o backend Go, rota, auth, RBAC, escopo de dado ou segredo.
- A cada commit (SAST/deps/segredos) e antes de cada deploy.
- Ao adicionar/alterar dependência, integração externa ou handler de webhook.
- Ao criar/alterar perfil, permissão ou fluxo de acesso a dado.

## Membros (→ agentes)
- [`appsec-go`](../../.claude/agents/appsec-go.md) — injeção, gosec/govulncheck/staticcheck, validação de entrada, erro seguro, deps.
- [`sec-isolamento-acesso`](../../.claude/agents/sec-isolamento-acesso.md) — BOLA/IDOR, ownership/escopo em toda leitura, RBAC enforcement, testes adversariais.
- [`sec-auth-webhooks`](../../.claude/agents/sec-auth-webhooks.md) — Clerk (JWT/JWKS, azp, iss/aud), webhooks HMAC, SSRF, 2FA de aprovadores.
- [`sec-segredos`](../../.claude/agents/sec-segredos.md) — gitleaks/trufflehog, sops+age, Swarm secrets, rotação, política anti-vazamento.
- **Skill:** [`appsec-go`](../../.claude/skills/appsec-go/SKILL.md) — checklist acionável de AppSec Go.

> **Fronteiras internas:** injeção/deps = `appsec-go`; o que a identidade pode acessar = `sec-isolamento-acesso`; provar quem é a identidade = `sec-auth-webhooks`; valor/custódia de segredo = `sec-segredos`. Cada agente encaminha o que é do outro.

## O que domina
OWASP (injeção, BOLA, autenticação quebrada, exposição de segredo), SAST/deps em Go, validação de entrada estrita, hash-chain de auth, anti-IDOR com ownership, HMAC/SSRF, detecção e rotação de segredos.

## Protocolos que obedece
- [`11-HARDENING-APLICACAO.md`](../../security/protocols/11-HARDENING-APLICACAO.md) — camadas extras (crypto, sessão, borda HTTP, segredos).
- [`01-RBAC-PERFIS.md`](../../security/protocols/01-RBAC-PERFIS.md) · [`03-ISOLAMENTO-DADOS.md`](../../security/protocols/03-ISOLAMENTO-DADOS.md) · [`04-AUTH-CLERK.md`](../../security/protocols/04-AUTH-CLERK.md) · [`05-GESTAO-SEGREDOS.md`](../../security/protocols/05-GESTAO-SEGREDOS.md).
- [`00-PRINCIPIOS-CAMADA-0.md`](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md) (§5 bateria obrigatória).
- [`CLAUDE.md`](../../CLAUDE.md) §5 (regras 1, 5, 7, 12).

## Entregáveis
- Relatórios datados por agente (AppSec, Isolamento, Auth/Webhooks, Segredos) com achados severidade × correção.
- Suíte de **provas de isolamento** adversariais (bypass cross-perfil/cross-escopo).
- Evidência de SAST/deps/segredos limpa por commit.

## Regras invioláveis
1. **Duas camadas em toda leitura:** permissão (string) **e** escopo (ownership). Ter a permissão não autoriza ler dado alheio.
2. Identidade/tenant **só do claim** validado, nunca do body.
3. Webhook sem HMAC constant-time = bloqueio. Produção nunca em instância `test` do Clerk.
4. **Segredo nunca** exposto; exposição = rotação. Nenhum valor de segredo é transcrito.
5. **Zero** SQL concatenado / `exec` com shell / input não validado. Dinheiro nunca em `float`.

## Como valida (dupla validação)
Executor implementa; **Validador** é um agente de segurança **distinto** (ex.: crypto validada pelo `appsec-go`; escopo validado pelo `sec-isolamento-acesso`). Achado de bypass que passa nos testes = **auto-veto** ao Security Master. Código em `internal/security` = revisão **humano + agente** (QA-12 §1).
