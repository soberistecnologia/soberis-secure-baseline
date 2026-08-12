---
protocolo: RBAC-01
titulo: Perfis de Usuário e Controle de Acesso (Fase 1)
status: canônico
fase: 1
fonte_requisito: Docs_dev/Perfil de Usuário.docx
atualizado: 2026-07-24
---

# RBAC-01 — Perfis de Usuário e Controle de Acesso

> **Foco da Fase 1.** Traduz o requisito do cliente (`Docs_dev/Perfil de Usuário.docx`) no modelo de autorização do projeto. Herda o RBAC ultragranular do Locus e corrige o IDOR do One Nexus (autorização conhece **escopo/ownership**, não só a string da permissão).

## 1. Modelo de permissão

Formato: **`modulo:recurso:acao`** — ex.: `contratos:registro:criar`, `pagamentos:*:aprovar`.

- **Ações canônicas:** `consultar`, `criar`, `editar`, `inativar`, `aprovar`, `exportar`.
- **Wildcards:** `*` (tudo), `modulo:*` (todo o módulo). Nunca `*:*:aprovar` sem SoD.
- **⭐ Namespace protegido (`admin`) — curinga de módulo NÃO o alcança.** Um *grant* com módulo `*`
  (ex.: `*:*:editar`) **não** cobre requisição no módulo `admin`. Só um *grant* que **nomeia**
  `admin` autoriza ali (`admin:perfis:*`, `admin:usuarios:editar`, …). A regra é **assimétrica de
  propósito**: o **`deny` mantém alcance total** — `deny *:*:*` **continua negando** `admin:…`
  (enfraquecer o deny seria fail-open). Ver [ADR-0006](../../docs/arquitetura/adr/ADR-0006-namespace-admin-protegido.md).
  Acrescentar módulo à lista de namespaces protegidos é ato **humano-obrigatório** (PRIN-00 §4).
- **Origem das permissões de um usuário:** `perfil (role) → permissões` **+** `overrides por usuário` (grant/deny).
- **Regra de resolução — DENY VENCE.** Qualquer `deny` (no perfil ou no override) supera qualquer `grant`. Ausência = negado (fail-closed).
- **Ownership/escopo obrigatório:** toda operação — inclusive **leitura** — valida se o registro pertence ao escopo permitido do usuário (unidade/centro de custo/módulo). Não basta ter a string da permissão. *(Correção direta do IDOR SEC-101 do One Nexus.)*

## 2. Os 6 perfis (do requisito)

| Perfil | Qtd | consultar | criar/editar | aprovar | exportar | admin acesso | Observação |
|---|---|:---:|:---:|:---:|:---:|:---:|---|
| **Diretor-Presidente** | 1 | ✅ tudo | ❌ | ✅ **final, geral** | ✅ | ❌ | Consulta + aprovação final geral. **Não edita.** |
| **Diretor Financeiro** | 1 | ✅ tudo | ❌ | ✅ **final financeiro** | ✅ | ❌ | Aprova só **operações financeiras**. **Não edita.** |
| **Assessor I** | 1 | ✅ tudo | ✅ **todos os módulos** | ❌ | ✅ | ❌ | Operador pleno. Não aprova. **Exporta** (§6.1). |
| **Assessor Especial** | 1 | ✅ tudo | ✅ **todos os módulos** | ❌ | ✅ | ✅ **perfis/usuários** | Operador pleno **+ admin de acesso** (2FA + travas SoD, §4.1). **Exporta** (§6.1). |
| **Assessor (Passagens/Compra Facilitada)** | 1 | ✅ tudo | ✅ **só** `passagens` e `compra-facilitada` | ❌ | ❌ | ❌ | Nos demais módulos: **só consulta**. |
| **Demais Usuários** | N | ✅ (só liberados) | ❌ | ❌ | ❌ | ❌ | **Somente leitura**, escopo por usuário. |

