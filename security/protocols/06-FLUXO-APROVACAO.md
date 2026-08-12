---
protocolo: APR-06
titulo: Fluxo de Aprovação, Assinatura de Aprovação (step-up) e Trava de Registro
status: canônico
fase: 1
fonte_requisito: Docs_dev/Perfil de Usuário.docx
handoff: security/handoffs/APROVACAO-STEPUP-brief.md (revisado pelos 4 squads)
atualizado: 2026-07-24
---

# APR-06 — Fluxo de Aprovação e Assinatura de Aprovação

> Requisito do cliente: *"após a aprovação do Diretor Financeiro ou Diretor-Presidente, o registro deve ficar bloqueado para edição. Caso seja necessária alteração, deve haver reabertura formal ou criação de nova versão."* Reflete a **segregação de funções (SoD)**: quem edita não aprova; quem aprova não edita.

## 1. Estados de um registro

```
rascunho → submetido → em_aprovacao ─┬→ aprovado → [TRAVADO]
                          (congelado)├→ reprovado
                                     └→ (TTL expira) → submetido
travado → reaberto (formal, assinado) → NOVA versão → submetido → ...
```

- **rascunho/submetido:** editável por quem tem permissão de edição no módulo.
- **`em_aprovacao`:** ⭐ **congelado** — o registro **rejeita qualquer edição** enquanto há desafio de assinatura em aberto. TTL curto sem assinatura → **auto-libera** para `submetido` (não trava para sempre).
- **aprovado → travado:** **imutável**. Nenhuma edição, por ninguém.
- **reabertura formal:** exige perfil aprovador + justificativa **dentro do payload assinado**; gera **nova versão** (a anterior e sua assinatura são preservadas e imutáveis). Ver [`07-SOFT-DELETE-VERSIONAMENTO.md`](07-SOFT-DELETE-VERSIONAMENTO.md).

## 2. Assinatura de aprovação — mecanismo próprio, **desacoplado do Clerk**

> ⚠️ **Correção 2026-07-24 (substitui a redação anterior):** o step-up **não** é o 2FA do Clerk.
> O Clerk é **só login/identidade**; a **assinatura do ato de aprovação é mecanismo nosso**, em
> `internal/security`. A identidade do ato continua sendo a sessão Clerk (`clerk_user_id`), ligada
> ao mesmo `user_id` que o RBAC reconhece como aprovador. Ver [`04-AUTH-CLERK.md`](04-AUTH-CLERK.md) §2.

### 2.1 Fator por USUÁRIO (não por request)

| Fator | Status | Uso |
|---|---|---|
| **WebAuthn/FIDO2** | **obrigatório** para toda aprovação financeira (`empenhos`, `pagamentos`, `ddf`, `passagens`, `compras`) e no/acima do limite da Regra dos Dois | Assinatura **assimétrica** sobre o `payload_hash` → prova o **valor exato**; chave privada nunca sai do autenticador |
| **TOTP** | **tier fraco** — prova de presença, **NÃO** assinatura nem não-repúdio | Só **abaixo** do limite dual-control e só para quem **não** tem WebAuthn |
| Hardware FIPS/YubiKey | fase 2 | — |

- **O HMAC do servidor sobre o payload é selo anti-adulteração em repouso** — **não** é "assinatura do aprovador" (a chave é do servidor: forjável se vazar). Versionar `hmac_key_id`; custódia/rotação por [`05-GESTAO-SEGREDOS.md`](05-GESTAO-SEGREDOS.md).
- **Anti-downgrade:** quem tem WebAuthn enrolado **só** assina com WebAuthn — TOTP nunca é oferecido como fallback no ato. Rebaixar fator = **cerimônia de re-enroll 2-admin auditada**, nunca self-service.

### 2.2 Binding ao conteúdo exato (anti-TOCTOU de valor)

A assinatura cobre `payload_hash = sha256(canonical(snapshot da versão))` sobre **todos** os campos financeiros (itens, favorecido, rateio, totais) **+ `version`** — não só "o valor".

