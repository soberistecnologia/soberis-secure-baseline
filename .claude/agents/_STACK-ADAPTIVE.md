# Diretriz comum a TODO o time — stack-agnostic

> Vale para **todos** os agentes desta pasta. O Claude lê antes de acionar qualquer um.

**Este time não é de uma stack só.** Antes de agir, cada agente:

1. **Detecta a stack real** do projeto — linguagem, framework, banco, auth, deploy (Docker/Compose/Swarm, K8s, systemd+nginx/Caddy, PaaS), e se há IA/ML.
2. Trata qualquer menção a **Go, Clerk, Docker Swarm, Traefik, Postgres** nos agentes/protocolos como **UMA implementação de referência** — não como a regra. **Aplica o PRINCÍPIO** (ex.: "nenhuma API de orquestrador exposta sem auth") à stack que encontrar (Docker `:2375` ≡ K8s API server anonymous-auth ≡ kubelet 10250).
3. Se **não houver skill pronta** para a stack, aciona a **forja** (`CLAUDE.md`): pesquisa verificada → compila com procedência → forja skill → **double-check por modelo de outra família** → só então confia.

**Mapa capacidade → instância** (o que generalizar):
| Capacidade (universal) | Docker | nginx/Caddy+systemd | K8s | PaaS |
|---|---|---|---|---|
| Artefato imutável pinado | digest sha256 | pacote/versão fixa | image digest + admission | build lock |
| Nenhuma API de orquestrador sem auth | `:2375` fechado | N/A | API server/kubelet auth | (gerenciado) |
| Teto global de concorrência na borda | Traefik `inFlightReq` | nginx `limit_conn` | Ingress/LB rules | WAF/LB |
| Rootfs imutável + non-root | `read_only`+`cap_drop` | systemd `ProtectSystem` | `securityContext` | (gerenciado) |
| Segredo fora do env/imagem | Docker secret | systemd cred / sops | Sealed/External Secrets | secret store da plataforma |
| Rede default-deny | `DOCKER-USER`/ufw | ufw/nftables | NetworkPolicy | security groups |

Na dúvida sobre um controle numa stack nova: **ligue-o (fail-closed)** e pesquise a instância correta — nunca assuma que "não dá" só porque a referência é Docker.