> **Decisões do cliente (fonte: `Docs_dev/Perfil de Usuário.docx`, confirmadas 2026-07-22):**
> - **Exportação (Excel/PDF/CSV): os dois Diretores e os dois Assessores** (I e Especial). O
>   Assessor de Passagens/Compra Facilitada e os Demais Usuários **não exportam**. ⚠️ **Revisto em
>   2026-07-28** — este item dizia *"SÓ os Diretores"* até então. O fundamento da mudança está em
>   **§6.1**, e é ele que precisa ser lido antes de qualquer tentativa de "corrigir" isto de volta.
> - **Admin de acesso** (criar perfis sem programar + gerir usuários): **embutido no Assessor Especial** (não há 7º perfil).
> - **Diretor Financeiro** aprova **só o financeiro** (`empenhos`, `pagamentos`, `ddf`, `passagens`, `compras`); **Presidente** = aprovação final geral.
> - **Demais Usuários**: leitura só dos módulos/relatórios **disponibilizados a cada um** ("conforme necessidade") — não é "tudo em leitura".

> **Segregação de funções (SoD):** quem **edita** (Assessores) nunca **aprova**; quem **aprova** (Diretores) nunca **edita**. Invariante do sistema — reflete o controle interno do órgão.

### 4.1 Travas do admin de acesso (Assessor Especial) — anti-privesc

Como o Assessor Especial **edita dados E administra acesso**, aplicam-se travas obrigatórias para não reabrir o *privilege escalation* do One Nexus (SEC-010/021):
1. **Invariante SoD em código (definição canônica — alcance GLOBAL):** um conjunto **efetivo**
   (perfil ∪ overrides) **não** pode reunir o eixo de **edição** e o eixo de **aprovação** em
   **nenhum** módulo de negócio — nem "no mesmo módulo": **editar em qualquer módulo + aprovar em
   qualquer módulo já colapsa**. Gravação que viole é **recusada** (`rbac.ValidarSoD`). Logo o
   admin não consegue se tornar aprovador. Os eixos, literalmente:
   - **edição** = `criar`, `editar`, `inativar`; **aprovação** = `aprovar`;
   - `consultar` e `exportar` **não** pertencem a nenhum eixo;
   - ação curinga (`modulo:recurso:*`) conta como **os dois** eixos;
   - o módulo **`admin`** fica **fora** do SoD de negócio (administrar acesso não é editar nem
     aprovar registro) — por isso as travas 2, 3 e 4 desta seção existem.

   **Por que global e não por módulo** (decisão D5, `security-master` + `sec-isolamento-acesso`,
   2026-07-24): os módulos do projeto — contratos → compras → empenhos → pagamentos → prestação de
   contas — são **etapas da mesma cadeia de despesa**. Quem edita o empenho e aprova o pagamento
   fechou o ciclo sozinho: um SoD por módulo **autorizaria exatamente essa fraude**. Global é, além
   de mais restritivo (fail-closed, PRIN-00 §1), o comportamento **correto** para este domínio.
   Nenhum dos 6 perfis da semente é afetado. Se algum dia houver necessidade legítima de "edita
   Passagens, aprova Diárias", é **novo ADR + decisão do dono** — nunca ajuste de código.
2. **Anti-auto-escalação:** o admin **não** altera as próprias permissões críticas nem se concede `aprovar` — exige **segundo ator** (dupla custódia).
3. **Auditoria + 2FA:** toda mudança de perfil/permissão/usuário é **evento imutável**; **2FA obrigatório** no Assessor Especial.
4. **Anti-lockout atômico:** operação serializada impede zerar todos os administradores.

### Matriz de permissões (semente inicial)

