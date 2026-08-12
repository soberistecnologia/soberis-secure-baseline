---
name: <stack>-<concern>            # ex.: node-appsec, python-appsec, k8s-hardening
description: <o que a skill cobre e QUANDO acionar>. Forjada pela pesquisa para a stack real do projeto. Gatilhos: <palavras>.
forja:
  stack_detectada: <ex.: Node 20 + Express 4 + Postgres>
  gerada_em: <data>
  status: rascunho            # rascunho → vetada (só 'vetada' é confiável)
  vetada_por: []              # ai-agent-security, security-master, qa-adversarial
---

# <Título> — skill forjada para <stack>

> ⚠️ **Só use se `status: vetada`.** Skill de segurança não vetada é hipótese, não verdade.

## Resumo acionável
<As regras práticas — o "faça isso" — específicas desta stack. Curto e direto.>

## Ferramentas desta stack
| Concern | Ferramenta | Comando |
|---|---|---|
| SAST | <ex.: semgrep> | `<comando>` |
| Deps | <ex.: npm audit / govulncheck> | `<comando>` |
| Segredos | <ex.: gitleaks> | `<comando>` |

## Ciladas conhecidas do framework
- <cilada 1 + como evitar>
- <cilada 2 + como evitar>

## PROCEDÊNCIA (obrigatório — sem isto a skill NÃO passa no gate)
Cada regra acima cita de onde veio. Doc oficial > blog. Versão e data.
- [ ] `<regra>` → `<fonte/URL>` (<doc oficial | CVE | OWASP | blog verificado>)
- [ ] ...

## Verificação (gate MECÂNICO — regra 12)
> ⚠️ O `status: vetada` no frontmatter **não é a prova** — quem forja não se auto-atesta.
> A prova é o **artefato de review** em `security/reviews/<data>-skill-<nome>.md`, escrito por outro
> revisor, + a devolutiva do double-check. Sem esses arquivos, trate como `rascunho` **mesmo que o YAML diga vetada**.
- [ ] Multi-fonte: ≥2 **fontes oficiais** concordam nas regras críticas (blog não conta como fonte única)
- [ ] **Double-check** (`/double-check`): validador de **outra família** (DeepSeek/Qwen/GLM/Kimi) tentou refutar — devolutiva anexada em `security/reviews/`
- [ ] `ai-agent-security` vetou (sem instrução oculta / conselho perigoso)
- [ ] `security-master` (Claude, autoritativo) resolveu ou documentou cada refutação
- [ ] Se pesquisa fraca / stack obscura sem boa fonte → **fail-closed: não forja, PARA** (não inventa)
