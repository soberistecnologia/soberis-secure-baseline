---
protocolo: BDM-13
titulo: Build, Deploy e Migration — Caderno de Regras Operacional
status: rascunho (a validar — devops-swarm + security-master + Magdiel)
origem: processo REAL do One Nexus (extraído do código + vault, jul/2026), adaptado ao Lunar (Go/Swarm/gov)
atualizado: 2026-07-24
---

# BDM-13 — Build, Deploy e Migration (caderno de regras)

> **Herda o processo comprovado do One Nexus** (mesmo dono), adaptado ao Lunar (backend **Go**, Docker Swarm, órgão público). Cada regra aqui nasceu de um **incidente real** — não são preferências, são travas pra **não quebrar o sistema nem subir código sujo em produção de dinheiro público**. Complementa [`08-INFRA-HARDENING`](08-INFRA-HARDENING.md), [`05-GESTAO-SEGREDOS`](05-GESTAO-SEGREDOS.md), [`02-AUDITORIA-LOGS`](02-AUDITORIA-LOGS.md) e o gate [`../checklists/GO-NO-GO.md`](../checklists/GO-NO-GO.md).
>
> **Sem GitHub.** O deploy é **manual via SSH + scripts na VPS** — build local + `docker service update`. Nada de CI/CD, nada de push de imagem. (Confirmado: o One Nexus não tem `.github/workflows/` nem `git push` em nenhum script; git-tag, quando usado, é manual.)

## 1. Autorização — a regra-mãe (INVIOLÁVEL)

> **NUNCA buildar nem deployar sem autorização EXPLÍCITA do dev na conversa atual.** São **dois portões separados**, nunca encadeados.

1. **Portão de BUILD:** resumir o que será buildado → perguntar **"Posso fazer o build?"** → esperar "sim/pode".
2. **Portão de DEPLOY:** resumir o que será deployado → perguntar **"Posso prosseguir com o deploy?"** → esperar "sim/pode".
3. **Mesmo que o dev diga "corrige e sobe"** — confirmar **antes do build E antes do deploy** (duas confirmações). Nunca `build && deploy` num comando só.
4. **Se o dev negar → parar imediatamente.**
5. **Migration** entra sob um **pré-flight próprio** (§4) + o portão de deploy. Ordem de gates num release: **Gate 0** (pré-flight + backup) → **migration** → **Gate 1** (build) → **Gate 2** (deploy).

**Por quê (incidente real):** o dono roda **várias sessões Claude simultâneas**. Duas sessões buildando a mesma tag → "a última vence silenciosamente" → deploy do código errado em produção. Confirmação separada é a trava.

## 2. Build da imagem

- **Sempre PATH ABSOLUTO no contexto.** Nunca confiar no `pwd`, nunca `.` como contexto, nunca `cd && docker build`. *(No One Nexus o pwd ficava preso no dir do backend e buildaram a imagem errada 4+ vezes, crashando o container.)*
  ```bash
  docker build -t <tag> -f /root/projeto_vp/backend/Dockerfile /root/projeto_vp/backend
  ```
- **Mecanismo (Go): multi-stage a partir de base pública, com cache de layers.** Um stage compila o binário estático; o runtime é uma imagem mínima (distroless/scratch) só com o binário. **Não** é `FROM imagem-anterior` — cada build rebuilda o fonte e reaproveita as camadas imutáveis (ex.: `go mod download` só refaz se `go.mod`/`go.sum` mudou). `--no-cache` é opcional (usar quando desconfiar do cache).
  > *Nota:* o padrão "base pesada nunca rebuildada + overlay `FROM base`" existe no One Nexus **só** pro agente Hermes (whisper/ffmpeg/modelos). Pro Go **não se aplica** (binário estático). Fica registrado como opção **só** se algum serviço futuro tiver dependência pesada.
- **Base pinada por digest SHA256** (não tag mutável) + scan **trivy/grype** no build; achado alto = **bloqueio** ([`08-INFRA-HARDENING`](08-INFRA-HARDENING.md) §2). Container non-root, `cap_drop`, `no-new-privileges`, `read_only` onde der.
- **Artefato de deploy sem estado/segredo** — nunca embutir `.env`, credencial ou `.data/` na imagem.
- **Tag:** `v{VERSION}-{sufixo-curto}` + também `:latest`. **A próxima tag incrementa da ÚLTIMA IMAGEM BUILDADA** (`docker images | grep backend`), **não** do arquivo `VERSION` (que pode estar velho por causa de outra sessão). **Checar a tag atual ANTES de propor a nova** — é parte do Portão de Build.

## 3. Deploy

- **`docker service update --image <tag> <serviço>` — INDIVIDUAL, um serviço por vez.**
- **NUNCA `docker stack deploy` no fluxo rotineiro.** Dois motivos reais: (a) reseta **TODAS as env vars** (no One Nexus zerou `DB_PASSWORD`/`REDIS_PASSWORD`/`CLERK_KEY` e **derrubou o sistema ~30 min**); (b) sobe as **tags velhas do compose** (defasagem de ~250 versões observada). `stack deploy` só em migração de infra deliberada, com re-set imediato das envs.
- **Segredos/env:** injetar via **Docker Swarm secrets / sops+age** ([`05-GESTAO-SEGREDOS`](05-GESTAO-SEGREDOS.md)); nunca env plano versionado. Env crítica **nunca vazia** (`DB_PASSWORD`, `JWT/keys`, `CLERK_SECRET_KEY`, etc. — fail-closed no boot, [`11-HARDENING-APLICACAO`](11-HARDENING-APLICACAO.md) §5).
- **Rolling update zero-downtime:** `update_config: { parallelism: 1, order: start-first, failure_action: rollback }` + `rollback_config`. Réplicas conforme o serviço vivo (a verdade é o serviço, não o compose).
- **Health check obrigatório:** poll no `/health` (Traefik + script) esperando `status: ok` antes de considerar o deploy OK.
- **Backup ANTES** (§6) e **rollback preparado ANTES** de qualquer mudança.
- **Rollback:** `docker service update --image <tag-anterior> <serviço>` (ou `docker service rollback <serviço>`). Testado, não teórico.

