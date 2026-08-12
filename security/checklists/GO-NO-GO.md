# Checklist Go / No-Go — Deploy do Lunar

> Nenhum deploy **público** sem este checklist 100%. Squads de segurança têm **veto**. Um item aberto P0/P1 = **No-Go**.

## Autenticação & Autorização
- [ ] Clerk em **produção** (`sk_live`/`pk_live`), nunca `test`.
- [ ] JWT validado por **JWKS**; `iss`/`aud`/`azp` verificados.
- [ ] `CLERK_WEBHOOK_SECRET` setado (webhook **não** fail-open).
- [ ] **2FA** ativo para perfis aprovadores **e para o Assessor Especial** (RBAC-01 §2 — é o perfil que administra acesso). *(2026-07-25: administrador gênese `ti@soberis.com.br` criado com **senha definida ✅** e **2FA `false` ❌** — verificado na API do Clerk, não presumido. O multi-fator pode estar habilitado na INSTÂNCIA sem estar cadastrado na CONTA: são coisas distintas, e o cadastro do TOTP exige um login que ainda não ocorreu (`last_sign_in_at: None`). Fechar pelo Account Portal do Clerk antes do go-live.)*
- [ ] RBAC com **escopo/ownership** em toda leitura (anti-IDOR) — suíte adversarial passa.

## Auditoria
- [~] Trilha **append-only**, hash-chain ativa, `verificar_cadeia()` no boot. *(2026-07-24: base AUDITORIA no ar, imutabilidade provada; `verificar_cadeia()` no boot da app entra na 8.4/8.6.)*
- [x] Usuário de banco só tem **INSERT/SELECT** na auditoria (UPDATE/DELETE/TRUNCATE revogados **e** bloqueados por trigger até para o dono — prova adversarial 2026-07-24, Fase 8.2.3).
- [ ] **AUD5-06 externo:** exportação periódica do topo do hash-chain para armazenamento offsite append-only (conectar na 8.5).
- [x] **AUD5-06 standby:** réplica da AUDITORIA na VPS1 por streaming pelo túnel — prova legal nas 2 VPS (8.2.5, 2026-07-24). *Falta decidir síncrono no go-live.*
- [ ] Auditoria gravada **na mesma transação** da mutação.

## Dados
- [ ] Sem exclusão física (soft-delete); registro aprovado **travado**.
- [ ] PII/segredo **nunca** em log.

## Segredos
- [~] **gitleaks** limpo (nenhum segredo no repo). *(2026-07-24: `Credenciais Clerk.md` no cofre; `.gitignore` + `.gitleaks.toml` criados; allowlist **testada por injeção** para não ser cega.)*
  - ✅ **2026-07-28 — o workspace entrou em git, e a limpeza foi provada sobre o COMMIT, não sobre a árvore.** Varrer a árvore com cinco sessões escrevendo lê arquivos **em movimento**; varrer o histórico lê conteúdo **imutável**. Os dois rodaram: árvore **10** achados, histórico **6** — exatamente os 10 menos os 4 em caminho ignorado. Todos fixtures sintéticas triadas. Conferido também: **nenhum gitlink** no índice (repositório de terceiro aninhado vira ponteiro vazio *sem erro e sem aviso*); regex de caminho proibido do gate de backup aplicada ao índice, limpa; os dois `.env.example` com **todos** os valores sensíveis vazios, conferidos variável a variável.
  - ✅ **Gate `pre-commit` instalado**, compartilhando a **mesma** lista de triagem do gate do backup — duas listas divergiriam e a divergência apareceria como *"passa no commit e derruba o backup"*, que foi literalmente o incidente de 28/07. Fail-closed inclusive na ausência da ferramenta: `gitleaks` sumido **recusa** o commit.
  - 🟡 **Registrado para antes do primeiro `push`:** três IPs públicos em documento versionado (dois em `_reconhecimento/`, um em relatório de pentest). **Não** violam a regra 1 — a proibição é segredo *junto a* IP, e os três arquivos têm zero achado. Mas topologia de órgão público fora do perímetro é decisão de política. **Nenhum remoto foi criado.**
