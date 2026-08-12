---
protocolo: AUD-02
titulo: Auditoria de Logs — Trilha Total e Imutável
status: canônico
prioridade: MÁXIMA
fase: 1
atualizado: 2026-07-21
---

# AUD-02 — Auditoria de Logs (Trilha Total e Imutável)

> **⭐ PRIORIDADE MÁXIMA.** Requisito do cliente: *"registrar quem criou, alterou, aprovou ou excluiu cada informação, com data e hora. O log não deve poder ser apagado pelo usuário."* Diretriz do dono: **cada "respiração" dentro do sistema fica registrada — não pode passar nada.**
> Contexto governamental: a trilha é prova de accountability perante controle interno / TCU. Adulterar ou perder trilha é falha crítica.

## 1. Princípios inegociáveis

1. **Append-only.** Só se insere. **Nunca** update/delete numa entrada de auditoria — por ninguém, incluindo administradores e o próprio banco de aplicação.
2. **Imutável e à prova de adulteração.** Encadeamento por hash (cada registro carrega o hash do anterior → *hash-chain*). Qualquer alteração quebra a cadeia e é detectável.
3. **Total.** Registra **toda** ação relevante: autenticação, autorização (inclusive **negada**), CRUD de negócio, aprovação, exportação, mudança de perfil/permissão, acesso a dado pessoal, e ações administrativas.
4. **Fora do alcance do usuário.** A aplicação (e seus usuários) tem permissão de **INSERT apenas** na trilha. Nenhuma rota, tela ou papel expõe delete/edit de log.
5. **Segregado em banco dedicado.** A trilha vive num **Postgres dedicado à auditoria** — instância própria, isolada do banco de negócio, com **credencial só-INSERT**. Mesmo que a aplicação seja comprometida, a prova não é adulterada. O **Elasticsearch** apenas **indexa** a trilha para **busca rápida** (por quem/módulo/data/ação) — é **mutável**, então **nunca é a prova legal**, só o buscador.
6. **Datado com precisão e fuso.** Timestamp em UTC + fuso local, com relógio confiável (NTP). Ordenação estável.
7. **Verificável.** No boot e em cron, rotina `verificar_cadeia()` valida a hash-chain e alerta se detectar quebra.

## 2. O que registrar (cada "respiração")

| Categoria | Eventos |
|---|---|
| **Sessão/Auth** | login, logout, falha de login, 2FA, expiração, troca de senha, impersonation (se existir) |
| **Autorização** | acesso **negado** (tentativa sem permissão/escopo), step-up de aprovação |
| **Dados de negócio** | criar, editar (com **antes/depois**), inativar/cancelar, reabrir, nova versão |
| **Aprovação** | submeter, aprovar, reprovar, travar registro, reabertura formal |
| **Exportação** | export Excel/PDF/CSV (o quê, filtro, formato, volume) |
| **Administração** | criar/editar perfil, conceder/revogar permissão, criar/inativar usuário |
| **Dado pessoal (LGPD)** | leitura/consulta de dado pessoal sensível, atendimento a direito de titular, compartilhamento |
| **Segurança/Sistema** | mudança de config sensível, detecção de quebra de cadeia, evento do runtime |

> Para **leitura** de dado pessoal sensível: registrar acesso (quem viu o quê e quando) — exigível pela LGPD/accountability. *(Detalhe a calibrar com a pesquisa LGPD em andamento — evitar volume inútil, focar em dado sensível e ações críticas.)*

## 3. Anatomia de um registro de auditoria

```jsonc
{
  "id": "ULID",                       // ordenável no tempo
  "ts": "2026-07-21T22:40:11.482Z",   // UTC
  "tz": "America/Sao_Paulo",
  "ator": {
    "user_id": "clerk_...",           // quem
    "perfil": "assessor-especial",
    "nome": "…",
    "ip": "…",
    "user_agent": "…",
    "sessao_id": "…"
  },
  "acao": "editar",                   // o quê
  "modulo": "contratos",
  "recurso": "registro",
  "recurso_id": "…",
  "resultado": "sucesso",             // sucesso | negado | erro
  "motivo": null,                     // p/ negado: qual regra bloqueou
  "antes": { /* snapshot/estado anterior (campos relevantes) */ },
  "depois": { /* estado novo */ },
  "diff": [ /* campos alterados */ ],
  "justificativa": "…",               // quando exigida (edição pós-reabertura)
  "correlation_id": "…",              // amarra eventos de uma mesma operação
  "hash_anterior": "sha256:…",        // encadeamento
  "hash": "sha256:…"                  // hash deste registro (inclui hash_anterior)
}
```

