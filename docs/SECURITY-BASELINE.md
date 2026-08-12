# Security Baseline — a doutrina

> Base validada em produção real (auditoria + pentest com carga real). As armadilhas mais comuns — detecção/alerta ausente, segredo fora de cofre, ausência de **backstop de concorrência** na borda — já vêm **corrigidas** de saída.

## Modelo mental
**Defesa em profundidade + fail-closed.** Toda rota valida 3 coisas: *quem é* (auth) → *pode?* (authz) → *pode ESSE dado?* (ownership/escopo). Na dúvida, **nega**. A segurança é **estrutural** (o compilador e o `WHERE` cobram; o boot aborta sem config), não "disciplina de lembrar".

> **Capability-first (stack-agnostic).** Cada linha abaixo é uma **capacidade universal**. As ferramentas (Traefik, Docker, Postgres, Clerk) são **uma instância** — em VPS pura é nginx/Caddy+systemd, em K8s é `securityContext`/NetworkPolicy/admission, em PaaS é o secret store + WAF da plataforma. O time detecta a stack e aplica a capacidade (`.claude/agents/_STACK-ADAPTIVE.md`).

## As 11 camadas

| # | Camada | Software | Regra-chave |
|---|--------|----------|-------------|
| 0 | Governança | protocolos + dupla validação | fail-closed, doc-first, soft-delete, segredo nunca no git |
| 1 | Perímetro | iptables/ufw, WireGuard, fail2ban | default-deny; só 22/80/443; SSH só chave |
| 2 | Borda/TLS | Traefik / nginx / Caddy | TLS 1.2/1.3; HSTS (`preload` = opt-in verificado); headers; rate-limit **+ inFlightReq** (limite de concorrência); CORS allowlist |
| 3 | Identidade | Clerk (JWT RS256) | rejeita alg:none; valida iss/azp/exp; JWKS fail-closed; webhook HMAC |
| 4 | Autorização | RBAC próprio | deny-vence; **ownership na query**; **404 anti-enumeração** |
| 5 | Step-up 2FA | TOTP/WebAuthn/OTP | operação crítica não conclui sem prova de fator na mesma transação |
| 6 | Aplicação | Go (distroless nonroot) | SQL parametrizado; input+body limit; **dinheiro int64 centavos, zero float**; crypto conforme |
| 7 | Dados | Postgres, PgBouncer | role least-privilege; `DISCARD ALL`; escopo NOT NULL+índice |
| 8 | Auditoria | PG dedicado + outbox | append-only (role só-INSERT); hash-chain; mutação não comita sem evento |
| 9 | Segredos | Docker secrets, age | zero segredo em `Env`; gitleaks no CI |
| 10 | Container | Docker Swarm/Compose | pin por digest; non-root; cap_drop; sem docker.sock; sem daemon TCP |
| 11 | **Detecção** | logs JSON + alerta | **logs+alerta DIA 1** (o buraco nº 1 mais comum) |

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
- [ ] Rate-limit por IP **+ inFlightReq global** (limite de concorrência — contra brute/spray distribuído); CORS allowlist exata *(DDoS volumétrico = CDN/WAF, não middleware)*
- [ ] `MaxBytesReader`/limite de corpo; rejeita campo JSON desconhecido; paginação com teto
- [ ] Role de banco sem SUPERUSER; `REVOKE DELETE, TRUNCATE` se usa soft-delete
- [ ] **Logs JSON com request_id + alerta** em 5xx / falha de auth / 429  ← não deixe pra depois
- [ ] **Backup cifrado + DRILL de restauração testado** + chave offsite (anti-ransomware/perda; backup que nunca foi restaurado não existe)
- [ ] **DNS**: CAA + registrar-lock; varredura de **CNAME órfão** (anti-subdomain-takeover)

### 🔴 TIER 3 — grau-governo/financeiro
- [ ] Auditoria hash-chain (PG dedicado, outbox na mesma transação)
- [ ] Step-up 2FA por operação crítica
- [ ] RBAC granular dinâmico (deny-vence)
- [ ] Dinheiro int64 centavos + validação cruzada por 2º agente
- [ ] LGPD (bases legais, ROPA, retenção, resposta a incidente)
- [ ] **CDN/WAF na frente** (se exposto à internet) — absorve **DDoS volumétrico**, que o middleware de borda não para

## 5 armadilhas comuns que este baseline JÁ corrige
1. **Observabilidade zero** → módulo `observabilidade` (logs+alerta) já em Tier 2.
2. **Segredo em claro fora de prod** → `.gitignore` + gitleaks + módulo `secrets` desde o Tier 1.
3. **Rate-limit só por-IP, sem teto de concorrência** → `borda` já inclui `inFlightReq` global (contra brute/spray distribuído; volumétrico é CDN/WAF).
4. **Registry/painel sem auth, imagem `:latest`** → `container-hardening` exige pin por digest + auth.
5. **`app_user` com DELETE/TRUNCATE** → `banco-minimo` faz o `REVOKE` na primeira migration.

## ⚠️ Armadilhas de implementação (falso senso de segurança)
- **Swarm ignora `no-new-privileges`**: em `docker stack deploy` o `security_opt` do compose é descartado **em silêncio** — você *acha* que travou escalonamento e não travou. Mitigue com `read_only: true` + `tmpfs nosuid,nodev,noexec` + remover binários setuid (`find / -perm /6000` = 0). Em Compose funciona; `seccomp`/`apparmor` funcionam nos dois.
- **HSTS `preload` é quase-irreversível e org-wide**: submete TODOS os subdomínios a HTTPS-only por meses. Default = `max-age` longo + `includeSubDomains`; `preload` só **depois** de confirmar 100% dos subdomínios em HTTPS.
- **Rate-limit no "IP real" vs CDN**: com CDN/proxy na frente, o IP TCP é o do proxy — o tráfego colapsa num balde (auto-DoS) e o IP do cliente some do log. Extraia o IP do cliente do XFF **só** dos proxies confiáveis (`forwardedHeaders.trustedIPs`), nunca de peer arbitrário.

## Como o time de agentes entra
Cada módulo declara (em `baseline.config.json`) qual **agente** o defende e qual **skill** ele usa. Ao clonar com o time ligado, você recebe em `.claude/agents/` os especialistas certos para o tier — e o `security-master` orquestra a auditoria (triagem P0-P3, veredito Go/No-Go) com **double-check** por modelo de outra família (`docs/DOUBLE-CHECK.md`).