- [ ] 🔴 **P1 — `.env` de produção FORA da árvore do projeto.** ⚠️ **Este item afirmava, até 2026-07-24 23h, que o `.env` havia sido *movido* para `/root/lunar-credenciais/`. Era falso.** Verificado pelo `security-master`: o arquivo existe **nos dois lugares** — na raiz do repositório **e** no cofre — e a cópia da raiz é a **mais recente** (o cofre está defasado). Foi **cópia**, não movimentação. Consequências: (1) a credencial viva mora dentro da árvore que é sincronizada/empacotada/copiada como "o projeto"; (2) **a rotação vai errar o alvo** — rotacionar uma cópia deixa a outra válida e esquecida, que é exatamente como um segredo "revogado" sobrevive. Mitigações reais hoje: `0600`, `.gitignore` cobre `.env`, `.dockerignore` barra `*.env` do contexto de build, e **não há repositório git iniciado** ⚠️ *(essa última mitigação deixou de valer em 2026-07-28 — o repositório existe. O item já estava consolidado desde 25/07 e a cópia da raiz foi removida, então nada entrou no histórico; mas a frase fica corrigida aqui porque mitigação vencida que continua escrita é a mesma classe de defeito que este arquivo denuncia — o gate que mente.)*. Por isso o risco de vazamento hoje é baixo — mas **o controle declarado estava aberto**, e um gate que mente é pior que gate nenhum (é o custo do "risco aceito" informal do `:2375` no Locus). **Fechar:** consolidar em `/root/lunar-credenciais/` como fonte única, remover a cópia da raiz com `shred`, apontar o carregamento para o cofre e reverificar.
  - ✅ **CONSOLIDADO em 2026-07-25 (autorizado pelo dono).** Comparação antes de agir (por contagem, sem imprimir valores): repo 70 chaves × cofre 69; **nenhuma** chave existia só no cofre; **4** existiam só no repo; **7** chaves tinham **valor divergente** entre as cópias. Ordem executada: (1) versão anterior do cofre preservada em `.env.anterior-2026-07-25T031043Z` (`600`) — as 7 divergências ficam recuperáveis; (2) cofre passou a ser a cópia autoritativa (superconjunto, mais recente), conferida por `cmp`; (3) verificado que nenhuma chave do cofre antigo desapareceu; (4) cópia da raiz removida; (5) repo reverificado — **zero** `.env` com credencial (só `.env.example` e o `.env.local` do protótipo, que tem 3 chaves de mock sem segredo).
  - ⚠️ **Residual honesto:** a remoção foi por `rm`, **não** por `shred` como este item pedia — os blocos podem permanecer recuperáveis no disco. Mitigação de fato: o mesmo segredo continua em repouso no mesmo host (no cofre), então o ganho do `shred` seria não deixar uma **segunda** cópia recuperável. Se o dono quiser certeza, o remédio é **rotação**, não `shred`.
  - ⚠️ **Divergência não resolvida:** as **7 chaves com valor diferente** foram resolvidas em favor da cópia mais recente (a do repo). Se alguma rotação foi feita apenas no cofre, o valor correto está no `.env.anterior-*`. Um humano precisa conferir — não é possível decidir isso sem ler valor de credencial.