> **Duas garantias, não uma** (ADR-0006, aprovado com veto do `security-master` em 2026-07-24):
> o curinga de módulo **não alcança `admin`** por **regra de resolução** (garantia estrutural, vale
> inclusive para perfil criado depois na tela, sem programar) **e** os cinco perfis não-admin
> recebem `deny admin:*:*` **explícito** na semente (garantia declarada, visível na tela de perfis e
> na trilha). Cinto e suspensório: nenhuma das duas sozinha basta — a regra sozinha é invisível ao
> operador; o deny sozinho é fail-open por omissão no perfil que alguém criar amanhã.
>
> ⚠️ **Consequência operacional do deny:** deny vence. Conceder `admin:perfis:*` a um destes cinco
> perfis **não terá efeito** enquanto o `deny admin:*:*` estiver lá. Promover um perfil a
> administrador é, de propósito, **dois atos deliberados e auditados** (remover o deny + conceder o
> grant). A tela precisa dizer isso ao operador em vez de deixá-lo achar que a concessão falhou.

```
# Diretor-Presidente
*:*:consultar
*:*:aprovar            # aprovação final GERAL
*:*:exportar
deny admin:*:*         # não administra acesso (coluna "admin acesso" = ❌)

# Diretor Financeiro
*:*:consultar
empenhos:*:aprovar     # aprova SÓ os módulos financeiros
pagamentos:*:aprovar
ddf:*:aprovar
passagens:*:aprovar
compras:*:aprovar
*:*:exportar
deny admin:*:*

# Assessor I
*:*:consultar
*:*:criar
*:*:editar
*:*:inativar
*:*:exportar           # ⭐ 2026-07-28 — ver §6.1 (o laço exportar→editar→reimportar)
deny admin:*:*         # ⭐ fecha a cadeia: sem isto + ADR-0006, `*:*:editar` cobriria
                       #    `admin:usuarios:editar` e ele se atribuiria perfil de Diretor
# (SEM :aprovar)

# Assessor Especial  (= Assessor I + admin de acesso; travas SoD §4.1 + 2FA)
*:*:consultar
*:*:criar
*:*:editar
*:*:inativar
*:*:exportar           # ⭐ 2026-07-28 — ver §6.1
admin:perfis:*         # ⚠️ SEMPRE por recurso — `admin:*:*` é PROIBIDO como grant (ADR-0006 §4):
admin:usuarios:*       #    capacidade administrativa nova nasce SEM titular, e alguém precisa
                       #    concedê-la por ato auditado
# (SEM :aprovar, SEM deny admin — este é o único administrador)

# Assessor (Passagens/Compra Facilitada)
*:*:consultar
passagens:*:criar
passagens:*:editar
compra-facilitada:*:criar
compra-facilitada:*:editar
deny admin:*:*
# demais módulos: só consultar

# Demais Usuários
<modulos-liberados>:*:consultar   # escopo por usuário ("conforme necessidade")
deny admin:*:*
```

**Administrador gênese (anti-lockout na semente — RBAC-01 §4.1.4).** O Assessor Especial é o
**único** perfil administrador. A semente **não pode** deixar o sistema com **zero administrador
ativo**: ou vincula um administrador gênese **nomeado** (identidade Clerk confirmada, ato auditado
com ator real), ou o boot **falha fechado** com diagnóstico explícito. **Proibido** resolver isso
com super-usuário chumbado no código ou *break-glass* implícito — seria reabrir por outra porta o
super-usuário que o ADR-0006 §4 acaba de fechar. Se houver *break-glass*, é **procedimento
documentado de dupla custódia**, auditado, nunca um caminho de código.

## 3. Módulos (controle por módulo)

Permissões separadas por módulo, para ampliação futura sem reprogramar:
`contratos` · `compras` · `prestacao-contas` · `passagens` · `ddf` · `empenhos` · `pagamentos` · `compra-facilitada` · *(extensível)*.

## 4. RBAC dinâmico ("criar perfis sem programação")

Requisito explícito: **criar novos perfis sem código**. Portanto:
- Perfis, permissões e overrides ficam em **dados** (tabelas), não em código.
- Tela de administração (perfil com permissão `admin:perfis:*`) permite criar/editar perfil, marcar permissões e overrides.
- Catálogo de permissões é gerado a partir dos módulos/recursos registrados.
- Toda mudança de perfil/permissão **gera evento de auditoria** (ver [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md)) e é operação **humano-obrigatório** (nunca automática por agente).

