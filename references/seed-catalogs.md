# Seed Catalogs — o ponto de partida da FORJA

> A forja **não pesquisa do zero**. Ela parte destes catálogos de referência — atuais, curados e mantidos pela comunidade — e **aprofunda + verifica** na web para a stack específica do projeto. Começar aterrado reduz alucinação e ganha velocidade.

## Protocolo da forja (research-first, verificado)
```
1. DETECTAR a stack real do projeto (linguagem, framework, deploy, auth, se tem IA/ML).
2. SEED   → abrir os catálogos abaixo pelo concern relevante.
3. APROFUNDAR → pesquisar na web o específico da stack (doc oficial > blog; versão atual).
4. COMPILAR → juntar com PROCEDÊNCIA (cada regra cita a fonte).
5. FORJAR  → escrever a skill em .claude/skills/<stack>-<concern>/SKILL.md (formato Soberis).
6. VETAR   → ai-agent-security (SkillSpector-style) + security-master + qa-adversarial red-teiam.
             Só então a skill é CONFIÁVEL. (Regra 12 — dupla validação.)
7. CACHE   → guardar a skill; re-pesquisar só sob pedido/refresh.
```

## Os catálogos-semente

### 1. Digital-Forensics-Guide — `github.com/mikeroyal/Digital-Forensics-Guide`
- **Semeia:** o agente `dfir-incident-response` e o lado de IR do `runtime-detection`.
- **O que puxar:** metodologia DFIR por domínio (memória/disco/rede/mobile/DB) + ferramentas de referência: **Volatility, Redline** (memória), **Autopsy, Sleuth Kit, FTK/EnCase** (disco), **Wireshark, NetworkMiner, Xplico** (rede), **Oxygen/XRY** (mobile). Trilhas de certificação (SANS, IACIS) como fonte de método.
- **Uso:** quando forjar a skill de resposta a incidente/forense, use isto como o esqueleto e adapte à plataforma real (Linux/Windows/container/cloud).

### 2. BruteForceAI — `github.com/MorDavid/BruteForceAI`
- **Semeia:** `runtime-detection` (regras de detecção) e `qa-adversarial` (playbook de teste de auth). Ver `security/playbooks/auth-attack-defense.md`.
- **O que puxar:** as **Blue Team Lessons** (detecção de parsing automatizado de login, timing randomizado = bot, padrão de spray, rate-limit que derruba multithread) + a **metodologia de ataque** (LLM lê o form → spray/brute via browser) para o red-team **autorizado** testar resistência de login.
- **⚠️ Uso ético:** ferramenta ofensiva — só em **teste autorizado** (autorização escrita do dono + escopo definido). O valor pra nós é **defensivo** + red-team com escopo.

### 3. awesome-ai-security-tools — `github.com/scadastrangelove/awesome-ai-security-tools`
- **Semeia:** o agente `ai-agent-security` (proteger a camada de agente) **e a própria forja** (catálogo de ferramentas por concern).
- **O que puxar:**
  - **Proteger o agente:** SkillSpector (vetar skills — usar no gate da forja), nono (sandbox de agente), Agent Governance Toolkit, garak/PyRIT (red-team de LLM), Guardrails AI/NeMo.
  - **IA pra segurança (a forja pode usar):** Vulnhuntr (achado de vuln por LLM call-chain), PentestGPT/PentAGI (pentest autônomo — para o qa-adversarial), modelscan (supply chain de modelo, se o projeto tiver ML).
- **Uso:** é o mapa "que ferramenta existe pra cada problema" — a forja consulta antes de reinventar, e o `ai-agent-security` usa o SkillSpector como gate.

## Como adicionar mais seeds
Colega tem um repo de referência bom? Adiciona aqui: **nome · URL · o que semeia · o que puxar · cuidado de uso**. Quanto melhor o seed, menos a forja alucina.