- **Congelamento na emissão do desafio** (`em_aprovacao`).
- **CAS transacional no commit:** `UPDATE … WHERE status='em_aprovacao' AND version=$v AND payload_hash=$h`. `rows_affected=0` → **aborta fail-closed**, audita anomalia **P0**, exige reaprovar.
- O servidor **re-deriva** o payload do estado autoritativo sob lock — **nunca** confia no payload vindo do cliente.
- **WYSIWYS:** o valor exibido ao aprovador deriva do **mesmo** payload assinado.

### 2.3 Nonce single-use e anti-double-spend

- Nonce **server-side**, ligado a `(record_id, version, payload_hash, aprovador, decisao)`, **consumido atomicamente na mesma transação do commit** (`UNIQUE` / `FOR UPDATE`). Rollback → **não queima** (permite retry); commit → **nunca reusa**.
- Fonte de verdade do consumo = **Postgres** (Redis só cache best-effort).
- Idempotency-key = nonce; retry pós-commit = **sucesso idempotente**, sem duplicar.
- **Aprovação em massa/batch é proibida:** invariante **1 nonce : 1 resposta de fator : 1 registro**. Rajada → alerta de velocidade.

### 2.4 Atomicidade da PROVA

❌ "hash-chain na mesma transação" é **impossível** — a auditoria é um Postgres separado (sem ACID cruzado).

✅ O que é atômico numa transação no **MASTER**: **trava do registro + linha de aprovação** (assertion/HMAC + `payload_hash` + `version` + `nonce` + `ts`) **+ linha no `outbox`**.
**Fail-closed: sem evento assinado gravado, o registro NÃO entra em TRAVADO.**

- A evidência de assinatura vive **também na linha do MASTER** (a prova existe antes do shipping).
- **Hash-chain é assíncrono** no Postgres de auditoria (`audit-shipper`, idempotente, at-least-once) e **nunca gateia a resposta** da aprovação (ARQUITETURA R2).
- Confirmar "aprovado" ao usuário **só após commit durável no MASTER** — nunca via projeção BI/dash (R3).
- **Job de reconciliação MASTER↔auditoria** (boot + cron): todo registro TRAVADO tem evento encadeado com `content_hash` batendo. Ausência/divergência = **P0**. *(O hash-chain detecta adulteração; **não** detecta registro faltando — daí a reconciliação.)* Alerta de lag do shipper.

### 2.5 Freshness, burn e limites

