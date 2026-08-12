---
protocolo: DEL-07
titulo: Soft-Delete, Versionamento e Histórico
status: canônico
fonte_requisito: Docs_dev/Perfil de Usuário.docx
atualizado: 2026-07-21
---

# DEL-07 — Soft-Delete, Versionamento e Histórico

> Requisitos do cliente: *"evitar exclusão definitiva — utilizar 'inativar' ou 'cancelar', mantendo o histórico"* e *"permitir visualizar todas as versões de um registro e as alterações realizadas."* Em sistema de recurso público, **nada some**.

## 1. Sem exclusão física

- **Proibido** `DELETE` físico de dado de negócio. Ações permitidas: **inativar** ou **cancelar** (soft-delete), preservando o registro e o histórico.
- Estado do registro tem campo de situação (`ativo`/`inativo`/`cancelado`) + quem/quando/por quê (auditado).
- Expurgo só quando legalmente permitido (fim do prazo de retenção — ver LGPD [`09-LGPD-COMPLIANCE.md`](09-LGPD-COMPLIANCE.md)); e mesmo o expurgo é **auditado**.

## 2. Versionamento & histórico

- Cada alteração relevante cria/atualiza versão com **antes/depois** e diff (coerente com [`02-AUDITORIA-LOGS.md`](02-AUDITORIA-LOGS.md)).
- **Registro aprovado é imutável**; alteração vira **nova versão** após reabertura formal ([`06-FLUXO-APROVACAO.md`](06-FLUXO-APROVACAO.md)).
- A tela do registro mostra **todas as versões** e quem mudou o quê (requisito "Histórico de Alterações").

## 3. Retenção / temporalidade

- Guarda alinhada a controle interno/TCU + tabela de temporalidade + LGPD. Dado sob obrigação legal de guarda **não é eliminado** enquanto durar o prazo. Detalhe pendente de 2ª rodada LGPD (item 🔲).

Dono: Arquitetura + Squad Auditoria & Compliance.