- [x] Segredos via **Swarm secrets** — nada em env plano versionado. *(24 secrets versionados `_v1`, gerados sem ecoar, arquivos destruídos com `shred`.)*
- [ ] 🔴 **Swarm autolock LIGADO** (cifra o raft, onde os secrets vivem) — *desligado durante a construção por decisão registrada; **ligar é pré-condição de entrega**, mesmo padrão da porta 22.*
- [ ] **Rotacionar** todos os segredos de construção antes da entrega (+ senhas de root das VPS e chave do iDrive, que vazaram no transcript).
- [~] 🟠 **P1 — `/root/lunar-credenciais/` com cópia segura própria.** ⚠️ **Este item esteve marcado `[x]` até 2026-07-27 sem que existisse cobertura — desmarcado, e depois parcialmente resolvido no mesmo dia.** ✅ **Item `cofre` implementado, executado e verificado (2026-07-27):** vai para `projeto:bkpvp/vps1/cofre/`, cifrado com **destinatário `age` dedicado** (`COFRE_AGE_RECIPIENT`), com guarda **fail-closed** no código que aborta se o destinatário faltar ou for igual ao compartilhado — os três casos testados. Conferido no artefato real: **abre com a chave dedicada, não abre com a compartilhada**. 🔴 **Continua `[~]` e não `[x]` por um motivo específico:** a privada do cofre vive só em `/root/.config/age/lunar-cofre.key` na VPS1, então o item protege contra comprometimento do bucket mas **não contra perda do host**. Vira `[x]` quando o dono assumir a custódia offsite e **provar** por decifra (passo B4 do [runbook de rotação](../../docs/deploy/RUNBOOK-ROTACAO-CHAVE-AGE.md)). *Registro do que era falso antes:* O que foi feito em 2026-07-25 foi *escrever* a cobertura, não *executá-la*: `infra/backup/backup-projeto.sh` passou a incluir o cofre no pacote cifrado, mas **nunca rodou** — não está agendado (nenhuma unidade systemd o referencia) e **não é executável no estado atual**: aponta para o remote `idrive:`, que não existe (os configurados são `projeto:` e `backup:`). **Hoje o cofre não tem backup nenhum.** O próprio item já dizia "Pendente: este script também nunca rodou agendado" ao final — e ainda assim exibia `[x]`, que é a mesma falha do escopo `projeto` da fase 8.5: **caixa marcada para controle inexistente faz quem lê parar de procurar**.
  - *Registro do que de fato existe (2026-07-25): o cofre foi movido para fora do repositório, `600`, e o script foi escrito para lê-lo e empacotá-lo — verificado por listagem do tar na época (4 membros do cofre presentes, `projeto_vp/.env` ausente). O que falta é a execução agendada.*
  - **Encaminhamento (decisão do dono, não executado):** item `cofre` no `lunar-backup.sh`, com destinatário `age` **próprio** — chave distinta da do backup de código, para que quem restaura o projeto não ganhe as credenciais junto. O item `projeto`, implementado em 2026-07-27, **exclui o cofre de propósito** e o gate do artefato reprova o pacote se ele entrar.
- [ ] Artefato de deploy **sem** `.data/`, DBs ou segredo. *(2026-07-27: vale para o **artefato de backup** do item `projeto` — `gate_artefato()` lê o índice do tar antes de cifrar e **aborta** se achar `.env*`, chave, `.data/`, DB, `*.log` ou cofre; 24 casos de regressão em `infra/backup/teste-gate-artefato.sh`; 805 membros, zero achado. **O artefato de deploy propriamente dito continua não coberto** — é item à parte.)*

## Infra
- [ ] 🔴 **SSH (porta 22) FECHADO para a internet** — acesso administrativo **só pelo túnel WireGuard** (`ListenAddress` no IP do túnel). *Durante a construção a 22 fica aberta com fail2ban+só-chave; **fechar é pré-condição de entrega ao cliente** — combinado com o dono em 2026-07-23. Antes de fechar: máquina do dono como peer do WG + console KVM confirmado.*
- [x] Nenhum painel (Portainer/DB/storage/observabilidade) público. *(2026-07-23: 2 painéis Portainer estavam públicos — removidos; `DOCKER-USER` bloqueia 9000/9443 preventivamente.)*
- [x] Docker API **não** exposta sem TLS/auth. *(verificado 2026-07-24; `DOCKER-USER` bloqueia 2375/2376.)*
- [x] Firewall fail-closed **IPv4 e IPv6** (`INPUT DROP`), persistente no boot; só 80/443/22 respondem. *(Fase 8.1)*
- [x] Rede privada entre as VPS por **WireGuard auto-hospedado** (soberania — sem control-plane de terceiro). *(Fase 8.1)*
- [x] SSH **só-chave** (`PasswordAuthentication no` efetivo), `MaxAuthTries 3`, KEX pós-quântico, banner legal (Lei 12.737/2012). *(Fase 8.1)*
- [x] **fail2ban** ativo com ban progressivo (até 1 semana) e rede do túnel na allowlist. *(Fase 8.1)*
- [x] **Avaliação de comprometimento** das VPS sem indício de backdoor. *(2026-07-24 — [relatório](../relatorios/2026-07-24-avaliacao-comprometimento.md))*
- [x] **AC-01** — bind do Swarm (2377/7946) fixado no IP do túnel, não em `*`; `docker.service` ordenado após `wg-quick@wg0`; tokens de join rotacionados. *(2026-07-24; verificado por teste de alcance externo nas duas VPS.)*
- [x] ✅ **P1 FECHADO em 2026-07-25 — Protótipo público endurecido.** Aplicado com autorização do dono, após **ensaio em container descartável** com o mesmo conjunto de opções. Conferido por `service inspect` (não pelo arquivo): `CapabilityDrop=["ALL"]` + só `CHOWN, NET_BIND_SERVICE, SETGID, SETUID`; rootfs somente-leitura; 3 tmpfs; config `_v2` com **CSP** e bloqueio de dotfile; **HSTS**/nosniff/frameDeny/rate-limit no Traefik. Site verificado: raiz 200, deep-link 200, **`/.env` 403**.
  - ⚠️ **`no-new-privileges` NÃO é aplicável a serviço Swarm** — `security_opt` do compose é ignorado em silêncio e o `docker service update` (Docker 29.6.2) não expõe a flag. **Risco residual medido, não presumido:** `find / -perm /6000` na imagem → **0 binários setuid/setgid**; rootfs somente-leitura **provado por tentativa real de escrita**; todos os pontos graváveis são tmpfs `nosuid,nodev,noexec` (lido de `/proc/mounts` do container); bounding set já limitado a 4 capabilities. Sem caminho prático de escalada. **Reavaliar se a imagem base mudar** — é a premissa da análise. A diretiva foi mantida no arquivo de propósito: removê-la esconderia a lacuna de quem revisar depois.
