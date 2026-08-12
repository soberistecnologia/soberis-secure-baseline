---
name: go-reference
description: >-
  Referência Go acionável para o Projeto Lunar (backend financeiro governamental). Use ao
  escrever/revisar Go: idiomático (Effective Go), layout cmd/internal/pkg, gestão de erro
  (wrap %w, errors.Is/As), context, concorrência segura, generics, testing table-driven +
  testify, módulos, ferramentas (go vet, staticcheck, gosec, govulncheck), DINHEIRO em
  precisão fixa (nunca float — int64 centavos ou shopspring/decimal), pgx/sqlc, e repositório
  atrás de interface (porta/adapter). Gatilhos: "Go idiomático", "layout de projeto",
  "erro em Go", "context", "goroutine/channel", "generics", "table test", "pgx", "sqlc",
  "decimal/dinheiro em Go", "repositório", "gosec/govulncheck".
---

# go-reference — Go acionável para o Lunar

Conhecimento de referência do backend Go do Lunar (gestão financeira **governamental**, single-tenant). Denso e prático. **Cite práticas reais; nunca invente API.** Contexto que muda tudo: dinheiro é público (zero erro de cálculo), banco **ainda não decidido** (tudo atrás de porta), segurança é Camada 0.

Regras do projeto que este documento faz cumprir: **dinheiro nunca em `float`**; **banco atrás de porta/adapter**; **fail-closed**; **erro tratado sempre**; **teste table-driven junto do código**.

---

## 1. Go idiomático (Effective Go / Google Go Style)

- **Nomes:** curtos no escopo pequeno (`i`, `r`, `ctx`); descritivos no escopo grande. Sem `stutter` (`http.HTTPServer`→`http.Server`). Getters sem `Get` (`u.Name()`, não `u.GetName()`). Erros: variáveis `errFoo`, tipos `FooError`.
- **Interfaces:** "accept interfaces, return structs". Interface pequena e definida **onde é consumida** (o service declara `ContratoRepo`, não o adapter). Interfaces de 1–3 métodos são a norma; `io.Reader`/`io.Writer` são o modelo.
- **Zero value útil:** projete tipos que funcionam no zero value (`sync.Mutex`, `bytes.Buffer`). Evite construtor obrigatório quando o zero basta.
- **Composição, não herança:** embedding para reuso; sem hierarquia de tipos.
- **Erros são valores** — não exceções. `panic` só para "impossível/bug de programação", nunca controle de fluxo. `recover` só na fronteira (ex.: middleware que transforma panic em 500 + log).
- **Formatação não é opinião:** `gofmt`/`goimports` sempre. `golines`/`gofumpt` opcionais.
- **`defer`** para liberar recurso perto de onde adquire (`defer f.Close()`, `defer rows.Close()`). Cuidado: `defer` em loop acumula até o fim da função — extraia função se necessário.
- **Aceite `context.Context` como 1º parâmetro** em qualquer função que faz I/O ou pode ser cancelada.

---

## 2. Layout de projeto (cmd / internal / pkg)

Segue o "Standard Go Project Layout" (comunidade) + camadas do Locus, organizado **por contexto de negócio**:

```
backend/
├── go.mod  go.sum
├── cmd/
│   └── lunar-api/main.go        # composição/wiring; só monta e sobe
├── internal/                    # privado ao módulo (o compilador garante)
│   ├── config/                  # carga + validação fail-closed (aborta boot em prod incompleta)
│   ├── http/                    # handlers finos + middlewares (auth→rbac→escopo→validação→auditoria)
│   ├── security/                # ⚠️ fronteira dura: crypto, sessão, ratelimit, hash-chain (revisão dupla)
│   ├── platform/                # infra transversal: logger, db pool, tx/UoW, telemetria
│   ├── contratos/               # um contexto de negócio
│   │   ├── domain/              #   entidades + invariantes + cálculo PURO (sem I/O)
│   │   ├── service/             #   regra/orquestração; depende de PORTAS (interfaces)
│   │   ├── ports.go             #   interfaces de repositório/integração (definidas aqui)
│   │   └── repo/                #   adapter que implementa as portas (pgx/sqlc vive AQUI)
│   ├── empenhos/  pagamentos/  compras/  ...   # mesmos submódulos
│   └── money/                   # fonte ÚNICA de cálculo financeiro (value object Dinheiro)
├── pkg/                         # reutilizável por terceiros (use com parcimônia)
│   └── integrations/            # adapters de fornecedor (Clerk, storage, e-mail...)
└── migrations/                  # SQL versionado, append-only (empilha, nunca destrói)
```

