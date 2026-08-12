---
protocolo: ISO-03
titulo: Isolamento e Escopo de Dados (single-tenant, multi-ready)
status: canônico
atualizado: 2026-07-21
---

# ISO-03 — Isolamento e Escopo de Dados

> O Lunar é **single-tenant** (um órgão). Não há schema-per-tenant como no Locus, mas **há escopo interno**: um usuário só acessa os dados do seu âmbito (unidade/centro de custo/módulo). Este protocolo corrige o **IDOR do One Nexus** (autorização que não conhecia ownership) e mantém a fundação **pronta para multi-tenant** no futuro.

## 1. Princípio

**Toda operação — inclusive leitura — valida o escopo do recurso, não só a permissão.** Ter `contratos:registro:consultar` não basta: o registro precisa pertencer ao escopo autorizado do usuário.

## 2. Regras

1. **Nunca** confiar em identificador "cru" do parâmetro/rota/querystring para buscar um recurso sem checar escopo (anti-BOLA/IDOR).
2. **Nunca** resolver escopo por substring de URL/querystring (erro SEC-020 do One Nexus). Escopo vem de identidade + regra, não de string de caminho.
3. Recurso fora do escopo → **404** (não 403), para não vazar existência (anti-enumeração).
4. Consultas parametrizadas sempre; nome de tabela/schema nunca interpolado a partir de entrada do usuário.
5. A verificação de escopo é **centralizada** (um `RequireScope`), testada por suíte adversarial de bypass.

## 3. Pronto para multi-tenant (futuro)

- Modelar com `tenant_id`/`org_id` presente desde já onde fizer sentido, mesmo com um só tenant ativo.
- Camada de repositório atrás de **porta/adapter** (o banco ainda não foi decidido — dono quer combinar dois bancos): a decisão de isolamento vive numa fronteira única, fácil de trocar por RLS FORCE (padrão Locus) se virar multi-tenant.

## 4. Ligações

Autorização: [`01-RBAC-PERFIS.md`](01-RBAC-PERFIS.md). Hardening/404 anti-enum: [`11-HARDENING-APLICACAO.md`](11-HARDENING-APLICACAO.md). Dono: Squad AppSec (`sec-isolamento-acesso`).