## 5. Autenticação (Clerk) + 2FA para aprovadores

- Identidade via **Clerk** (ver [`04-AUTH-CLERK.md`](04-AUTH-CLERK.md)). O papel/perfil do projeto vive no **nosso** RBAC, não no Clerk (Clerk autentica, a aplicação autoriza).
- **2FA obrigatório para perfis aprovadores** (Diretor-Presidente e Diretor Financeiro) **no login** — requisito do cliente ("autenticação em dois fatores para perfis de aprovação").
- ⚠️ **Assinatura de aprovação (correção 2026-07-24):** cada aprovação exige **assinatura de step-up
  própria do projeto** — **não** o 2FA do Clerk. **WebAuthn/FIDO2 assina o `payload_hash`** do ato
  (prova o valor exato, chave privada nunca sai do autenticador); TOTP é **tier fraco**, só abaixo
  do limite dual-control e só para quem não tem WebAuthn. Ver [`06-FLUXO-APROVACAO.md`](06-FLUXO-APROVACAO.md) §2.
- **SoD por capacidade (fail-closed) — alcance GLOBAL:** quem tem `aprovar` em **qualquer** módulo
  de negócio **não pode** ter `editar`/`criar`/`inativar` em **nenhum** módulo de negócio — nem via
  override. Definição canônica e justificativa em **§4.1.1** (é lá, e só lá, que esta invariante é
  definida). Validado a cada mudança de perfil/override, sobre o conjunto **efetivo**, dentro da
  transação. *(Correção D5, 2026-07-24: este marcador dizia "no mesmo módulo" e contradizia a §4.1.1
  — o texto estava errado, o código sempre foi global. Validação no **boot** ainda é **pendente de
  verificação**; não presuma implementada.)*
- **⚠️ O eixo temporal NÃO está coberto** (achado P3-06/P2, aberto): o SoD é avaliado sobre o
  conjunto **no instante** da gravação. Editar hoje como Assessor, ser promovido a Diretor e aprovar
  amanhã o próprio registro **não é barrado** por `ValidarSoD`. Quem fecha isso é o SoD de
  **linhagem** do [`06-FLUXO-APROVACAO.md`](06-FLUXO-APROVACAO.md) §3 — desenhado, **não
  implementado**. Confirmar o alcance global **não** significa que o SoD está completo.

## 6. Controles derivados (do requisito)

- **Exportação controlada:** permissão `modulo:recurso:exportar` define quem exporta Excel/PDF/CSV. Toda exportação é auditada (quem, o quê, formato, quando). **Quem exporta: §6.1.**

### 6.1 Quem exporta, e por quê — o laço exportar → editar → reimportar

> **Decisão do dono, 2026-07-28.** Substitui a regra anterior (*"exportação: SÓ os Diretores"*).
> **Isto aqui é o fundamento, e ele existe para que ninguém "corrija" a decisão de volta** achando
> que ampliar a exportação foi descuido. Não foi.

**Quem exporta:** Diretor-Presidente, Diretor Financeiro, **Assessor I** e **Assessor Especial**.
**Quem não exporta:** Assessor (Passagens/Compra Facilitada) e Demais Usuários.

#### O que mudou no raciocínio

A regra antiga tratava exportação como **risco de vazamento sem contrapartida**: um arquivo sai do
sistema, e a partir daí o sistema não sabe mais nada sobre ele. Sob essa premissa, restringir era o
único controle disponível, e restringir aos Diretores era o mais apertado que se podia fazer.

A premissa deixou de valer, porque **exportar virou a ponta de um laço que se fecha na importação**:

1. a **exportação** é ato de negócio registrado na trilha — quem baixou, o quê, quando, com que
   filtro, quantas linhas, e o digesto do arquivo entregue;
2. o servidor edita a planilha na máquina dele — que é o que ele **já faz hoje**, porque o órgão
   trabalha em planilha;