- **`internal/`** é privado por regra do compilador — use-o para quase tudo. `pkg/` só para o que realmente seria importável de fora.
- **`cmd/<bin>/main.go`** é fino: lê config, monta dependências (injeção manual de interfaces), sobe o servidor. Sem regra de negócio.
- **Domínio não importa infraestrutura.** `contratos/domain` nunca importa `pgx`, `net/http` ou Clerk. Se importar, é bug.

---

## 3. Gestão de erro

```go
// Embrulhar preservando a cadeia (%w):
if err != nil {
    return fmt.Errorf("salvar contrato %s: %w", id, err)
}

// Inspecionar por sentinela:
var ErrNaoEncontrado = errors.New("não encontrado")
if errors.Is(err, ErrNaoEncontrado) { /* 404 */ }

// Inspecionar por tipo (extrai dados):
var vErr *ValidacaoError
if errors.As(err, &vErr) { /* usa vErr.Campos */ }
```

- **Sempre** trate ou propague — nunca `_ = err`. Só descarte deliberado e comentado (`_ = f.Close() // best-effort em cleanup`).
- **Embrulhe com contexto** ao subir de camada; **use `%w`** quando o chamador possa querer inspecionar; use `%v` quando quiser esconder a causa (fronteira externa).
- **Erros sentinela** (`errors.New`) para condições que o chamador testa; **tipos de erro** quando precisa carregar dados. Múltiplos: `errors.Join`.
- **Fronteira com o usuário:** traduza erro interno para mensagem segura (sem stacktrace, sem SQL, sem segredo, sem detalhe de sistema). Log detalhado fica no servidor; resposta ao cliente é enxuta. Anti-enumeração: recurso fora de escopo → **404**, não 403.
- Não use `panic`/`log.Fatal` fora de `main`/inicialização. Biblioteca retorna erro; quem decide abortar é o topo.

---

## 4. context.Context

- **1º parâmetro, sempre `ctx`.** Nunca guarde `Context` em struct; passe-o.
- Propague o `ctx` da requisição por toda a cadeia (handler→service→repo→pgx). Respeite cancelamento: operações longas checam `ctx.Err()` ou usam APIs que aceitam `ctx`.
- **Timeout/deadline** na borda: `ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second); defer cancel()`. `context.AfterFunc` (Go 1.21+) para cleanup no cancelamento.
- **`context.WithValue`** só para dados de escopo de requisição (ex.: `RequestID`, identidade autenticada), com **chave de tipo próprio não-exportado** para evitar colisão — nunca para passar parâmetros opcionais de função.

---

## 5. Concorrência segura

- **Goroutine precisa de dono e fim.** Quem inicia sabe quando termina. Goroutine sem via de saída = vazamento.
- **`context` cancela; channel comunica; mutex protege estado.** "Don't communicate by sharing memory; share memory by communicating" — mas mutex simples é ok para contador/cache.
- **`golang.org/x/sync/errgroup`** para fan-out com propagação de erro e cancelamento:
  ```go
  g, ctx := errgroup.WithContext(ctx)
  for _, it := range itens {
      it := it // pré-1.22; em Go 1.22+ o loopvar já é por-iteração
      g.Go(func() error { return processa(ctx, it) })
  }
  if err := g.Wait(); err != nil { /* ... */ }
  ```
- **`sync.WaitGroup`** para esperar sem coletar erro; **`sync.Once`** para init único; **`sync.Mutex`/`RWMutex`** para estado; **`atomic`** para contadores.
- **`-race` obrigatório** em CI e no teste local de qualquer código concorrente. Data race é bug, mesmo que "funcione".
- **Anti-lockout / seções críticas de RBAC** (ex.: não zerar todos os admins): serializar na camada de dados via transação, não só mutex de processo (vários processos). Ver protocolo HARD-11.
- Channel: feche do lado do **produtor**, nunca do consumidor; `nil` channel bloqueia para sempre (útil em `select`); use `select` com `ctx.Done()` para cancelar.

---

## 6. Generics (Go 1.18+)

- Use quando remove duplicação **real** com type-safety: containers (`Set[T]`), utilitários (`Map`/`Filter`/`Keys`), `Result[T]`, mapeadores linha↔domínio.
- Constraints: `comparable`, `any`, ou interfaces de conjunto de tipos (`constraints.Ordered` de `golang.org/x/exp/constraints`, ou defina a sua).
- **Não** troque interface por generic sem ganho; **não** generalize prematuramente. Se um `any` + type switch resolve com clareza, ele pode ser melhor.
- Regra prática: "escreva o código concreto duas vezes antes de generalizar".

---

## 7. Testing