- **TTL curto (2–5 min)** cobrindo a **janela inteira** (render → assinatura → commit). Commit fora do TTL → **void fail-closed**.
- **TOTP:** queimar o timestep usado (rejeitar ≤ último), drift ±1, segredo ≥160 bits, single-use.
- **Rate-limit e lockout por ator (`user_id`) e por credencial** — não só por IP (XFF só de proxy confiável por CIDR, [`11-HARDENING-APLICACAO.md`](11-HARDENING-APLICACAO.md) §3). O lockout **não pode** ser DoS gatilhável por terceiro nem forçar downgrade de fator.
- **NTP** no servidor; **nunca** usar o relógio do cliente.
- Respostas **constant-time/genéricas** (anti-enumeração). Negação por SoD é **auditada** ([`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md) §2) mas devolve erro genérico.
- Front declarando `stepUpVerified:true` sem verificação server-side → **rejeitado** (teste obrigatório).

## 3. SoD reforçada (capacidade + linhagem)

- **Nível-capacidade (alcance GLOBAL):** usuário com `aprovar` em **qualquer** módulo de negócio **não pode** ter `editar`/`criar`/`inativar` em **nenhum** módulo de negócio — **nem via override**. Definição canônica em [`01-RBAC-PERFIS.md`](01-RBAC-PERFIS.md) **§4.1.1** (fonte única; não redefinir aqui). Validado a cada mudança de perfil/override sobre o conjunto **efetivo**, fail-closed. *(Correção D5, 2026-07-24: dizia "no mesmo módulo" — os módulos do Lunar são etapas da MESMA cadeia de despesa, e por módulo autorizaria editar o empenho e aprovar o pagamento. Validação **no boot** é pendente de verificação, não presuma implementada.)*
- O check varre **toda a linhagem de edição** (todas as versões/reaberturas), no servidor, **deny-vence**, na **emissão do desafio** *e* no **commit**.
- **Reprovar / devolver / reabrir também são step-up assinados** — reabrir destrava dinheiro, é altíssimo risco — com **justificativa dentro do payload assinado**.
- A assinatura da versão anterior é **imutável e preservada**; a reabertura cria **nova versão sem assinatura** (estado fresco).

## 4. Enrollment blindado

- Enroll / adição / troca de fator exige step-up com fator **já existente** **OU** cerimônia **2-admin auditada**. Enroll inicial de aprovador = alta garantia (presencial / 2-admin).
- Vincular auditavelmente `clerk_user_id ↔ credential_id`; **negar** credencial não vinculada à sessão (anti-swap).
- **Proibido** adicionar fator mais fraco ao lado de um mais forte.
- Re-enroll: transação **serializada** (anti-race), invalida credencial antiga, encadeado, autorização 2-admin **nonce-bound não-replayável**; aprovações seguintes ficam **sinalizadas** na trilha.

## 5. WebAuthn — hardening obrigatório

- `RPID` = domínio registrável **exato**; `RPOrigins` = allowlist **exata** (sem wildcard/subdomínio-pai); rejeitar Origin ausente e `"null"`.
- Algoritmos COSE em allowlist (**ES256/EdDSA**); rejeitar `none`.
- `UserVerification="required"` no enroll e no login; **validar o bit UV no `authenticatorData`** (não confiar no request).
- `SessionData` (challenge + credenciais + UV) **server-side**; verificar `type=="webauthn.get"` e `challenge` == o emitido.
- **`signCount` monotônico** — regressão = autenticador clonado = **deny + P0**.

## 6. Payload canônico do evento assinado (não-repúdio TCU)

`clerk_user_id` · `perfil` (no momento) · `credential_id`|`totp_key_id` · `decisao ∈ {aprovado,reprovado,devolvido,reaberto}` + flag final/intermediário · `modulo,recurso,recurso_id,version,payload_hash` · `valor_total_centavos:int64` · `itens_hash` · `nonce,issued_at,expires_at,session_id,correlation_id` · método (`webauthn` → `authenticatorData + clientDataJSON + signature + credential_id + AAGUID + signCount`; `totp` → HMAC + `hmac_key_id`) · `ip,user_agent,ts(UTC) + tz America/Sao_Paulo` · `justificativa` (**obrigatória** em reprovar/devolver/reabrir, **dentro** do payload).

Depois: `hash = sha256(canonical(payload) + hash_anterior)` no banco de auditoria.

- **Segredo TOTP e backup codes NUNCA na trilha** ([`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md) §4.4).
- Custódia com AAD = `entidade:campo:id` (ex.: `totp_secret:{user_id}`) + **zeroize**.

### 6.1 Canonicalização e precisão financeira

- **Uma única** `canonical(payload) []byte` em `internal/security` (JCS/RFC 8785 ou TLV length-prefixed), UTF-8 NFC, chaves ordenadas, inteiros sem zero à esquerda. **Mesma função** para signatário e verificador, com **golden vectors**. **Proibida uma 2ª implementação.**
- `valor_centavos int64` — **nunca** float nem string formatada. Estorno é `decisao` explícita, **não** valor negativo furtivo.
- **`total == soma(itens)`**, ambos assinados; divergência = **bloqueio fail-closed** (nunca arredondar a diferença).

## 7. Motor de controles por faixa de valor (parametrizável)

> **Decisão do dono (2026-07):** o **limite/valor** é decisão de negócio do órgão (movimenta milhões; R$ 50 mil é rotina), a fechar em mesa própria com o cliente. Constrói-se o **MOTOR** agora; faixas e valores são **configuração**. **Nada de limite chumbado no código.**

| Faixa | Valor | Controle mínimo |
|---|---|---|
| **Rotina** | até `[A]` | 1 aprovador (WebAuthn é o piso) |
| **Relevante** | `[A]`–`[B]` | 1 aprovador + notificação ao controle interno |
| **Alto** | `[B]`–`[C]` | **2 aprovadores distintos** (Regra dos Dois) |
| **Crítico** | acima de `[C]` | 2 aprovadores + **janela de resfriamento** + **notificação out-of-band** ao Presidente/controle interno |

