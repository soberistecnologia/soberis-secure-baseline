---
name: <stack>-<concern>            # ex.: node-appsec, python-secrets, k8s-hardening
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

## Verificação (gate — regra 12)
- [ ] Multi-fonte (≥2 fontes concordam nas regras críticas)
- [ ] `ai-agent-security` vetou (sem instrução oculta / conselho perigoso)
- [ ] `qa-adversarial` tentou furar
- [ ] `security-master` aprovou