- **Table-driven é o padrão:**
  ```go
  func TestCalculaTotal(t *testing.T) {
      casos := []struct {
          nome    string
          itens   []Item
          querErr bool
          quer    Dinheiro
      }{
          {"vazio", nil, false, Zero},
          {"soma simples", []Item{cents(150), cents(250)}, false, cents(400)},
          // casos-limite financeiros: zero, negativo, arredondamento, muitos itens
      }
      for _, c := range casos {
          t.Run(c.nome, func(t *testing.T) {
              got, err := CalculaTotal(c.itens)
              if c.querErr { require.Error(t, err); return }
              require.NoError(t, err)
              require.True(t, c.quer.Equal(got), "quer %s, veio %s", c.quer, got)
          })
      }
  }
  ```
- **`testify`:** `require` (aborta o teste no fail — use para pré-condições) vs `assert` (segue). `require.NoError` antes de usar o resultado.
- **`t.Parallel()`** em subtests independentes; cuidado com estado compartilhado e loopvar.
- **Handlers:** `net/http/httptest` (`httptest.NewRequest` + `httptest.NewRecorder`). **Domínio/service:** teste contra **fakes das portas** (structs que implementam a interface), sem banco.
- **Financeiro:** além de table tests, **testes de propriedade** (ex.: `soma(partes) == total`; `arredonda(arredonda(x)) == arredonda(x)`) e casos-limite. Determinístico sempre (sem `time.Now()`/random direto — injete relógio/seed).
- **`testing.F`** para fuzzing de parsers/validadores de input. Cobertura: `go test -cover`; mire no que importa (regra/segurança/dinheiro), não em número.

---

## 8. Módulos & versionamento

- `go mod init <module>`, `go get pkg@versão`, `go mod tidy` (remove não usados, fixa `go.sum`), `go mod verify`, `go mod download`.
- **Fixe a versão do toolchain** no `go.mod` (`go 1.2x` + linha `toolchain`). Atualize deliberadamente (CVE de stdlib se corrige subindo o toolchain — lição do Locus).
- Prefira **stdlib**; cada dependência é superfície de ataque e manutenção. Avalie com `govulncheck` antes e depois de adicionar.
- `replace`/`exclude` no `go.mod` só com justificativa. `GOFLAGS=-mod=readonly` em CI para builds reproduzíveis.

---

## 9. Ferramentas (o "pronto" de verdade)

Rode tudo antes de considerar código pronto:

| Ferramenta | Para quê |
|---|---|
| `gofmt` / `goimports` | formatação + imports (obrigatório) |
| `go vet` | erros suspeitos que compilam (printf, locks copiados, tags) |
| `staticcheck` | linter forte (código morto, bugs sutis, simplificações) |
| `gosec` | SAST de segurança (SQL concat, crypto fraca, segredo hardcoded, perms de arquivo) |
| `govulncheck` | CVEs em dependências **e na stdlib**, com análise de alcançabilidade |
| `go test -race -cover` | testes + detector de data race |
| `trivy fs` / `gitleaks` | (CI) vulnerabilidades de deps e segredos vazados |

No Lunar isso é **gate**, não sugestão: `gosec`/`staticcheck` com achado HIGH param o trabalho (stop condition, herdada do Locus). Segurança tem veto.

---

## 10. Dinheiro — precisão fixa (NUNCA float) ⚠️

Regra inviolável (constituição §10, protocolo QA-12). Duas estratégias válidas; escolha uma por ADR e **não misture**:

**A) `int64` em centavos** (menor unidade da moeda):
- Simples, exato, rápido, serializa trivial. Bom para BRL onde tudo é centavo.
- Cuidado: multiplicação/percentual/rateio geram fração → **regra de arredondamento explícita** (ex.: half-even/bankers, ou half-up documentado) e trate o "resto" do rateio (distribua os centavos que sobram deterministicamente).

**B) `github.com/shopspring/decimal`** (decimal de precisão arbitrária):
- Para percentuais, taxas, câmbio, muitas casas. `decimal.NewFromInt`, `decimal.NewFromString` (nunca `NewFromFloat` com literal de dinheiro), `.Mul`, `.Div` com `.DivRound`, `.Round(2)`.
- Mais seguro para aritmética complexa; custo de performance/alocação aceitável no domínio financeiro.

Invioláveis em qualquer caso:
- **Nunca** `float32/float64` para valor monetário, nem intermediário. Nem `JSON` number solto — desserialize dinheiro como string/inteiro.
- **Fonte única de cálculo:** um pacote `internal/money` com o value object `Dinheiro` e as operações; ninguém recalcula por fora.
- **Arredondamento documentado e testado**; casos-limite (zero, negativo, soma de N itens, rateio, percentual) com testes de propriedade.
- **Validação cruzada** (segundo método/segundo agente) antes de persistir/exibir; invariante `soma(partes)==total` checada em runtime (divergência → bloqueio + auditoria).
- Coluna de banco: inteiro (centavos) ou `NUMERIC/DECIMAL` — **jamais `float`/`double`**.

