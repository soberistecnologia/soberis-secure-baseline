# Security Baseline — a doutrina

> Destilado do sistema **Lunar** (auditado + pentestado com carga real, 2026-08). As camadas 0–8 estão bem construídas; as fragilidades do Lunar (detecção, segredos fora de prod, backstop anti-DDoS) já vêm **corrigidas** aqui.

## Modelo mental
**Defesa em profundidade + fail-closed.** Toda rota valida 3 coisas: *quem é* (auth) → *pode?* (authz) → *pode ESSE dado?* (ownership/escopo). Na dúvida, **nega**. A segurança é **estrutural** (o compilador e o `WHERE` cobram; o boot aborta sem config), não "disciplina de lembrar".

## As 11 camadas

| # | Camada | Software | Regra-chave |
|---|--------|----------|-------------|
| 0 | Governança | protocolos + dupla validação | fail-closed, doc-first, soft-delete, segredo nunca no git |
| 1 | Perímetro | iptables/ufw, WireGuard, fail2ban | default-deny; só 22/80/443; SSH só chave |
| 2 | Borda/TLS | Traefik, nginx | TLS 1.2/1.3 só; HSTS+headers; rate-limit **+ inFlightReq**; CORS allowlist |
| 3 | Identidade | Clerk (JWT RS256) | rejeita alg:none; valida iss/azp/exp; JWKS fail-closed; webhook HMAC |
| 4 | Autorização | RBAC próprio | deny-vence; **ownership na query**; **404 anti-enumeração** |
| 5 | Step-up 2FA | TOTP/WebAuthn/OTP | operação crítica não conclui sem prova de fator na mesma transação |
| 6 | Aplicação | Go (distroless nonroot) | SQL parametrizado; input+body limit; **dinheiro int64 centavos, zero float**; crypto conforme |
| 7 | Dados | Postgres, PgBouncer | role least-privilege; `DISCARD ALL`; escopo NOT NULL+índice |
| 8 | Auditoria | PG dedicado + outbox | append-only (role só-INSERT); hash-chain; mutação não comita sem evento |
| 9 | Segredos | Docker secrets, age | zero segredo em `Env`; gitleaks no CI |
| 10 | Container | Docker Swarm/Compose | pin por digest; non-root; cap_drop; sem docker.sock; sem daemon TCP |
| 11 | **Detecção** | logs JSON + alerta | **logs+alerta DIA 1** (o buraco nº 1 do Lunar) |

## Checklist por tier (gate de PR)

### 🟢 TIER 1 — SEMPRE (até formiga)
- [ ] Firewall default-deny; só 22/80/443; SSH só chave + fail2ban
- [ ] TLS + HSTS + X-Frame-Options DENY + nosniff + Referrer-Policy; HTTP→HTTPS
- [ ] `.env*`/`*.key` no `.gitignore`; **gitleaks no CI**; zero segredo commitado
- [ ] SQL 100% parametrizado; erro genérico (sem stack/SQL ao cliente)
- [ ] Container `USER nonroot`; sem `docker.sock` no app; sem daemon Docker em TCP

### 🟡 TIER 2 — se tem usuários/dados
- [ ] Auth gerenciada (Clerk); JWT valida alg fixo/iss/exp/azp; JWKS fail-closed
- [ ] **Ownership na query** (`WHERE owner=$user`), não `if` depois; **404** (não 403) para recurso alheio
- [ ] Rate-limit por IP **+ inFlightReq global**; CORS allowlist exata
- [ ] `MaxBytesReader`/limite de corpo; rejeita campo JSON desconhecido; paginação com teto
- [ ] Role de banco sem SUPERUSER; `REVOKE DELETE, TRUNCATE` se usa soft-delete
- [ ] **Logs JSON com request_id + alerta** em 5xx / falha de auth / 429  ← não deixe pra depois

### 🔴 TIER 3 — grau-governo/financeiro
- [ ] Auditoria hash-chain (PG dedicado, outbox na mesma transação)
- [ ] Step-up 2FA por operação crítica
- [ ] RBAC granular dinâmico (deny-vence)
- [ ] Dinheiro int64 centavos + validação cruzada por 2º agente
- [ ] LGPD (bases legais, ROPA, retenção, resposta a incidente)

## Os 5 erros do Lunar que este baseline JÁ corrige
1. **Observabilidade zero** → módulo `observabilidade` (logs+alerta) já em Tier 2.
2. **Segredo em claro fora de prod** → `.gitignore` + gitleaks + módulo `secrets` desde o Tier 1.
3. **Rate-limit só por-IP** → `borda-traefik` já inclui `inFlightReq` global.
4. **Registry/painel sem auth, imagem `:latest`** → `container-hardening` exige pin por digest + auth.
5. **`app_user` com DELETE/TRUNCATE** → `banco-minimo` faz o `REVOKE` na primeira migration.

## Como o time de agentes entra
Cada módulo declara (em `baseline.config.json`) qual **agente** o defende e qual **skill** ele usa. Ao clonar com o time ligado, você recebe em `.claude/agents/` os especialistas certos para o tier — e o `security-master` orquestra a auditoria (triagem P0-P3, veredito Go/No-Go), exatamente como na auditoria do Lunar.
