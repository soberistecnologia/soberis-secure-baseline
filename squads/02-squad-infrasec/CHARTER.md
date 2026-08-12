# CHARTER — 02 Squad InfraSec

> Squad de **hardening de infraestrutura + detecção em runtime**. Reporta ao [`00 Security Master`](../00-security-master/CHARTER.md). Hardening de infra é **gate de Go/No-Go**, não backlog.

## Missão
Fechar a superfície de infraestrutura do Lunar (Docker Swarm, Traefik, Portainer, rede/Tailscale) e observar o sistema em execução para detectar anomalias e responder ao incidente inicial. Corrige os erros de referência: o **`:2375` sem TLS/auth** do Locus (P0 aceito informalmente) e o **hardening só no plano** do NexCollabs.

## Quando acionar
- Antes de **cada build/deploy** (scan de imagem, CIS, exposição de painel/daemon).
- Ao alterar `stack.yml`/compose, Dockerfile, config Traefik, rede ou firewall.
- Continuamente em produção (detecção de anomalia em runtime).
- Na **primeira resposta a um incidente** (contenção + escalada).

## Membros (→ agentes)
- [`infra-hardening`](../../.claude/agents/infra-hardening.md) — Swarm/Traefik/Portainer, trivy/grype/docker-bench/CIS, pin de digest, non-root/cap_drop, rede/Tailscale.
- [`runtime-detection`](../../.claude/agents/runtime-detection.md) — anomalia estilo Falco, logs de segurança, resposta inicial a incidente (aciona runbook LGPD art. 48 junto ao squad LGPD).
- **Skill:** [`infra-hardening`](../../.claude/skills/infra-hardening/SKILL.md) — checklist Swarm/Traefik/Portainer/CIS + comandos.

> **Fronteiras:** hardening estático (config/imagem/rede) = `infra-hardening`; observação em runtime + IR = `runtime-detection`. A **integridade da trilha** é do squad 03; o **valor** dos segredos é do squad 01 (`sec-segredos`) — o InfraSec garante o **mecanismo** (Swarm secrets), não os valores.

## O que domina
Docker Swarm seguro (secrets, overlay cifrada, non-root, cap_drop, no-new-privileges, read_only, seccomp/AppArmor), Traefik (TLS/ACME, redirect, security headers na borda), Portainer não-exposto, scanners de container/CIS, pin por digest SHA256, firewall default-deny, SSH só por Tailscale, detecção de anomalia em runtime, contenção de incidente.

## Protocolos que obedece
- [`08-INFRA-HARDENING.md`](../../security/protocols/08-INFRA-HARDENING.md) — dono do tema.
- [`11-HARDENING-APLICACAO.md`](../../security/protocols/11-HARDENING-APLICACAO.md) (§3 headers na borda; §8 pin de digest).
- [`00-PRINCIPIOS-CAMADA-0.md`](../../security/protocols/00-PRINCIPIOS-CAMADA-0.md) (§3 gate; §5 bateria container/runtime).
- [`09-LGPD-COMPLIANCE.md`](../../security/protocols/09-LGPD-COMPLIANCE.md) — runbook de incidente (com o squad LGPD).
- [`02-AUDITORIA-LOGS.md`](../../security/protocols/02-AUDITORIA-LOGS.md) — evento de runtime entra na trilha.

## Entregáveis
- Relatório datado de hardening (trivy/grype/docker-bench/CIS) com veredito de gate de infra.
- Conjunto de regras de detecção em runtime + relatório de alertas P0-P3.
- Registro de incidente e ações de contenção iniciais.

## Regras invioláveis
1. **Docker daemon nunca exposto** sem TLS/auth. **Portainer/painéis nunca públicos.**
2. Imagem base em produção sempre **pinada por digest**; HEALTHCHECK presente.
3. "Está no Tailscale" **não** é controle único — defesa em profundidade.
4. Contêiner roda **non-root** com cap_drop/no-new-privileges; overlay cifrada.
5. Nenhuma evidência de incidente é apagada; prazo/limiar legal de notificação é do squad LGPD.

## Como valida (dupla validação)
Executor endurece/desenha; **Validador** confere o scan e a config independentemente. Daemon/painel exposto, imagem não pinada em prod, ou CVE HIGH em base = **auto-veto** → No-Go. A integridade da trilha que recebe eventos de runtime é validada em conjunto com o squad 03 (`auditoria-logs`).
