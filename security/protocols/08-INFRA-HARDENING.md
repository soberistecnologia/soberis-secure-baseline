---
protocolo: INFRA-08
titulo: Hardening de Infraestrutura
status: canônico
atualizado: 2026-07-26
---

# INFRA-08 — Hardening de Infraestrutura

> Infra igual à dos sistemas irmãos: **Docker Swarm + Portainer + Traefik**. Aqui ficam as regras para **não repetir** a superfície exposta do One Nexus (5× Portainer públicos, Adminer, MinIO, dev-server, Swagger) nem o Docker `:2375` aberto do Locus.

## 1. Regras de exposição

1. **Nenhum painel administrativo público:** Portainer, banco (Adminer/DBeaver), storage console, dashboards de observabilidade ficam **atrás de túnel WireGuard/VPN/firewall** — nunca na internet.
2. **Docker API nunca exposta sem TLS/auth** (`:2375` proibido aberto). *(P0 aceito no Locus — aqui é bloqueio.)*
3. **Traefik** é a única porta de entrada: 80→443 redirect, **TLS Let's Encrypt**, security headers/HSTS (defesa em profundidade com o app), dashboard `api.insecure=false`.
4. Portas mínimas; sem serviço de debug/dev em produção; `ENV=production` real (sem stack trace/Swagger público).
5. Comunicação entre nós por rede privada (túnel WireGuard / overlay `encrypted`).

## 2. Imagens & supply chain

- **Pin por digest SHA256** nas base images (não tags mutáveis). *(Correção do NexCollabs.)*
- **trivy** (image + config) + **grype** + **docker-bench**/CIS a cada build. Achado alto = bloqueio.
- Container non-root, `cap_drop`, `no-new-privileges`, `read_only` onde possível.

## 3. Papéis de banco — o dono do schema nunca é a aplicação

> Acrescentado em 2026-07-26 depois que dois validadores independentes provaram, em PG 17.10 e com
> papel **não superusuário**, que o papel da aplicação do MASTER apagava uma trilha de assinaturas
> protegida por gatilho. Ver [`CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md) e
> [`infra/swarm/vps2/manutencao/PROCEDIMENTO-SEPARACAO-DONO-APP.md`](../../infra/swarm/vps2/manutencao/PROCEDIMENTO-SEPARACAO-DONO-APP.md).

1. **Toda base tem um papel DONO e um papel de APLICAÇÃO, e eles são diferentes.** Em PostgreSQL,
   *ownership* é privilégio: o dono desliga gatilho (`ALTER TABLE … DISABLE TRIGGER`), reescreve
   função de gatilho (`CREATE OR REPLACE FUNCTION`, que **não** altera `pg_trigger.tgenabled`),
   faz DDL e reconcede privilégio. Gatilho de imutabilidade só vale contra quem **não** é dono.
2. **`CREATE DATABASE … OWNER <papel_da_aplicação>` é proibido.** No PG 15+ o schema `public`
   pertence a `pg_database_owner`; dar a base à aplicação dá o schema junto.
3. **O papel da aplicação não tem `CREATE` no schema, nem `TEMP` na base, nem `TRUNCATE`, nem
   `UPDATE` em sequência** (`UPDATE` em sequência é `setval`, isto é, rebobinar identificador).
4. **`DELETE` é nominal, nunca padrão.** `ALTER DEFAULT PRIVILEGES` do papel DDL concede
   `SELECT, INSERT, UPDATE` — jamais `DELETE`. Tabela que precise de exclusão física recebe o
   `GRANT` explícito na própria migração, onde a concessão fica datada e revisável (§5.2 e §5.7 da
   Constituição).
5. **A credencial do papel DDL vive num secret montado só no migrador.** Não é disciplina de quem
   opera: o contêiner da aplicação não declara o secret e por isso não tem como lê-lo. A janela de
   uso do papel DDL é a vida do serviço efêmero de migração.
6. **A aplicação só lê `schema_migrations`.** Se ela puder gravar, pode declarar aplicada uma
   migração que nunca correu — e a recusa de boot por schema atrasado deixa de valer.
7. **Verificação de gatilho compara o CORPO da função, não só o estado.** `tgenabled` sozinho não
   pega a troca silenciosa. A consulta canônica (`md5(pg_proc.prosrc)` por gatilho) está no modo
   `conferir` de
   [`11-separar-dono-app.sh`](../../infra/swarm/vps2/manutencao/11-separar-dono-app.sh).
8. **Mudança de propriedade e privilégio vai numa transação única**, com `lock_timeout`. `ALTER …
   OWNER` reescreve a ACL: fora de transação, a ordem errada tira o privilégio da aplicação antes de
   devolvê-lo e derruba o sistema.

Estado por base:

| Base | Dono | Aplicação | Situação |
|---|---|---|---|
| AUDITORIA (`auditoria`) | `lunar_audit_admin` | `lunar_audit_writer` (só `INSERT`/`SELECT`) | ✅ desde o início |
| MASTER (`lunar`) | `lunar_ddl` | `lunar_app` (DML mínimo) | 🔨 scripts e provas prontos; **pendente de GO e de dois bloqueantes** (ver o procedimento, §3 e §7) |

## 4. Backup & continuidade

- Backup **cifrado com `age`** → destino remoto (S3/iDrive), **rotação** definida, **restauração testada** (padrão Locus). Chave privada offsite.
- A trilha de auditoria entra no backup imutável ([`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md)).

## 5. Gate Go/No-Go

Nenhum deploy público sem o checklist [`../checklists/GO-NO-GO.md`](../checklists/GO-NO-GO.md). Squad InfraSec (`infra-hardening`) + Security Master têm veto. Secrets: [`05-GESTAO-SEGREDOS.md`](05-GESTAO-SEGREDOS.md).
