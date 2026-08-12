---
protocolo: SEC-05
titulo: Gestão de Segredos
status: canônico
atualizado: 2026-07-21
---

# SEC-05 — Gestão de Segredos

> Regra nº 1 da constituição: **segredo nunca** em git, log, chat, doc versionada ou junto a IP. Este protocolo consolida os erros dos três sistemas de referência (PAT no `.git/config`, `.env` de 52 chaves commitado, `sk_test`/chave iDrive coladas em `.env`, `.data/` no bundle de deploy) para **não repeti-los**.

## 1. Onde os segredos vivem

- **Produção:** **sops + age** (arquivos cifrados versionáveis) e/ou **Docker Swarm secrets** (montados em runtime). **Nunca** env plano versionado.
- **Local/dev:** arquivos `.env` com permissão `600`, **fora** de qualquer pasta versionada/sincronizada. Gerar segredo escrevendo **direto no arquivo**, nunca ecoando no chat/stdout.
- **Chave privada de backup (`age`) e chaves-mestras:** guarda **offsite**, removidas do host após uso (padrão Locus).

## 2. Regras invioláveis

1. `.gitignore` cobre `.env`, `.env.*` (exceto `.env.example`), `*.pem`, `*.key`, `*.dump`, `.data/`, `BACKUP-*/`.
2. **Nunca** commitar segredo — nem no histórico. Se vazar, **rotacionar na origem** (reescrever git não basta).
3. **Nunca** colocar segredo em log (no máximo prefixo de 8 chars para correlação), em doc, no chat, ou junto ao IP/host.
4. **Nunca** empacotar estado/segredo (`.data/`, DBs) no artefato de deploy. *(Erro do NexCollabs.)*
5. `.env.example` só com **nomes** de variáveis, jamais valores.

## 3. Detecção & rotação

- **gitleaks** (pre-commit + CI) e **trufflehog** varrem segredo exposto. Achado = bloqueio (Squad Segredos, fail-closed).
- **Rotação** periódica e imediata em suspeita de vazamento. Rotação é registrada no [`../CHANGELOG-SECURITY.md`](../CHANGELOG-SECURITY.md).

Dono: Squad AppSec (`sec-segredos`) + Squad DevOps (entrega). Ver também [`04-AUTH-CLERK.md`](04-AUTH-CLERK.md), [`08-INFRA-HARDENING.md`](08-INFRA-HARDENING.md).