---

## 11. pgx / sqlc (tudo dentro do adapter)

O banco **não está decidido** (dono quer combinar dois) → pgx/sqlc vivem **só** em `internal/<ctx>/repo`, implementando a porta. `service`/`domain` não importam pgx.

**pgx/v5:**
```go
// pool no platform/, injetado no adapter:
pool, err := pgxpool.New(ctx, dsn)   // configure timeouts, MaxConns
// query parametrizada — SEMPRE ($1,$2...), nunca fmt.Sprintf em SQL:
row := pool.QueryRow(ctx, `SELECT id, total_cents FROM contratos WHERE id=$1`, id)
var c contratoRow
if err := row.Scan(&c.ID, &c.TotalCents); err != nil {
    if errors.Is(err, pgx.ErrNoRows) { return nil, domain.ErrNaoEncontrado }
    return nil, fmt.Errorf("buscar contrato: %w", err)
}
// coleta tipada de várias linhas:
rows, _ := pool.Query(ctx, q, args...)
itens, err := pgx.CollectRows(rows, pgx.RowToStructByName[contratoRow])
```
- **SQL sempre parametrizado.** Zero concatenação de input em SQL (gosec pega; é o achado nº 1). Identificadores dinâmicos (nome de coluna) só de allowlist, nunca do usuário.
- **Transação como porta (UoW):** o adapter expõe `WithTx(ctx, func(Tx) error)`; o service coordena atomicidade sem conhecer pgx. `pgx.Tx` com `defer tx.Rollback(ctx)` + `tx.Commit(ctx)` no sucesso.
- **`context`** em toda chamada pgx; timeout herdado da requisição.

**sqlc:** escreva SQL em `.sql`, gere Go tipado (`sqlc generate`). Vantagem: queries checadas contra o schema, structs gerados, menos boilerplate — mas o código gerado ainda fica atrás da porta (o service usa a interface, não o `Queries` do sqlc diretamente). Bom para reduzir erro humano em query.

---

## 12. Repositório atrás de interface (porta/adapter)

O padrão central do Lunar (banco indefinido, hexagonal). A porta é definida pelo **consumidor**:

```go
// internal/contratos/ports.go — definido pelo service, tipos são do DOMÍNIO
type ContratoRepo interface {
    Salvar(ctx context.Context, c *domain.Contrato) error
    PorID(ctx context.Context, id domain.ID) (*domain.Contrato, error)
    Inativar(ctx context.Context, id domain.ID, motivo string) error // nunca "Deletar"
}

// internal/contratos/service/service.go — depende da INTERFACE
type Service struct{ repo ContratoRepo; audit AuditPort }

// internal/contratos/repo/pg.go — adapter; ÚNICO lugar com pgx
type PgContratoRepo struct{ pool *pgxpool.Pool }
func (r *PgContratoRepo) PorID(ctx context.Context, id domain.ID) (*domain.Contrato, error) { /* SQL + map linha→domínio */ }
```

- **Tipos do domínio cruzam a porta**, nunca `contratoRow`/struct de banco. Mapeamento linha↔domínio é interno ao adapter.
- **Dois bancos** = decisão de adapter (ADR do arquiteto): repositório composto que roteia por operação; ou outbox + projeção; ou R/W split. O service não muda.
- **Nada de `Deletar` na porta:** só `Inativar`/`Cancelar` (soft-delete, histórico preservado). Registro aprovado trava para edição.
- **Auditoria** também é porta (append-only, hash-chain), fora do alcance de escrita comum.
- **Teste do domínio/service** usa um fake da porta (`type fakeRepo struct{...}`), sem banco — rápido e determinístico. O adapter real tem seu próprio teste de integração.

---

## Conexão com a Camada 0

Este documento serve a segurança: SQL parametrizado (anti-injeção), erro sem vazamento (anti-enumeração + sem segredo em log), context/timeout (anti-DoS), dinheiro exato (integridade financeira), banco atrás de porta (evita acoplamento e vaza menos), `-race`/`gosec`/`govulncheck` (gate de qualidade). Código em `internal/security` = **revisão dupla obrigatória** (humano + `appsec-go`). Na dúvida, **fail-closed**: negar/parar e escalar. Ver `security/protocols/00-PRINCIPIOS-CAMADA-0.md`, `11-HARDENING-APLICACAO.md`, `12-DUPLA-VALIDACAO-E-INTEGRIDADE.md`.