**Regra dos Dois:** ambos assinam o **MESMO** `payload_hash`/`version`; humanos **distintos**; nenhum na linhagem de edição; ambos com `:aprovar` + escopo; `user_id` ligado ao Clerk (rejeitar FIDO de outro principal). **Sequencial com congelamento:** A assina → `aguardando_2a_assinatura` congelado em V/H → B assina a MESMA V/H → commit. Edição/reabertura no meio **derruba as duas**. **Lock financeiro só no commit da 2ª** (a 1ª é aprovação parcial durável e auditada — R2).

**Controles transversais (todas as faixas):**
- **Anti-fracionamento (structuring)** — agregar por `favorecido + objeto/natureza + janela`; agregado que cruza faixa aplica o controle da **faixa agregada** + alerta. *(R$ 3 mi em 60× R$ 49 mil cai na faixa crítica.)* Sem isto, o limite é teatro.
- **Teto por período/orçamento (velocity)** por centro de custo / favorecido / exercício; ultrapassar = alerta ou **bloqueio fail-closed**.
- **Janela de resfriamento** (crítica): pré-aprovado por N horas antes de efetivar.
- **Notificação out-of-band** (alta/crítica) por canal separado.
- **Painel de risco:** aprovações, pendências de 2ª assinatura, alertas de fracionamento/velocity, itens em resfriamento.

**Parametrização:** faixas, valores, janelas, tetos e canais vivem em **política versionada e auditada** — **não** em código. **Alterar a política é ato de alto privilégio sob a própria Regra dos Dois**; toda mudança = **evento assinado + encadeado**.

**Limitação honesta:** barra pessoa/credencial **única** comprometida; **não** barra conluio de dois diretores — controle compensatório = trilha imutável + ambos nomeados e não-repudiáveis.

## 8. Fluxo completo

```
submeter
  → clicar aprovar
  → back valida: é aprovador? escopo/módulo? NÃO é editor de nenhuma versão? capacidade não-conflitante?
  → congela em `em_aprovacao` + gera desafio ligado a (id, version, payload_hash, decisão, nonce, TTL)
  → aprovador assina (WebAuthn obrigatório se financeiro/alto-valor; senão TOTP tier-fraco)
  → back verifica (assinatura + origin/UV/signCount/challenge  OU  TOTP burn) + CAS
  → transação MASTER: trava + linha de aprovação + outbox   [fail-closed]
  → resposta só após commit durável
  → hash-chain assíncrono (audit-shipper)
  → reconciliação MASTER↔auditoria
```

Fora do TTL / CAS falha / SoD / gap de prova → **nega + audita, fail-closed**.

## 9. Garantias técnicas

1. A trava é verificada **no backend** (nunca confiar no front): editar registro travado → negado + auditado.
2. Reabertura **nunca** apaga a versão aprovada — cria nova versão vinculada.
3. Notificações de pendência de aprovação alimentam o requisito de "Notificações".
4. **Prazo de guarda** dos eventos assinados (anos — TCU/CONARQ) definido pelo squad LGPD; retenção **sem expurgo silencioso** (AUD-02 §4.6, DEL-07 §3).

## 10. Gate de implementação

Nada entra em `internal/security` sem: **(a)** spec fechada (este protocolo + o [handoff](../handoffs/APROVACAO-STEPUP-brief.md)); **(b)** **suíte adversarial de 16 testes verde** (§8 do handoff); **(c)** **dupla validação** (Executor ≠ Validador — humano + agente de segurança, regras 6 e 9 da Constituição); **(d)** ok do dono.

Dono: Squad Auditoria & Compliance (`compliance-geral`) + Arquitetura + `security-master` (veto).
Ligações: [`01-RBAC-PERFIS.md`](01-RBAC-PERFIS.md), [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md), [`04-AUTH-CLERK.md`](04-AUTH-CLERK.md), [`05-GESTAO-SEGREDOS.md`](05-GESTAO-SEGREDOS.md), [`11-HARDENING-APLICACAO.md`](11-HARDENING-APLICACAO.md), [`12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`](12-DUPLA-VALIDACAO-E-INTEGRIDADE.md).
