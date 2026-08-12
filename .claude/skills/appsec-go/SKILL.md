---
name: appsec-go
description: Checklist acionável de AppSec para o backend Go do Projeto Lunar. Use ao escrever ou revisar código Go, rodar SAST/deps (gosec, govulncheck, staticcheck), validar entrada, tratar erro com segurança, prevenir injeção (SQL/comando/path), e conferir os padrões anti-IDOR e de dinheiro em precisão fixa. Gatilhos: "revisar Go", "gosec", "govulncheck", "injeção", "IDOR", "validação de entrada", "PR do backend".
---

# Skill: AppSec Go (Projeto Lunar)

Conhecimento na ponta da língua para blindar o backend **Go** do Lunar (governamental, auth Clerk, precisão financeira absoluta). Use com os agentes `appsec-go`, `sec-isolamento-acesso`, `sec-auth-webhooks`, `sec-segredos`. Alinhado a [`security/protocols/11-HARDENING-APLICACAO.md`](../../../security/protocols/11-HARDENING-APLICACAO.md) e [`00-PRINCIPIOS-CAMADA-0.md`](../../../security/protocols/00-PRINCIPIOS-CAMADA-0.md).

## 1. Bateria SAST/deps (a cada commit)

```bash
gosec -severity medium -confidence medium ./...      # SAST; HIGH = para o trabalho
govulncheck ./...                                     # CVE ALCANÇÁVEL no call-graph (não só listada)
staticcheck ./...                                     # bugs/estilo/correção
go vet ./...                                          # análise do compilador
trivy fs --scanners vuln,secret,misconfig .           # deps + segredo + config
```
Regra: `gosec`/`staticcheck` **HIGH** ou `govulncheck` com CVE alcançável = **auto-veto** (para e reporta ao Security Master). Falso-positivo só passa **documentado e justificado** (`#nosec` com motivo, revisado).

## 2. Injeção — checklist

- [ ] **SQL:** sempre query parametrizada / prepared statement (`$1`, `?`). **ZERO** concatenação/`fmt.Sprintf` montando SQL com input. Nome de tabela/coluna dinâmico só por allowlist.
- [ ] **Comando:** `exec.Command("bin", arg1, arg2)` — nunca `sh -c` com input interpolado. Sem shell.
- [ ] **Path traversal:** validar nome/caminho — bloquear `..`, prefixos `/`, `-`; usar `filepath.Clean` + verificar que o resultado está dentro do dir base.
- [ ] **Template/HTML:** `html/template` (auto-escape), nunca `text/template` para saída web.
- [ ] **Deserialização:** decoder estrito (ver §3).

## 3. Validação de entrada (DTO estrito)

```go
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()          // rejeita campo desconhecido
```
- [ ] `DisallowUnknownFields()` em todo DTO de entrada.
- [ ] Validar tipo, tamanho, formato (e-mail, IDs, faixa de valor); rejeitar vazio/negativo onde inválido.
- [ ] **Body limit** (ex.: `http.MaxBytesReader`, ~2 MB) + **timeout** (~30 s) no handler.
- [ ] `id`/identidade **nunca** confiando no body para escopo/tenant — vem do claim (ver §6).

## 4. Tratamento de erro seguro

- [ ] Erro ao cliente é **genérico** (sem stack, sem estrutura interna, sem SQL, sem segredo). Detalhe vai para o log estruturado (JSON).
- [ ] **Nunca** logar senha/token/chave/PII crua — no máximo prefixo de 8 chars para correlação.
- [ ] Sempre checar `err`; nunca engolir erro que deveria falhar-fechado. `defer` em `Close()` com checagem onde relevante.
- [ ] Sem `panic` em handler (recover no middleware, resposta 500 genérica).

## 5. Padrões seguros de Go

- [ ] `crypto/rand` para qualquer valor de segurança (token, nonce, salt). **Nunca** `math/rand`.
- [ ] Comparação de segredo/HMAC: `hmac.Equal` / `subtle.ConstantTimeCompare` (constant-time).
- [ ] `context.Context` com timeout propagado em I/O externo.
- [ ] Crypto (se houver credencial local — HARD-11 §1): `argon2.IDKey` (m=19456, t=2, p=1, 32B) para hash de senha; `chacha20poly1305.NewX` com **AAD contextual** (`entidade:campo:id`) para cifra em repouso; limpar `[]byte` sensível após uso (zeroize). Vive em `internal/security` = **revisão dupla**.

## 6. Anti-IDOR / escopo (as DUAS camadas — corrige o One Nexus)

Ter a permissão **não** autoriza ler dado alheio. Toda leitura passa por **permissão E escopo**:

```go
// 1) Permissão (string RBAC) — middleware
RequirePermission("contratos:registro:ler")
// 2) Escopo/ownership — NA QUERY, não num if pós-carga
repo.BuscarContrato(ctx, id, escopoDo(ator))   // WHERE id=$1 AND <ownership do ator>
```
- [ ] Nenhum `id` do request vai direto ao repositório sem filtro de ownership/hierarquia.
- [ ] Escopo entra no **WHERE**, nunca decidido por substring/querystring ou no front.
- [ ] Recurso fora do escopo → **404** (anti-enumeração), não 403.
- [ ] Acesso **negado** é auditado (protocolo 02). Identidade/tenant **só do claim** Clerk validado.
- [ ] Existe suíte adversarial de bypass cross-perfil (A tenta ler/editar recurso de B por body/path/query/ID sequencial).

## 7. Dinheiro (integridade financeira — regra 10)

- [ ] **Nunca** `float`/`float64` para valor monetário. Usar centavos `int64` ou `shopspring/decimal`, com arredondamento explícito e documentado.
- [ ] Fonte **única** de cálculo (sem cálculo duplicado espalhado).
- [ ] Encaminhar todo cálculo ao squad **Integridade Financeira** para validação cruzada.

## 8. Stop conditions (auto-veto ao Security Master)

- gosec/staticcheck HIGH · govulncheck CVE alcançável.
- SQL concatenado · `exec` com shell · input sem validação.
- Teste de bypass IDOR que passa · escopo decidido no front.
- Segredo detectado no diff (→ `sec-segredos`) · JWT sem azp/iss/aud (→ `sec-auth-webhooks`).
- `float` para dinheiro.

## 9. Fronteiras (encaminhe, não invada)

| Tema | Dono |
|---|---|
| JWT/Clerk/webhook/SSRF/2FA | `sec-auth-webhooks` |
| Escopo/RBAC/BOLA/IDOR | `sec-isolamento-acesso` |
| Segredo em git/log, rotação | `sec-segredos` |
| Cálculo/dinheiro | squad Integridade Financeira |
| Dado pessoal / prazo legal | squad LGPD (`lgpd-gov`) |
