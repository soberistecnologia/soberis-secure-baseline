---
protocolo: AUTH-04
titulo: Autenticação com Clerk
status: canônico
atualizado: 2026-07-21
---

# AUTH-04 — Autenticação (Clerk)

> **Clerk autentica; o Lunar autoriza.** A identidade vem do Clerk; o **perfil/permissões** vivem no nosso RBAC ([`01-RBAC-PERFIS.md`](01-RBAC-PERFIS.md)), não no Clerk. Este protocolo corrige os erros de auth mapeados no One Nexus e no Locus.

## 1. Validação de token (correções dos erros mapeados)

- ✅ **Validar JWT por JWKS/chave pública** (RS256), **não** com a secret key. *(Erro do One Nexus: `secretOrKey = CLERK_SECRET_KEY` — conceitualmente errado.)*
- ✅ **Validar `iss` e `azp`** (authorized parties) — defesa contra reuso de chave entre ambientes e CSRF. *(One Nexus não validava `iss`/`aud`; Locus corrigiu `azp` num bundle.)*
  - **Nota sobre `aud`:** o token de **sessão** do Clerk **não emite `aud`** por padrão; a defesa equivalente é **`azp` + `iss`** (ambos validados). Portanto `aud` **não é exigido** na verificação de sessão. Se um dia o Clerk emitir `aud` (ex.: templates de JWT), aí sim validá-lo. *(Decisão registrada na Fase 3; ver `internal/auth/verifier.go`.)*
- ✅ **Nunca rodar produção na instância `test` do Clerk.** Produção usa `sk_live`/`pk_live`. *(Erro do Locus: `sk_test` em produção.)*
- ✅ **`CLERK_WEBHOOK_SECRET` sempre setado** — webhook sem verificação de assinatura é **fail-open** e pode criar admin. Fail-closed: sem secret, rejeita. *(Erro crônico do One Nexus.)*

## 2. Sessão & 2FA

- Sessão conforme [`11-HARDENING-APLICACAO.md`](11-HARDENING-APLICACAO.md) (flags seguras).
- **2FA obrigatório para perfis aprovadores** (Diretor-Presidente, Diretor Financeiro) — no **login**.
- ⚠️ **Correção 2026-07-24 — o step-up de aprovação NÃO é o 2FA do Clerk.** O Clerk é **só
  login/identidade** (sessão reautenticada, JWT via JWKS, `iss`/`aud`/`azp` validados). A
  **assinatura do ato de aprovação é mecanismo próprio** do Lunar, em `internal/security`
  (**WebAuthn/FIDO2** assinando o `payload_hash`; TOTP só como tier fraco limitado por valor).
  A identidade do ato continua sendo a sessão Clerk (`clerk_user_id`), ligada ao mesmo `user_id`
  que o RBAC reconhece como aprovador. Spec completa em
  [`06-FLUXO-APROVACAO.md`](06-FLUXO-APROVACAO.md) §2.

## 3. Provisionamento & papéis

- Cargo inicial do Lunar semeado a partir do papel do Clerk, mas a **fonte de verdade de autorização é o RBAC do Lunar**.
- Toda criação/alteração de usuário via webhook Clerk é **auditada** ([`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md)).

## 4. Segredos

Chaves do Clerk (`sk_live`, `pk_live`, `CLERK_WEBHOOK_SECRET`, `CLERK_JWT_KEY`) seguem [`05-GESTAO-SEGREDOS.md`](05-GESTAO-SEGREDOS.md). Dono: Squad AppSec (`sec-auth-webhooks`).
