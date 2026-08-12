---
description: Roda o double-check — um validador de OUTRA família de modelo (DeepSeek/Qwen/GLM/Kimi) refuta a peça de segurança. Claude decide.
---

Você vai submeter uma peça de segurança (skill forjada, review, achado, ou o veredito) ao **double-check**: um modelo de **outra família** valida e tenta **refutar**. Doutrina em `docs/DOUBLE-CHECK.md`.

## Passos
1. **Confirme as chaves** — `NVIDIA_API_KEY` e/ou `OLLAMA_API_KEY` no `.env` local. Se faltarem: **PARE** (fail-closed), marque a peça como `rascunho` e diga "double-check indisponível". Não valide sozinho.
2. **Escolha um validador de família diferente** de quem produziu a peça (skill forjada por Claude → valida com DeepSeek/Qwen/GLM/Kimi).
3. **Mande o material + a instrução de refutar:** "Ache brechas, o que falta, conselho errado/perigoso. Refute com evidência. Isto vai proteger projetos reais — seja implacável."
4. **Anexe a devolutiva** em `security/reviews/<data>-doublecheck-<peça>.md`, com modelo + data.
5. **Se refutar:** o Claude (autoritativo) **resolve ou documenta** cada ponto — nunca ignora. Só então a peça sai de `rascunho`.

> Provedores: NVIDIA NIM (`https://integrate.api.nvidia.com/v1`, OpenAI-compat) e Ollama Cloud. Use o SDK/curl OpenAI-compat com a chave do `.env`. Nunca imprima a chave.

$ARGUMENTS
