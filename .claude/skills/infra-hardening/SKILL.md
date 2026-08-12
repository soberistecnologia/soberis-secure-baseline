---
name: infra-hardening
description: Checklist acionável de hardening de infraestrutura do Projeto Lunar (Docker Swarm, Traefik, Portainer, CIS) com comandos de trivy/grype/docker-bench. Use ao revisar stack.yml/compose/Dockerfile/config Traefik, verificar exposição de painel/daemon, pin de imagem por digest, rede/firewall/Tailscale, e antes de qualquer build/deploy. Gatilhos: "hardening", "Swarm", "Traefik", "Portainer", "trivy", "docker-bench", "CIS", "pin de digest", "deploy".
---

# Skill: Infra Hardening (Projeto Lunar)

Checklist na ponta da língua para endurecer a infra do Lunar: **Docker Swarm + Traefik + Portainer**, VPS em **Tailscale**. Use com os agentes `infra-hardening` e `runtime-detection`. Alinhado a [`security/protocols/08-INFRA-HARDENING.md`](../../../security/protocols/08-INFRA-HARDENING.md) e [`11-HARDENING-APLICACAO.md`](../../../security/protocols/11-HARDENING-APLICACAO.md). **Hardening de infra é gate de Go/No-Go, não backlog.**

## 1. Bateria de scanners (a cada build)

```bash
trivy image --severity HIGH,CRITICAL <imagem>@sha256:<digest>   # CVE na imagem
trivy config .                                                   # misconfig em Dockerfile/compose/stack
grype <imagem>:<tag>                                             # 2ª opinião de CVE
docker-bench-security                                            # CIS Docker Benchmark
docker scout cves <imagem>                                       # (opcional) CVEs + remediação
```
Regra: CVE HIGH/CRITICAL em imagem base de produção, ou misconfig crítica = **auto-veto** (No-Go).

## 2. Dockerfile — checklist

- [ ] **Pin por digest:** `FROM debian:bookworm-slim@sha256:...` — **nunca** tag mutável em produção (lacuna Locus/NexCollabs).
- [ ] **Usuário non-root:** cria usuário dedicado + `USER app`; nada roda como root.
- [ ] **HEALTHCHECK** presente.
- [ ] Multi-stage; imagem final mínima (sem toolchain de build, sem pacote desnecessário).
- [ ] Evitar instalar via `curl | bash` de fontes externas não pinadas.
- [ ] Sem segredo em `ARG`/`ENV`/camada (usar Swarm secrets em runtime).

## 3. Docker Swarm — `stack.yml`

- [ ] **Segredos via Swarm secrets** (`secrets:` → `/run/secrets/...`), nunca env plano versionado.
- [ ] Overlay network com `driver_opts` / `encrypted: "true"`.
- [ ] `deploy:` com `resources.limits` (cpu/mem) e `restart_policy`.
- [ ] Endurecimento do serviço:
```yaml
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true
      - seccomp:default            # ou perfil próprio
      - apparmor:docker-default
    read_only: true                # + tmpfs para dirs graváveis
```
- [ ] **Docker daemon nunca exposto** sem TLS/auth. **Sem `-H tcp://0.0.0.0:2375`** (o P0 do Locus: root remoto no host). Se precisar de daemon remoto → TLS mútuo + ACL Tailscale por tag, ou bind interno.
- [ ] Estado de runtime (`.data/`) **fora** do artefato de deploy (o NexCollabs empacotou hash de senha).

## 4. Traefik — borda

- [ ] Entrypoint web(80) → **redirect permanente** → websecure(443).
- [ ] **TLS** por Let's Encrypt/ACME (`certresolver`); renovação automática.
- [ ] **Security headers** no middleware (defesa em profundidade com o app — HARD-11 §3):
  - `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
  - `X-Content-Type-Options: nosniff` · `X-Frame-Options: DENY`
  - `Referrer-Policy` · `Permissions-Policy` · `Cross-Origin-Opener-Policy: same-origin`
- [ ] Roteamento por labels; **nenhuma rota interna** (`/internal`, painel, métricas) exposta publicamente.
- [ ] Dashboard do Traefik **não** público.

## 5. Portainer

- [ ] **Nunca exposto na internet.** Acesso só via Tailscale / rede privada / túnel SSH.
- [ ] Sem porta publicada no host para a UI; autenticação forte; edge agent restrito.

## 6. Rede / firewall / Tailscale

- [ ] Firewall **default-deny** (UFW): abrir só 80/443 públicos; resto fechado.
- [ ] **SSH só por Tailscale** (sem 22 público); ACL Tailscale por tag.
- [ ] Portas de banco/daemon/painel **nunca** na interface pública.
- [ ] `sslmode=disable` no banco só com mitigação explícita e documentada (túnel privado); "está no Tailscale" **não** é controle único — defesa em profundidade.
- [ ] Egress controlado onde aplicável (anti-exfil), em conjunto com `runtime-detection`.

## 7. Supply chain

- [ ] Base images pinadas por digest + atualização controlada (Renovate/Dependabot).
- [ ] Minimizar pacotes/binários instalados; preferir fontes oficiais pinadas.
- [ ] Scan de imagem no CI (trivy/grype) bloqueando HIGH/CRITICAL.

## 8. Veredito de gate (infra)

**GO** só se: sem daemon/painel exposto · imagem pinada por digest · sem CVE HIGH/CRITICAL em base · non-root + cap_drop + no-new-privileges · Traefik com TLS + headers · Portainer privado · firewall default-deny + SSH via Tailscale. Qualquer item crítico faltando = **NO-GO** → escala ao Security Master.

## 9. Fronteiras (encaminhe, não invada)

| Tema | Dono |
|---|---|
| Segurança do código Go / injeção | `appsec-go` |
| Valor/custódia de segredo, rotação | `sec-segredos` (infra garante o mecanismo Swarm secrets) |
| Integridade da trilha (hash-chain) | `auditoria-logs` |
| Anomalia em runtime / IR | `runtime-detection` |
| Prazo/limiar legal de incidente | squad LGPD (`lgpd-gov`) |