- [ ] ~~🔴 P1 — Protótipo público endurecido~~ *(texto original preservado abaixo para rastreabilidade da decisão)* O serviço do protótipo está no ar **sem `cap_drop`, sem `no-new-privileges` e com o processo master como root** (`Privileges.NoNewPrivileges=false`, `CapabilityDrop` vazio no `service inspect`). É o 08-INFRA-HARDENING §2 "só no plano" — o erro que CLAUDE.md §2 promete não repetir. **A correção já existe e já foi testada** (`stack-prototipo.yml` + `nginx-prototipo.conf` `_v2`, validados em container descartável), **só não foi aplicada**: aplicar é `deploy` = **humano-obrigatório** (PRIN-00 §4), nunca ato de agente. Exposição prática exige antes um RCE no nginx (serve estático, sem API, sem dado, `/root` 0700) — logo **não é emergência de madrugada**, é a correção de melhor relação custo/benefício em aberto. **Aplicar na próxima janela com humano, ≤48h.** Dono `infra-hardening` + `devops-swarm`.
- [ ] Linha-de-base de integridade de pacotes (`debsums`) executada.
- [x] Traefik: 80→443, TLS **Let's Encrypt válido** (`apivp`), **HSTS**/nosniff/frameDeny + rate-limit por IP na borda. *(Fase 8.4)* — falta o dashboard do Traefik atrás de auth.
- [x] Base images **pinadas por digest** (Postgres/PGBouncer/Rabbit/Redis/ES + os 3 serviços Go via registry no túnel); **trivy** 0 HIGH/CRITICAL nas imagens do app. *(Fase 8.3/8.4)*
- [x] **API nunca exposta na internet** — só overlay; Traefik (443) roteia por Host; IP cru = 404. *(Fase 8.4)*
- [x] Segredo via **`_FILE`/Docker secret** (nunca env plano); **validação por papel** (menor-privilégio). *(Fase 8.4)*
- [~] Backup cifrado (`age`) — **script** testado com restauração real e retenção provada (Fase 8.5), mas o **agendamento nunca executou**. *(Verificado pelo `security-master` em 2026-07-24 23h35, somente-leitura, nas **duas** VPS: `lunar-backup.timer` `ActiveState=active`, `Persistent=yes`, **`LastTriggerUSec=` vazio** e `ExecMainStartTimestamp=` vazio → a **unit** jamais rodou; primeiro disparo previsto 2026-07-25 04:00/04:03 BRT.)* **A distinção importa:** o que foi provado foi o **script**, executado à mão; o que **não** foi provado é o caminho do systemd — ambiente, `PATH`, ausência de TTY, leitura da credencial mínima —, que é precisamente onde backup agendado falha em silêncio. **P2 hoje** (sistema em construção, sem dado de cliente, mecanismo comprovado). **Vira P1 e bloqueia a entrega** se a execução de 2026-07-25 04:00 não produzir artefato cifrado nas duas VPS. **Verificação obrigatória em 2026-07-25** — dono `devops-swarm`.
  - ✅ **Agendamento comprovado (conferido em 2026-07-27):** `lunar-backup.timer` `enabled`+`active`, última execução **2026-07-27 04:01:10**, `status=0/SUCCESS`, com artefatos correspondentes no bucket (5 ciclos em `vps1/`, 4 em `vps2/`). O caminho do systemd está provado.
  - ⚠️ **Mas o escopo estava incompleto, e isso não aparecia em lugar nenhum.** A fase 8.5 declarava o item **`projeto` → `bkpvp`** desde 24/07; o script **nunca o escreveu**. O timer ficou verde por dias sobre um escopo que não cumpria — não falhou, o item não existia. Cobrado em **2026-07-27**, quando uma publicação do front quebrou a produção e não havia artefato anterior. **Corrigido no mesmo dia** (item implementado, executado e com restauração testada — 805 membros em 8 s). **Lição para esta checklist: "o agendamento executou" não é o mesmo que "o escopo prometido foi coberto"** — verificar os dois.
  - ⚠️ O item **`logs`** continua **sem drill de restauração** próprio.