3. ao **reimportar**, o sistema lê o que mudou e registra a diferença (ADR-0012: transcrição fiel,
   cálculo soberano, **divergência declarada**).

Guardando a réplica do que saiu, a auditoria consegue afirmar, com evidência: *"a planilha X foi
baixada por fulano em tal data, e voltaram alteradas estas informações."* Nas palavras do dono: **"o
cara não teria como mentir"**.

#### Por que isso justifica ampliar, e não apenas tolerar

Os dois Assessores **já enxergam tudo** (`*:*:consultar`) e **já editam tudo** (`*:*:criar|editar`).
Negar-lhes `exportar` não protegia o dado: protegia apenas o *formato*. Quem enxerga a tela copia a
tela. O que a restrição de fato produzia era **contorno informal** — captura de tela, cópia manual,
planilha paralela —, e todo contorno informal sai **de dentro do rastro**: nenhum deles passa pela
trilha, e nenhum deles volta pela importação. A restrição empurrava o trabalho real para o único
caminho que a auditoria não enxerga.

Com o laço, o caminho rastreável passa a ser também o caminho **mais cômodo** — e é assim que
controle funciona em órgão público: quando a via auditada é a via fácil.

E o documento do cliente concorda: `Docs_dev/Perfil de Usuário.docx` diz, dos dois Assessores, com
frase idêntica, *"Pode **gerar** e visualizar todos os relatórios"* — sendo **"gerar"** o verbo que
os separa de quem apenas *"visualiza os relatórios disponibilizados"*.

#### Ressalva escrita: os Diretores

O documento descreve os Diretores com *"**visualiza** todas as informações, relatórios e
dashboards"* — **não** com "gerar". Pela letra estrita do critério acima, eles não exportariam.
Foram **mantidos como exportadores por decisão expressa do dono**, e a divergência fica registrada
aqui em vez de ser alisada: quem revisar este protocolo contra o documento de origem vai encontrar a
diferença, e ela é **decisão**, não erro de transcrição.

#### O que este fundamento EXIGE, e que ainda não existe

⚠️ A ampliação se apoia no laço. **O laço ainda não está fechado**, e enquanto não estiver, o que
existe é a metade de saída (a trilha da exportação) sem a metade de volta. Isto não é motivo para
reverter a decisão — é a lista do que falta, e ela tem dono:

1. **Réplica do que saiu.** Sem guardar o conteúdo exportado (ou prova suficiente dele), não há
   contra o que comparar na volta, e a frase *"voltaram alteradas estas informações"* não se
   sustenta.
2. **Reconhecimento por CONTEÚDO, não só por carimbo.** Quem quiser esconder a origem **apaga a
   marca** e reenvia como arquivo novo. O vínculo não pode depender só dela: o sistema precisa
   reconhecer *"é a exportação de terça com sete células mexidas"* mesmo sem marca — e aí **tirar o
   carimbo vira sinal**, em vez de ser evasão bem-sucedida.
3. **Retenção e base legal da réplica.** Guardar réplica é **guardar dado pessoal**: diárias e
   passagens têm nome de servidor. Prazo, base legal e eliminação precisam estar escritos antes de a
   primeira réplica existir — `lgpd-compliance` decide, e a réplica **não** herda o regime da trilha.

#### Nota de escopo — isto é padrão de entrega, não lei

O cliente **personaliza** perfis pelo painel (RBAC dinâmico, §1). Esta matriz é o **estado inicial
entregue**, e mudá-la tem de ser **configuração, nunca deploy**.