## 4. Migration

- **Idempotente sempre:** `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, `DO $$ ... EXCEPTION WHEN duplicate_object`. Arquivos `.sql` numerados e versionados.
- **Roda MANUALMENTE no Postgres MASTER (VPS de dados), NÃO pela app**, via `ssh + docker exec ... psql`. *(No Lunar, respeitar a arquitetura CQRS: escrita autoritativa no MASTER; a base de **auditoria** é dedicada/só-INSERT — migration nela é raríssima e sob dupla revisão, [`02-AUDITORIA-LOGS`](02-AUDITORIA-LOGS.md).)*
- **Ordem: migration PRIMEIRO, depois build+deploy do backend.**
- **Verificação OBRIGATÓRIA:** conferir o estado **ANTES** (`SELECT to_regclass(...)`) e **DEPOIS** (`\d tabela` + `SELECT COUNT(*)`). **Backup do PG antes.** Migration que não foi verificada antes+depois **não passou**.
- **Rollback de DB é separado e por último** (`DROP` só se necessário, com backup na mão).

## 5. Versionamento / tags

- Arquivo **`VERSION`** (semver) na raiz; bump `patch|minor|major`.
- Tag da imagem incrementa da **última imagem buildada** (não do `VERSION`).
- **Imagens buildadas LOCALMENTE na VPS** — sem push pra registry no caminho real.
- **Sincronizar as tags no compose PÓS-deploy** (senão vira defasagem e o disaster-recovery sobe versão antiga).

## 6. Backup & observabilidade

- **Backup pré-deploy é OBRIGATÓRIO** (`--type pre-deploy`).
- **Sem artefato local:** backup full **streama direto pro S3 por pipe** (`tar -cf - ... | zstd | aws s3 cp - s3://...`), **zero exclusões** no full. Nomear `lunar-FULL-<version>-<YYYYMMDD>-<HHMM>.tar.zst`.
- **Falha de backup NUNCA silenciosa** *(incidente One Nexus: 14 dias sem alerta)*: `trap EXIT` que **notifica em toda execução** (sucesso E falha); defaults defensivos (`: "${VERSION:?}"`); **watcher externo** em outro servidor no mtime do log de backup (>48h → alerta); auditoria semanal de que as senhas batem entre os `.env`.
- **Notificação por canal em TODO deploy** (a notificação de **início é bloqueante** — sem ela, aborta o deploy). Trilha de auditoria do órgão registra o deploy também.
- **Log de deploy no repo/vault SEMPRE:** um `deploys/AAAA-MM-DD-vX.Y.Z-tag.md` por deploy. **Deploy sem registro = incompleto.**

## 7. Higiene / pós-deploy (INVIOLÁVEL)

- **Documentação pós-deploy é obrigatória:** CHANGELOG + tabela de serviços/imagens + doc do módulo + log de deploy + memória + notificação. **Um deploy só está COMPLETO quando tudo isso foi atualizado.** Se houve deploys sem doc, **preencher tudo antes de prosseguir**.
- **Nada quebrado, rollback pronto ANTES, um serviço por vez, ambiente de fallback vivo.** Testar conexões antes de apontar o sistema.
- **Gate [Go/No-Go](../checklists/GO-NO-GO.md)** obrigatório antes de qualquer subida pública (InfraSec + Security Master têm veto).
- **Nunca subir código sujo:** vet/lint/`go vet`/staticcheck/gosec/govulncheck + testes verdes são pré-condição do build (ver skill `appsec-go` e `12-DUPLA-VALIDACAO`).

## 8. Sem GitHub — o processo real

- Build **local na VPS** + `docker service update` via **SSH**. Sem CI/CD, sem push de imagem.
- **Sandbox → produção = scp/rsync do código-fonte + rebuild em prod** (nunca "mover a imagem de staging pra prod"). Cherry-pick manual dos arquivos.
- Git-tag, se usado, é **manual e opcional** — não faz parte do fluxo de deploy.

## 9. Diferenças que o Lunar herda mas endurece (vs One Nexus)

| One Nexus | Lunar (mais rígido) |
|---|---|
| Confirmação de encerrar sessão foi relaxada | Mantém: **build/deploy/migration sempre com autorização explícita** (é dinheiro público) |
| Réplicas divergentes compose×vivo | Compose = fonte da verdade **sincronizada** pós-deploy (rule §5) |
| Backup com incidente de 14 dias | Watcher externo + trap notify desde o dia 1 |
| — | **Toda ação de build/deploy/migration é evento de auditoria** (ator, timestamp) na trilha imutável do órgão |

> **Status:** rascunho. Antes de virar canônico: validar com `devops-swarm` + `security-master` (dupla validação, rules 6/9) e ok do Magdiel; então entrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md). Dono: Squad InfraSec (`infra-hardening`) + DevOps (`devops-swarm`).