- **hash** = `sha256(canonical(registro_sem_hash) + hash_anterior)`. Torna a sequência inviolável.
- **antes/depois**: para dados pessoais, aplicar redação/minimização conforme LGPD (não duplicar dado sensível à toa; registrar que mudou, e o valor só quando necessário e protegido).

## 4. Garantias técnicas (backend Go)

1. **Interceptação central:** toda mutação passa por uma camada (`audit.Registrar(evento)`) — não confiar em cada handler lembrar de logar. Onde possível, hook no repositório/serviço.
2. **Transacional via Outbox — ZERO auditoria perdida (regra crítica):** como a trilha vive em **instância separada**, não há transação ACID única atravessando dois bancos. Então: o evento de auditoria é gravado numa tabela **`outbox` na MESMA transação** da mutação de negócio (atômico — se a operação commita, o evento **existe**; se falha, some junto). Um **worker confiável** (`audit-shipper`) lê o outbox e **entrega** o evento ao **Postgres de auditoria dedicado** (onde o hash-chain é calculado) e **indexa no Elasticsearch**. Worker caiu → o evento **permanece no outbox** e reprocessa (entrega ao menos uma vez, idempotente). **Nunca** aprovar/gravar e confiar que "a auditoria grava depois" sem o outbox.
3. **Permissão de banco:** a aplicação recebe **apenas INSERT** na trilha (no Postgres de auditoria); `UPDATE`/`DELETE` revogados no nível do SGBD. O `audit-shipper` é o único escritor da trilha, com credencial própria.
4. **Sem log de segredo/PII crua:** nunca gravar senha, token, chave. PII sensível conforme política de minimização.
5. **Correlação:** um `correlation_id` por requisição amarra todos os eventos derivados.
6. **Retenção:** definir prazo de guarda alinhado a controle interno/TCU e LGPD (a pesquisa vai fixar prazos). Nada de expurgo silencioso — expurgo, quando legalmente permitido, é ele próprio auditado.

## 5. Integridade e verificação contínua

- **Escritor único & ordenação:** o `audit-shipper` é o **único** processo que grava no Postgres de auditoria, garantindo a ordem estável necessária ao hash-chain (cada registro encadeia o hash do anterior).
- **`verificar_cadeia()`**: recalcula a hash-chain do período **no Postgres de auditoria** (a prova); qualquer divergência → alerta P0 ao Squad de Auditoria + Security Master. O Elasticsearch nunca é usado para verificar integridade — só para buscar.
- **Selo periódico:** a cada janela (ex.: diária), gravar um "checkpoint" com o hash do último registro em local independente (ex.: armazenamento externo append-only), permitindo provar que a trilha não foi reescrita retroativamente.
- **Backup imutável:** a trilha entra no backup cifrado (padrão Locus: `age` + destino remoto), com rotação e restauração testada.

## 6. Consulta e exportação da trilha

- Apenas perfis autorizados (auditor/diretoria) **consultam** a trilha — e essa consulta também é auditada.
- Exportação da trilha para fiscalização (TCU/controle interno) é uma função explícita, auditada, com escopo por período/módulo.
- **Histórico de alterações de um registro** (requisito): a tela de um registro mostra todas as versões e quem mudou o quê — alimentada pela trilha + versionamento (ver [`07-SOFT-DELETE-VERSIONAMENTO.md`](07-SOFT-DELETE-VERSIONAMENTO.md)).

## 7. Anti-padrões proibidos

- ❌ Botão/rotina que "limpa logs".
- ❌ Log só em arquivo texto rotacionável e sobrescrevível.
- ❌ Auditoria gravada *depois* e *fora* da transação **sem outbox** (pode se perder).
- ❌ **Elasticsearch como fonte da prova** — ele é mutável; serve só para busca.
- ❌ Confiar no front para decidir o que auditar.
- ❌ Admin com poder de editar a trilha.

## 8. Responsáveis

Dono técnico: **Squad de Auditoria & Compliance** + **Squad LGPD** (dado pessoal na trilha) + **Security Master** (integridade). Mudança neste protocolo → entrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md) e revisão dupla.
