# Playbook — Ataque a Login: detecção (blue) + teste (red)

> Fonte: Blue Team Lessons do **BruteForceAI** + o que validamos ao vivo na auditoria do Lunar
> (rate-limit por-IP corta ~75 req/s de fonte única; não há teto global; anti-XFF-spoof de pé).
> Usado por `runtime-detection` (detectar) e `qa-adversarial` (testar — **só autorizado**).

## Como um ataque moderno de login funciona (o que defender)
1. **Reconhecimento do form** — atacante (hoje, com LLM) lê o HTML do login e identifica os seletores/campos automaticamente.
2. **Ataque** — brute force (todas as combinações) ou **password spray** (uma senha contra muitos usuários — foge do lockout por-conta), multi-thread (100+), com **timing randomizado, jitter e rotação de user-agent** pra parecer humano.
3. **Validação de sucesso** — detecta mudança no DOM/redirect.
4. **Exfil** — resultado sai por webhook (Discord/Slack/Telegram).

## 🛡️ Detecção (regras para o `runtime-detection` / observabilidade)
- [ ] **N× 401 do mesmo IP** em janela curta → alerta. (No Lunar: 268 de 401 na borda sem ninguém saber.)
- [ ] **Falha de auth espalhada por muitos usuários** com **poucas senhas** = **password spray** → alerta (o lockout por-conta NÃO pega isso).
- [ ] **Timing bom demais** — intervalos perfeitamente randomizados / user-agent rotativo = bot, não humano.
- [ ] **Rajada em `/login`** vinda de IP único → o rate-limit deve cortar; **logar quando cortar** (429 é sinal, não silêncio).
- [ ] **Egress para plataformas de chat** (webhook Discord/Slack/Telegram) de dentro da app/host → possível exfil.
- [ ] **Parsing automatizado do form** — requests que buscam só o HTML do login repetidamente.

## 🛡️ Defesa (o que o baseline liga)
- **Rate-limit por IP** (borda) **+ inFlightReq global** (o backstop que o Lunar não tinha — contra spray distribuído).
- **Lockout/backoff por conta E por IP** — spray precisa do por-IP; brute precisa do por-conta.
- **Anti-XFF-spoof** — rate-limit e log no IP TCP real, ignorar headers forjados (validado no Lunar: STRONG).
- **2FA/step-up** (módulo `stepup-2fa`) — mata credential stuffing mesmo com senha certa.
- **CAPTCHA/PoW** após N falhas; **CDN/WAF** na frente para volumétrico distribuído.
- **Resposta de login uniforme** — mesmo tempo/mensagem para "usuário não existe" vs "senha errada" (anti-enumeração, igual ao 404-uniforme que provamos no Lunar).

## 🔴 Teste (playbook do `qa-adversarial` — SÓ com autorização escrita)
1. Confirme autorização do dono + escopo (como fizemos: Magdiel autorizou, contas de teste).
2. **Resistência a spray:** 1 senha × N usuários → o **por-IP** corta? o **por-conta** não deixou passar?
3. **Resistência a brute (1 conta):** o lockout/backoff dispara? em quantas tentativas?
4. **Anti-XFF-spoof:** varie XFF/X-Real-IP/CF-Connecting-IP aleatório **com controle na mesma vazão** (a armadilha: comparar ferramentas de vazão diferente gera falso-positivo — sempre o controle).
5. **Distribuído (multi-IP):** só mede o agregado que fura o por-IP; use IPv6 /64 **na sua infra** ou frota própria — **nunca** botnet/proxy. Contra prod é DoS.
6. **Enumeração:** "usuário não existe" difere de "senha errada"? (tempo, corpo, status) → vazamento.
7. Reporte: onde corta, onde quebra, recuperação — e **ancore ao nº de IPs de origem** (senão o número não é interpretável).