- [ ] 🔴 **P0 — Chave privada `age` guardada OFFSITE pelo dono** — sem ela o backup é irrecuperável. ⚠️ **Agravado em 2026-07-27:** a chave privada não está apenas fora do offsite — ela está **dentro do próprio bucket de backup** (`projeto:bkpvp/lunar-backup.key`, contém `AGE-SECRET-KEY`, verificado por contagem sem imprimir byte algum) e **idêntica em SHA-256** à do host produtor (`/root/.config/age/lunar-backup.key`, VPS1). Quem obtiver a credencial do bucket obtém **cifra e chave juntas** — MASTER, **AUDITORIA (prova legal)** e o projeto inteiro. **Nada foi removido de propósito:** apagar a única cópia acessível antes de o dono confirmar posse da cópia offsite trocaria um problema de confidencialidade por perda irreversível de todos os backups. **Inventário completo (2026-07-27, por varredura de conteúdo):** existem **exatamente duas** cópias, **idênticas** — a do host e a do bucket. **Nenhuma na VPS2, nenhuma no cofre, nenhuma no `bckpbancodelogs`, nenhuma no repositório.** Ou seja: **não há hoje cópia independente conhecida**, e é por isso que remover qualquer uma sem a confirmação do dono é inaceitável. **Provado por drill com a cópia do host** (não a do bucket): projeto ✅ 805 membros/8 s **e** bancos ✅ `pg-master` + `pg-auditoria`/4 s — logo a cópia do bucket é **redundante**, e removê-la não custa capacidade de restauração. Sequência completa, com os pontos irreversíveis marcados, em [`docs/deploy/RUNBOOK-ROTACAO-CHAVE-AGE.md`](../../docs/deploy/RUNBOOK-ROTACAO-CHAVE-AGE.md) §7 — **preparada, não executada**; o passo A5 (confirmação escrita do dono de que possui a chave fora da VPS1 e fora do bucket) está **aguardando resposta**.
- [x] ✅ **`pg-auditoria` com destinatário `age` próprio — EM VIGOR (2026-07-27), provado em artefato real.** `AGE_RECIPIENT_PG_AUDITORIA` ativado **na VPS2** (onde o dump da trilha nasce; a ausência da variável no env da VPS1 é correta, não lacuna). Medido no `pg-auditoria-20260727T205142Z.age`: **não abre** com a chave operacional, **abre** com a da auditoria e com a de custódia. Controle contra efeito colateral: `pg-master` recém-gerado **abre** com operacional + custódia e **não abre** com a da auditoria. **Quem opera backup de aplicação deixou de conseguir abrir a prova legal.** A custódia entra em **todo** item desde o primeiro artefato, então nenhum fica atrás de uma única chave. 🔴 *Pendente à parte: a privada da auditoria ainda vive só na VPS1 — ver o item de custódia.* **Motivo original do item:**
- [x] ✅ **P0 FECHADO — chave privada `age` removida da raiz do bucket `bkpvp` (2026-07-27, B7 sob GO do dono).** Raiz do bucket **vazia**. Antes de remover: conferido que a cópia do bucket era idêntica à do host (SHA-256), que a do host decifra o artefato mais recente, e que a pública bate com a linha verificada pelo dono. Depois: **os dois drills continuam passando** com a cópia do host. Operação **reversível** — a cópia do host existe. 🔴 **B8 (remover do host) e B11 (destruir a velha) seguem BLOQUEADOS:** o dono confirmou a **linha pública** (prova de *identidade*) mas **não** a impressão digital `f44cbf74…` (prova de *integridade*) — um byte perdido no download deixa o arquivo idêntico à vista e inútil para decifrar.
- [ ] 🔴 **P1 — custódia offsite das chaves que vivem só na VPS1.** ⚠️ **Atualizado em 2026-07-28: são CINCO, não três** — `cofre`, `auditoria`, `custodia` (27/07), a **operacional nova** (28/07) e a **operacional velha**, que segue necessária até o rollover terminar. Todas vivem **só na VPS1** — o mesmo defeito que estamos consertando, multiplicado por três. Enquanto não saírem: `cofre` e `auditoria` protegem contra comprometimento do **bucket**, não contra perda do **host**; e a `custodia` **não é custódia**, porque quebra-vidro guardado na máquina que ele deveria socorrer não socorre nada. Procedimento, linhas públicas e impressões digitais em [`RUNBOOK-ROTACAO-CHAVE-AGE.md`](../../docs/deploy/RUNBOOK-ROTACAO-CHAVE-AGE.md) §8. **Canal de entrega deliberadamente não definido** — chave privada não passa por e-mail, chat, bucket nem sessão de agente.
- [x] ✅ **Drill de restauração agendado — Nível 1 (2026-07-27).** `lunar-vigia-backup.timer` (05:30) roda `vigia-backup.sh`, que **não usa chave** e confere existência, frescor, tamanho e retenção dos **7 itens das duas VPS**. Resolve a contradição de os drills mandarem remover a chave de que precisavam: o que dá para automatizar sem chave foi automatizado; o Nível 2 (restauração real) é manual **de propósito**, e o Nível 1 **reprova** se ele passar de 30 dias sem rodar. Unidades **novas**; `lunar-backup.{service,timer}` **intocados**. *(O Nível 1 não prova restauração — só o Nível 2 prova, e o próprio script diz isso.)*
- [x] ✅ **Rotação da chave principal — EXECUTADA em 2026-07-28, sob GO do dono, provada por decifra.** Nova operacional gerada em arquivo `600`, nunca ecoada; só a linha `AGE_RECIPIENT=` mudou, nas **duas** VPS, com a anterior guardada ao lado. Ciclo manual completo e **os dois drills de Nível 2 verdes** contra os artefatos novos (`projeto` 1.782 membros/12 s; `pg-master` 31 tabelas; `pg-auditoria` 2 tabelas). Matriz medida no bucket real: a **velha não abre nenhum artefato novo** (a rotação pegou), a **nova não abre `pg-auditoria`** (a segregação sobreviveu), e a **custódia abre as duas gerações** — logo em instante nenhum houve janela com artefato ilegível. ⚠️ **Ressalva:** a recomendação era rotacionar **depois** de B8; o dono autorizou **antes**, com B8 bloqueado. Efeito aceito: **cinco** privadas na VPS1 em vez de quatro. Nenhuma capacidade de restauração foi perdida; o risco residual (perder a VPS1) não foi criado nem agravado pela rotação. 🔴 **A chave velha NÃO foi destruída e não pode ser** até `infra/backup/verifica-rollover.sh` aprovar (dia 0: **5 itens ainda dependem dela**; previsão ≈2026-08-02) **e** a custódia ser provada **e** haver GO — os três, não um.
- [ ] ~~🔴 **P1 — `pg-auditoria` deve ter destinatário `age` próprio.**~~ *(resolvido acima)* Hoje há **um único** `AGE_RECIPIENT` para todos os itens, e isso desmancha no backup uma segregação que o banco impõe (Postgres dedicado, papel escritor só-INSERT, hash-chain). **Medido em 2026-07-27:** com a chave que está na raiz do `bkpvp` — o bucket do backup de **código** — foi decifrado o `pg-auditoria`, que vive no **outro** bucket. Comprometer o ativo de menor valor entrega a **prova legal**. Recomendação e argumento em [`RUNBOOK-ROTACAO-CHAVE-AGE.md`](../../docs/deploy/RUNBOOK-ROTACAO-CHAVE-AGE.md) §3; o código **já resolve destinatário por item** (`AGE_RECIPIENT_PG_AUDITORIA`) e **já aceita um segundo destinatário de custódia**, então é mudança de configuração, não de código.

