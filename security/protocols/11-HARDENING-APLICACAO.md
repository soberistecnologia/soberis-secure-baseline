---
protocolo: HARD-11
titulo: Hardening de Aplicação — Camadas Extras (herança NexCollabs)
status: canônico
origem: NexCollabs (o mais maduro em segurança de aplicação dos 3)
atualizado: 2026-07-21
---

# HARD-11 — Hardening de Aplicação (Camadas Extras)

> Requisito do dono: **replicar as proteções aplicadas no NexCollabs** (Argon2, e demais) como **proteções extras** do projeto. O NexCollabs foi o sistema com a melhor engenharia de segurança de *aplicação* dos três (ver [`../../\_reconhecimento/03-NEXCOLLABS.md`](../../_reconhecimento/03-NEXCOLLABS.md) §2). Aqui ficam as camadas a portar para Go.
>
> **Nota:** o sistema autentica via **Clerk**. Argon2id/sessão própria valem para **qualquer credencial local** que o sistema guarde (ex.: senha de assinatura/aprovação, PIN, credenciais de serviço, segredos cifrados em repouso) e como padrão de referência de crypto. Onde o Clerk cobre, usamos Clerk; onde guardarmos segredo/credencial nós mesmos, usamos estas camadas.

## 1. Criptografia de credenciais/segredos em repouso

| Item | Especificação (referência NexCollabs) |
|---|---|
| **KDF / hash de senha** | **Argon2id** — parâmetros OWASP 2024: `m=19456 KiB, t=2, p=1`, saída 32 bytes; senha mín. 12 chars. Salt per-usuário de 16 bytes persistido. |
| **AEAD (cifra de dados sensíveis em repouso)** | **XChaCha20-Poly1305**, nonce aleatório de 24 bytes por item. **AAD contextual** (ex.: `entidade:campo:id`) para impedir *swap* de ciphertext entre linhas. |
| **Master key** | Derivada por Argon2id, mantida **só em memória**, nunca em disco. |
| **Zeroize** | Chaves e plaintexts embrulhados em wrapper com zeragem de memória no `Drop` (equivalente Go: limpar `[]byte` sensível após uso). |

> Em Go: `golang.org/x/crypto/argon2` (IDKey), `golang.org/x/crypto/chacha20poly1305` (NewX), e limpeza explícita de buffers sensíveis. Toda essa lógica vive em `internal/security` (revisão dupla obrigatória).

## 2. Sessão & login

- **Cookie de sessão opaco** (aleatório, não JWT no cookie), **HttpOnly + SameSite=Strict + Secure** (Secure em produção). TTL definido + coleta de sessões expiradas.
- **Verificação constant-time** no login: comparar sempre contra um hash *dummy* quando o usuário não existe, para **não vazar existência por timing** (anti user-enumeration).
- **Anti-lockout atômico:** operações que mudam papéis/admins usam transação serializada (`BEGIN IMMEDIATE`/equivalente) para impedir corrida que zere todos os administradores. Com teste de concorrência dedicado.
- **2FA** para perfis aprovadores (ver [`01-RBAC-PERFIS.md`](01-RBAC-PERFIS.md)) — NexCollabs não tinha MFA; **nós adicionamos** (correção da lacuna deles).

## 3. Borda HTTP

- **CORS allowlist explícita — nunca `*`** (incompatível com credenciais). Vazio = same-origin.
- **Rate limiting** com **anti-spoof de IP**: só confiar em `X-Forwarded-For`/`X-Real-IP` quando o peer TCP está numa allowlist de proxies confiáveis (CIDR). Atacante forjando XFF fica preso ao próprio IP.
- **Security headers:** `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`, `Cross-Origin-Opener-Policy: same-origin`, **HSTS** (`max-age` longo, `includeSubDomains; preload`). Reforçar também no Traefik (defesa em profundidade).
- **Body limit** (ex.: 2 MB) + **timeout** (ex.: 30 s) nos handlers.
- **Validação de input:** rejeitar campos desconhecidos (`deny_unknown_fields`/DTO estrito), validar nomes/paths (anti path-traversal: bloquear `..`, prefixos perigosos), validar e-mail e tamanhos.
- **404 anti-enumeração:** recurso fora do escopo do usuário retorna **404**, não 403 (não vaza existência). Casado com o anti-IDOR do RBAC.

## 4. WebSocket (se houver realtime — notificações/dashboards)

- **Origin allowlist** obrigatória (rejeitar Origin ausente e literal `"null"`) — anti-CSWSH.
- **Token efêmero one-shot (TTL ~60s)** para abrir o socket, + defesa em profundidade: revalidar que a sessão pertence ao escopo.

## 5. Config fail-closed em produção

Boot **aborta** se, em produção, faltar: CORS allowlist, cookies seguros, log estruturado (JSON), TLS/HTTPS, ou se o bind expuser rota interna. Bind default restritivo. **Coberto por teste.** (Padrão `validate_production_strict` do NexCollabs.)

## 6. Segredos & logs

- Token/segredo **nunca** logado em claro (no máximo prefixo de 8 chars para correlação).
- Secrets em produção via **sops + age** e/ou **Docker Swarm secrets** — nunca em env plano versionado. Ver [`05-GESTAO-SEGREDOS.md`](05-GESTAO-SEGREDOS.md).

## 7. Auditoria com hash-chain

Já é protocolo próprio ([`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md)); o NexCollabs confirma o padrão: **append-only + hash chain + `verify_chain()` no boot**.

## 8. O que corrigimos em relação ao NexCollabs

| Lacuna deles | Nosso ajuste |
|---|---|
| Sem MFA | 2FA obrigatório para aprovadores |
| `.data/` (hashes) ia no bundle de deploy | Artefato de deploy nunca inclui estado/segredo (ver DevOps) |
| Hardening de container só no plano | Infra hardening é gate de Go/No-Go ([`08-INFRA-HARDENING.md`](08-INFRA-HARDENING.md)) |
| Supply chain Docker não pinada | Pin por digest SHA256 + scan (trivy) |

## 9. Checklist de porte (Go)

- [ ] `internal/security/crypto.go` — Argon2id + XChaCha20-Poly1305 + AAD + zeroize
- [ ] `internal/security/session.go` — cookie opaco, flags, TTL, GC, constant-time
- [ ] `internal/security/ratelimit.go` — governor + trusted-proxy XFF
- [ ] `internal/middleware/headers.go` — security headers + HSTS
- [ ] `internal/config/validate.go` — fail-closed em produção (+ teste)
- [ ] `internal/http/validation.go` — DTO estrito + anti path-traversal + 404 anti-enum
- [ ] suíte adversarial (enumeração, IDOR, XFF spoof, race de admin)