⚠️ **Ponto de atenção medido (2026-07-28):** `Semente.Aplicar` chama
`DefinirPermissoesDoPerfil` para todo perfil **ativo**, isto é, **reescreve** o conjunto de
permissões para o da matriz. Só o perfil **inativado** é preservado. Portanto **reexecutar
`cmd/lunar-seed` depois de o cliente personalizar desfaz a personalização, em silêncio, para os
perfis ativos**. A semente é binário explícito (não roda no boot), o que limita o dano, mas o
comportamento precisa ser decidido antes de haver cliente personalizando — hoje ele contraria a
regra *"mudar é configuração, nunca deploy"*. **Encaminhado a `sec-isolamento-acesso` +
`security-master`.**
- **Dashboards por perfil:** cada perfil vê só os indicadores pertinentes à sua função (autorização também na camada de visualização).
- **Pesquisa:** por protocolo, contrato, fornecedor, período, valor, centro de custo — respeitando escopo de leitura do usuário.
- **Notificações:** pendência de aprovação, documento vencendo, saldo insuficiente, prazo próximo (eventos que também alimentam a auditoria).

## 7. Regras de implementação (para o backend Go)

1. Middleware de autorização roda **depois** do de autenticação e **antes** do handler: `RequirePermission("modulo:recurso:acao")` + `RequireScope(...)`.
2. Nenhum handler lê identificador de recurso "cru" do parâmetro sem checar escopo (anti-IDOR).
3. Resolução de permissão é centralizada (um único ponto de verdade), testada com suíte adversarial de bypass.
4. `deny` e ausência sempre bloqueiam. Sem "fail-open em loading" no front (erro do One Nexus).
5. **Toda autorização NEGADA é auditada** — evento próprio (ator, permissão exigida, rota, origem,
   horário). *(Correção 2026-07-24, achado P2-07: este item dizia "pode ser auditada". "Pode" não é
   controle. Sem trilha de negação, quem sonda o RBAC não deixa rastro e o `runtime-detection` não
   tem o que detectar — contra PRIN-00 §1.4 e o vetor "acesso indevido" do §2.)* A resposta ao
   cliente continua **genérica** (anti-enumeração: 404 para fora de escopo, 403 para sem
   permissão); o **detalhe** vive só na trilha.
6. **`admin` só se alcança por grant literal** — o curinga de módulo não o cobre (§1, ADR-0006).
   Como consequência, o **override por usuário passa a ser o único caminho restante** até o módulo
   `admin` (um override **nomeia** `admin`, então funciona por desenho). Logo o **anti-auto-escalação
   (§4.1.2)** deixa de ser desejável e passa a ser **o controle** daquele caminho: alterar override
   é ato auditado, o ator nunca altera os próprios, e conceder capacidade administrativa exige
   segundo ator.
7. **Revogar perfil revoga o acesso inteiro.** Trocar/rebaixar o perfil de um usuário **também
   reavalia e revoga os overrides pessoais incompatíveis** com o novo perfil. *(Achado P3-03,
   reclassificado **P2** em 2026-07-24: como `usuario_perfil` tem PK em `user_id` e
   `usuario_overrides` é tabela independente, rebaixar alguém hoje **mantém** os overrides antigos —
   o usuário rebaixado continua com o privilégio pessoal, inclusive `admin:*`. Trocar o perfil é
   justamente o que o órgão faz quando alguém muda de função ou sai: é o principal controle de
   revogação, e hoje ele não revoga.)*

## 8. Pendências — RESOLVIDAS (2026-07-22, com o cliente)

- ✅ **"Demais Usuários"**: leitura **só dos módulos/relatórios liberados a cada usuário** ("conforme necessidade") — escopo individual, não "tudo".
- ✅ **Diretor Financeiro**: aprova **só os módulos financeiros** (`empenhos`, `pagamentos`, `ddf`, `passagens`, `compras`). Presidente = aprovação final geral.
- ✅ **Exportação** *(revisto em 2026-07-28 — antes dizia "somente os Diretores")*: os dois
  **Diretores** e os dois **Assessores** (I e Especial). O Assessor de Passagens/Compra Facilitada e
  os Demais Usuários **não** exportam. Fundamento em **§6.1**.
- ✅ **Admin técnico**: **não há 7º perfil** — a administração de acesso (perfis/usuários) fica no **Assessor Especial**, com as travas anti-privesc da §4.1 (invariante SoD, anti-auto-escalação, 2FA, anti-lockout).