## Aplicação (hardening)
- [ ] Config **fail-closed** em produção (boot aborta se faltar hardening).
- [ ] CORS allowlist (nunca `*`); rate-limit anti-XFF-spoof; security headers.
- [ ] Crypto em `internal/security` com **revisão dupla**.

## Qualidade
- [ ] **Dupla validação** concluída (Executor ≠ Validador).
- [ ] **Integridade financeira** validada por recálculo independente (zero erro).
- [ ] **Revisor de português** aprovou todos os textos.

## Conformidade
- [ ] Runbook de incidente (LGPD art. 48 / Res. 15/2024) pronto e testado.
- [ ] ROPA (registro de operações) atualizado.

## Segurança geral
- [ ] SAST (gosec/staticcheck) + govulncheck sem achado alto.
- [ ] Último **security-review** sem P0/P1 aberto.
- [ ] Entrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md).

---

## Veredito corrente

**2026-07-24 — ☒ NO-GO para deploy público** — `security-master`, veto exercido (PRIN-00 §3).

> **2026-07-27 — ☒ VETO ADICIONAL E INDEPENDENTE: NO-GO para qualquer IMPORTAÇÃO REAL de planilha.**
> Escopo próprio, que **não se confunde** com o NO-GO de deploy acima: alcança o ato de importar
> dado de origem para o domínio, inclusive em ensaio com arquivo real. Motivo: o portão aritmético
> **falha aberto** — `Classificar` nunca compara o local do desvio com o que a regra reconheceu, e a
> faixa que bloqueia deixa de existir sobre a pasta de trabalho real (3 P0, provados em teste).
> Classe: **falsificação de conferência**. **9 condições suspensivas numeradas**, donos e tiers em
> [`../relatorios/2026-07-27-veto-portao-aritmetico-importacao.md`](../relatorios/2026-07-27-veto-portao-aritmetico-importacao.md).
> **O veto é da importação, não do trabalho** — construção, correções, testes e doc seguem liberados.

