---
description: Configura a base de segurança Soberis sob medida (infere + pergunta + gera, sem TTY)
---

Você vai configurar o **Soberis Secure Baseline** para este projeto. Siga o maestro em `./CLAUDE.md` (raiz deste repo):

1. **Infira** do projeto atual: stack, se tem banco, se tem auth, se mexe com dinheiro, se guarda dado pessoal, e se o deploy é Compose ou Swarm.
2. **Pergunte** ao humano o porte (🐜 formiga / 🔧 padrão / 💥 canhão / custom) via AskUserQuestion, e só os gatilhos que você não inferiu com confiança.
3. **Escreva** `baseline.answers.json` (formato em `baseline.answers.example.json`), mostre o resumo do que liga/desliga e **peça confirmação**.
4. **Rode** `node setup.mjs` (não-interativo — lê o answers.json).
5. **Relate** o que ficou ligado, o time de agentes, e aponte o `CHECKLIST-PR.md`. Rode `git status`.

Regra: na dúvida sobre ligar um módulo de segurança, o default é **ligar** (fail-closed). Nunca commite com segredo — confira o `.gitignore`.

$ARGUMENTS
