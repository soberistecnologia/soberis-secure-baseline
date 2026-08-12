# Double-Check — validação por modelos independentes

> Princípio: **nenhuma conclusão de segurança vale com um cérebro só.** O Claude é
> autoritativo, mas outra família de modelo **valida e refuta** antes de confiar. É a
> dupla validação (Executor ≠ Validador) levada à camada de modelo.

## Por que (o furo que isso fecha)
Três agentes Claude revisando os mesmos inputs falham **do mesmo jeito** — uma fonte
envenenada convincente engana os três igual. Independência de verdade vem de **famílias
diferentes**, que erram diferente. Sem isso, "dupla validação" é uma instância revisando a si mesma.

## Papéis
- **Autoritativo — Claude** (ex.: `fable` para as tarefas mais críticas): conduz, decide, é a palavra final.
- **Validadores independentes — outra família** (DeepSeek, Qwen, GLM, Kimi via NVIDIA NIM / Ollama Cloud):
  recebem o mesmo material de segurança, **procuram brecha, o que falta, e REFUTAM**. Entregam uma
  **devolutiva**, não a decisão.
- **Regra de independência:** o validador de uma peça **nunca** pode ser da mesma família de quem a produziu.
  Skill forjada pelo Claude → validada por DeepSeek/Qwen/etc. (e vice-versa).

## Onde roda (obrigatório)
1. **Gate da forja** — toda skill forjada é refutada por ≥1 validador de outra família antes de sair de `rascunho`.
2. **Reviews/auditorias de segurança** — todo achado (ou a ausência de achado) passa por um validador que tenta refutar.
3. **Veredito Go/No-Go** — o `security-master` (Claude) decide, mas registra a devolutiva dos validadores.

## Como (operacional)
- Chaves em `.env` local (gitignored): `NVIDIA_API_KEY`, `OLLAMA_API_KEY`. Nunca commitadas.
- Se as chaves **não** estiverem presentes: **fail-closed** — a peça fica `rascunho` e o relatório diz
  "double-check indisponível (sem validador de outra família)". Nunca marque como validado sem o 2º par de olhos.
- A devolutiva do validador é **anexada** (em `security/reviews/`), com o modelo e a data. Se refutar, o Claude
  **resolve ou documenta** — não ignora.

## Modelos (atualize conforme o topo de linha muda)
| Papel | Família | Exemplos hoje | Via |
|---|---|---|---|
| Autoritativo | Claude | Fable / Opus | (nativo) |
| Validador | DeepSeek | deepseek-v3/r1 | NVIDIA NIM |
| Validador | Qwen | qwen-2.5/3 | NVIDIA NIM / Ollama |
| Validador | GLM | glm-4.x | Ollama Cloud |
| Validador | Kimi | kimi/moonshot | Ollama Cloud |