> **Atualização 2026-07-25:** o P1 do **`.env` na árvore do projeto foi FECHADO** (consolidado no
> cofre, com o backup do cofre consertado no mesmo ato — ver o item correspondente). Os demais
> motivos do NO-GO continuam de pé, então **o veredito não muda**: o P0 do ADR-0006 sozinho o
> sustenta. Duas ressalvas ficaram registradas no item: remoção por `rm` em vez de `shred`, e 7
> chaves com valor divergente resolvidas em favor da cópia mais recente (o valor antigo está
> preservado, mas conferir qual é o correto exige um humano ler credencial).

> **Atualização 2026-07-25 (Validador independente) — o P0 da ADR-0006 está FECHADO.** Verificado
> sem confiar no relato do executor: regra de resolução (`CobreComoGrant`), recusa de `admin:*:*`
> como grant na gravação, e a suíte de invariante — **122 PASS / 0 FAIL / 0 SKIP** contra Postgres
> descartável. As **5 condições do veto** foram conferidas uma a uma (ver `CHANGELOG-SECURITY.md`,
> entrada "Validação independente de B1.1, B1.3, ADR-0006 e ADR-0007"): 1, 3 e 5 **cumpridas**; 2 e
> 4 **cumpridas no Go, com ressalva** (segundo motor de resolução no front; condição 4 vale por uma
> camada só). **O veredito não muda** — o P1 do protótipo e os 🔴 de entrega seguem de pé.

**Motivo em uma linha:** ~~1 P0 (ADR-0006)~~ **fechado em 2026-07-25**; permanecem **P1 abertos**
(protótipo público sem `cap_drop`/`no-new-privileges`; anti-lockout sem enforcement) e os itens 🔴
já conhecidos: **porta 22 aberta**, **Swarm autolock desligado**, **segredos de construção não
rotacionados** e **chave `age` ainda não guardada offsite pelo dono**.

**O NO-GO é do *deploy público*, não do trabalho.** Continua **liberado**: construção, sub-fase
**B1.2** assim que cumpridas as suas quatro condições (ver `CHANGELOG-SECURITY.md`, entrada
"Consolidação de segurança da B1.1"), correções de documentação e os itens em paralelo. Nada aqui
autoriza `migration`, build de imagem ou deploy por agente — os três permanecem
**humano-obrigatório**.

**Riscos aceitos, explícitos e datados** (PRIN-00 §3 — risco aceito sem registro não existe):
porta 22 aberta e Swarm autolock desligado **durante a construção**, ambos com fechamento como
**pré-condição de entrega** (combinado com o dono em 2026-07-23). Nenhum outro risco está aceito.

**Assinatura:** Security Master ☒ (2026-07-24) + humano responsável __________ (pendente)

**Veredito da entrega:** ☐ GO ☐ NO-GO — assinado por: Security Master + __________
