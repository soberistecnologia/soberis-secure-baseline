---
titulo: Modelo de Ameaça — projeto
status: vivo
atualizado: 2026-07-21
---

# Modelo de Ameaça — projeto

> Sistema de gestão financeira de **órgão público**. Ativos = dados pessoais de servidores/fornecedores + movimentação de recursos públicos + trilha de auditoria. Atacante de interesse elevado (fraude ao erário, vazamento, adulteração de registro). Metodologia: STRIDE + lições dos 3 sistemas de referência.

## 1. Ativos a proteger

| Ativo | Por quê | Impacto se comprometido |
|---|---|---|
| Dados pessoais (servidores/fornecedores) | LGPD, dados sensíveis | Vazamento, sanção, dano a titular |
| Registros financeiros (empenhos, pagamentos, DDF) | Recurso público | Fraude, dano ao erário |
| **Trilha de auditoria** | Accountability TCU | Perda de prova, impunidade |
| Fluxo de aprovação | Governança | Aprovação forjada, burla de SoD |
| Credenciais/segredos | Acesso ao sistema | Takeover |

## 2. Ameaças (STRIDE) e mitigações

| Categoria | Ameaça | Mitigação | Protocolo |
|---|---|---|---|
| **Spoofing** | Falsificar identidade/token | Clerk JWKS + iss/aud/azp + 2FA aprovadores | [04](../protocols/04-AUTH-CLERK.md) |
| **Tampering** | Adulterar registro/trilha | Auditoria append-only + hash-chain; registro aprovado travado | [02](../protocols/02-AUDITORIA-LOGS.md) · [06](../protocols/06-FLUXO-APROVACAO.md) |
| **Repudiation** | Negar autoria de ação | Trilha total, assinatura na aprovação (step-up) | [02](../protocols/02-AUDITORIA-LOGS.md) |
| **Information disclosure** | Vazar dado (IDOR/BOLA, enumeração) | Escopo/ownership em toda leitura, 404 anti-enum, cripto | [03](../protocols/03-ISOLAMENTO-DADOS.md) · [11](../protocols/11-HARDENING-APLICACAO.md) |
| **Denial of service** | Derrubar/abusar | Rate-limit anti-XFF-spoof, body/timeout | [11](../protocols/11-HARDENING-APLICACAO.md) |
| **Elevation of privilege** | Auto-promoção / burlar SoD | RBAC deny-vence, SoD (edita≠aprova), sem fail-open | [01](../protocols/01-RBAC-PERFIS.md) |

## 3. Ameaças específicas herdadas dos 3 sistemas (não repetir)

| Origem | Erro | Nossa defesa |
|---|---|---|
| One Nexus | Tenant/escopo por substring de URL | Escopo por identidade+regra, nunca string ([03](../protocols/03-ISOLAMENTO-DADOS.md)) |
| One Nexus | RBAC sem ownership (IDOR) | Ownership em toda leitura ([01](../protocols/01-RBAC-PERFIS.md)) |
| One Nexus | Webhook fail-open / `search_path` sem LOCAL | Webhook secret obrigatório; acesso a dados atrás de porta segura |
| One Nexus | Segredos no git (PAT, `.env` 52 chaves) | sops+age, gitleaks, rotação ([05](../protocols/05-GESTAO-SEGREDOS.md)) |
| Locus | `sk_test` em produção; Docker `:2375` aberto | Só `sk_live`; API Docker nunca exposta ([04](../protocols/04-AUTH-CLERK.md) · [08](../protocols/08-INFRA-HARDENING.md)) |
| NexCollabs | `.data/` (hashes) no bundle de deploy | Artefato sem estado/segredo ([05](../protocols/05-GESTAO-SEGREDOS.md)) |

## 4. Ameaça-alvo do domínio: fraude financeira

- **Aprovação forjada:** mitigada por SoD + step-up + 2FA + auditoria assinada.
- **Alteração pós-aprovação:** registro travado; só nova versão via reabertura formal auditada.
- **Erro/manipulação de cálculo:** integridade financeira (precisão fixa, validação cruzada por 2º agente) — [12](../protocols/12-DUPLA-VALIDACAO-E-INTEGRIDADE.md).

## 5. Superfície & confiança

- Fronteira pública: **só o Traefik** (TLS). Tudo mais em rede privada.
- Confiança zero em entrada de usuário e no front (validação e autorização sempre no backend).
- Agentes de IA: tiers de autonomia (auto-safe/auto-veto/humano); mudança de schema/prod/RBAC é sempre humano.

> Documento vivo — revisado a cada security-review e atualizado quando surgir novo vetor. Dono: Security Master.
