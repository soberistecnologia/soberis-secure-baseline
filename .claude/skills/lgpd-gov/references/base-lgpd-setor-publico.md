# Base de Conhecimento — LGPD no Setor Público (Projeto Lunar)

> Compilação da pesquisa profunda (deep research, 5 ângulos → 25 fontes → 113 claims → verificação adversarial 3-votos → 23 confirmados). **Data:** 2026-07-21.
> **Marcadores:** ✅ = verificado 3-0 (fonte primária); ⚠️ = caveat/atenção; 🔲 = escopo NÃO verificado ainda (precisa 2ª rodada antes de virar afirmação de conformidade).
> **Fontes primárias-chave:** Lei 13.709/2018 (Planalto, texto compilado); Guia Orientativo ANPD — Tratamento de Dados pelo Poder Público; Resolução CD/ANPD nº 15/2024 (incidentes); Resolução CD/ANPD nº 4/2023 (dosimetria); Resolução CD/ANPD nº 18/2024 (encarregado); Decreto 10.222/2020 (E-Ciber); Decreto 9.637/2018 (PNSI); IN GSI/PR nº 1/2020.

---

## 1. Fundamentos (art. 5 e 6)

- **Definições** (art. 5): dado pessoal (identifica ou torna identificável pessoa natural); dado pessoal **sensível** (origem racial/étnica, convicção religiosa, opinião política, saúde, vida/orientação sexual, dado genético/biométrico); **tratamento** (toda operação — coleta, uso, acesso, armazenamento, eliminação…); **controlador** (decide sobre o tratamento); **operador** (trata em nome do controlador); **encarregado/DPO** (canal com titulares e ANPD).
- ✅ **Princípios (art. 6)** regem TODO tratamento: **finalidade, adequação, necessidade (minimização)**, livre acesso, qualidade, transparência, segurança, prevenção, não discriminação, responsabilização e prestação de contas. **Necessidade** = limitação ao mínimo necessário; dados pertinentes, proporcionais e **não excessivos**. *(Fonte: Planalto, texto da lei.)*

## 2. Bases legais no setor público (arts. 7, 11, 23-30)

- ✅ Tratamento pelo Poder Público exige base legal do **art. 7** (ou **art. 11** p/ sensíveis) **interpretada sistematicamente com o art. 23 e o Capítulo IV**. O **art. 23** condiciona o tratamento a: atendimento da **finalidade pública**, persecução do **interesse público**, execução de **competências/atribuições legais**, com **informação clara** sobre previsão legal, finalidade e procedimentos. *(Planalto + Guia ANPD Poder Público.)*
- ✅ **Consentimento em regra NÃO é a base apropriada no setor público** quando o tratamento é necessário ao cumprimento de obrigações/atribuições legais (relação de desbalanceamento de forças). Base **preferencial**: **art. 7º, II** (cumprimento de obrigação legal/regulatória) ou **art. 7º, III** (execução de políticas públicas com uso compartilhado). O art. 7º, III aplica-se a órgãos dos três Poderes no exercício de **função administrativa**. Consentimento só em uso **não-compulsório**. *(Guia ANPD Poder Público.)*
- ✅ **Uso compartilhado** de dados pelo Poder Público deve ser **formalizado e registrado** (art. 37); recomenda-se **processo administrativo** com análise técnica e jurídica que motive o compartilhamento. *(Guia ANPD + art. 37.)*
- ✅ Para transparência ativa (LAI), a base do **art. 7º, II** fundamenta o tratamento/disponibilização de dados em atendimento à Lei de Acesso à Informação. *(Revista CGU.)*

## 3. Registro das operações de tratamento (art. 37 → ROPA)

- ✅ **Art. 37** obriga controlador e operador a **manter registro das operações de tratamento** de dados pessoais (especialmente quando baseado no legítimo interesse). É a **base legal do inventário/registro de operações (ROPA)** — controle técnico verificável no sistema. *(Planalto.)*

## 4. Segurança e governança (arts. 46-51) 🔲 parcial → **RIPD/frameworks resolvidos na Rodada 2 (R2.4, R2.5)**

- 🔲 **Art. 46** (medidas técnicas e administrativas de segurança) e **art. 38** (RIPD/Relatório de Impacto) e **arts. 50-51** (programa de governança em privacidade) — **não tiveram claim verificado nesta rodada**; tratar como diretriz a aprofundar. Fontes primárias já localizadas: Guia de Segurança da Informação da ANPD (medidas do art. 46, com checklist) e página oficial do RIPD (ANPD). *(Usar na 2ª rodada.)*

## 5. Incidentes de segurança (art. 48 + Resolução CD/ANPD nº 15/2024)

- ✅ **Art. 48**: o controlador deve **comunicar à ANPD e ao titular** incidente que possa acarretar **risco ou dano relevante**, com conteúdo mínimo (natureza dos dados, titulares envolvidos, medidas técnicas, riscos, medidas adotadas — incisos I a V). *(Planalto.)*
- ✅ **Resolução CD/ANPD nº 15, de 24/04/2024** (Regulamento de Comunicação de Incidente de Segurança) regulamenta o art. 48:
  - **Prazo: 3 dias úteis** contados do conhecimento de que o incidente afetou dados pessoais (**art. 6** da Resolução; ressalvado prazo em legislação específica). **6 dias úteis** para **agentes de pequeno porte**.
  - Cabe **comunicação preliminar** com **complementação em até 20 dias úteis**.
  - Obrigação recai sobre o **controlador** (pode ser feita pelo encarregado/representante); o **operador** deve informar o controlador **sem demora injustificada**.
  - **Registro do incidente guardado por 5 anos**.
  - ⚠️ **Citação correta:** prazo à ANPD = **art. 6**; comunicação ao titular = **art. 9**; **art. 5** define o **limiar** de "risco ou dano relevante" (não o prazo). Cobertura secundária erra atribuindo o prazo ao art. 5.
- ✅ **Gatilho da notificação (art. 5 da Res. 15/2024):** incidente que pode **afetar significativamente** interesses e direitos fundamentais **E**, cumulativamente, envolve **pelo menos um** de seis critérios: (1) dados **sensíveis**; (2) dados de **crianças/adolescentes/idosos**; (3) dados **financeiros**; (4) dados de **autenticação** em sistemas; (5) dados protegidos por **sigilo** legal/judicial/profissional; (6) dados em **larga escala**. Exige ocorrência confirmada. **Criptografia é fator de avaliação, não isenção automática.**
- ✅ **Comunicação à ANPD: 12 itens** (natureza/categoria dos dados; nº de titulares — distinguindo crianças/adolescentes/idosos; medidas técnicas de proteção; riscos/impactos; motivo de eventual demora; medidas de mitigação; data da ocorrência e do conhecimento; dados do encarregado; identificação do controlador e se é pequeno porte; identificação do operador; descrição/causa do incidente; total de titulares) via **formulário eletrônico da ANPD**. **Comunicação ao titular:** linguagem simples, direta e individualizada quando possível (telefone, e-mail, mensagem, carta); se inviável, divulgação em site/apps/redes por **no mínimo 3 meses**. *(Res. 15/2024, arts. 6 §2 e 9.)*

## 6. Retenção e eliminação (arts. 15-16) vs. LAI e guarda pública 🔲 → **RESOLVIDO na Rodada 2 (R2.1, R2.2)**

- ✅ **Não há hierarquia** entre proteção de dados (LGPD) e transparência ativa (LAI) — são direitos fundamentais que se **harmonizam** (Capítulo IV da LGPD), não um "conflito real". A ponderação é **caso a caso** pela autoridade administrativa/judicial. *(ConJur; FGV Direito.)*
- ⚠️ Risco concreto de órgãos usarem a LGPD como **pretexto para reduzir transparência** — evitar "descaracterização" (redação de CPF etc.) automática sem base legal de sigilo. *(Ronny Charles / fiquemsabendo.)*
- 🔲 **Arts. 15-16 (término do tratamento e eliminação)** e o **conflito concreto com prazos de guarda contábil/TCU e tabelas de temporalidade de arquivo** — **não verificado** nesta rodada. **Diretriz operacional:** dado pessoal que integra registro público/contábil sujeito a prazo legal de guarda **não é eliminado** enquanto durar a obrigação de retenção; a eliminação segue a **tabela de temporalidade** do órgão. **Aprofundar na 2ª rodada.**

## 7. Sanções (art. 52) e regulamentos ANPD

- ✅ **Art. 52**: sanções administrativas pela ANPD (advertência, multa, publicização, bloqueio, eliminação…). Multa simples de **até 2% do faturamento no Brasil**, limitada a **R$ 50 milhões por infração**. **Dosimetria: Resolução CD/ANPD nº 4, de 24/02/2023.** *(Planalto + índice ANPD.)*
- ⚠️ **SETOR PÚBLICO:** o cap de 2%/R$50M refere-se a **pessoa jurídica de direito PRIVADO**. Para **entes públicos**, o **art. 52 §3** restringe as sanções (a **multa em geral não incide** contra órgãos públicos, restando **advertência, publicização, bloqueio/eliminação de dados**). **A skill NÃO deve prometer imunidade** — a **responsabilização de agentes públicos** e a ação da ANPD permanecem.
- ✅ **Resolução CD/ANPD nº 18, de 16/07/2024**: regula a **atuação do encarregado** pelo tratamento de dados pessoais. *(Secundária confiável — reverificar no gov.br para citação final.)*

## 8. Frameworks de segurança do governo federal (cruzamento com LGPD)

- ✅ **Decreto 10.222/2020** aprova a **E-Ciber** (Estratégia Nacional de Segurança Cibernética), módulo que implementa a **PNSI (Decreto 9.637/2018)**, com validade **2020-2023**. Cabe aos órgãos da APF implementar as ações. ⚠️ **Quadriênio já expirou** — verificar estratégia/atualização vigente antes de citar como plano corrente.
- ⚠️ **REFUTADO (voto 1-2), NÃO afirmar:** que o art. 2 do Decreto 9.637/2018 e a IN GSI/PR nº 1/2020 vinculem **explicitamente** segurança cibernética à "proteção de dados"/LGPD com "cinco dimensões". O vínculo textual não foi confirmado — não usar sem reverificar o texto normativo.
- 🔲 IN GSI/PR nº 1/2020 (estrutura de gestão de SI na APF), **e-PING**, **ISO/IEC 27001 e 27701** — mencionados no escopo mas **sem claim verificado**; tratar como referência a confirmar.

## 9. Direitos do titular (arts. 17-22) 🔲 → **RESOLVIDO na Rodada 2 (R2.3)**

- 🔲 Arts. 17-22 (acesso, correção, anonimização/bloqueio/eliminação, portabilidade, informação sobre compartilhamento, revisão de decisões automatizadas) **não verificados em detalhe operacional** nesta rodada. Base já mapeada (texto da lei) — **aprofundar operacionalização na 2ª rodada.**

---

---

# RODADA 2 — Lacunas fechadas (verificado 2026-07-21, 25/25 claims, 0 refutados)

> Atualiza as seções 4, 6 e 9 acima (que estavam 🔲). Fontes primárias ANPD/Planalto/GSI. Substitui as diretrizes provisórias pelos fatos verificados.

## R2.1 Retenção e eliminação (arts. 15-16) — ✅ RESOLVIDO

- ✅ **Regra operacional (Guia ANPD Poder Público, verbatim):** *"o tratamento de dados pessoais é um processo com duração definida, após o qual, em regra, os dados pessoais devem ser eliminados, observados as condições e os prazos previstos em normas específicas que regem a gestão de documentos e arquivos."* Ou seja: **eliminar é a regra**, mas condicionada às **tabelas de temporalidade / CONARQ (Lei 8.159/1991)**.
- ✅ **Hipóteses de conservação (art. 16), que autorizam reter:** (I) **cumprimento de obrigação legal/regulatória** — é o que ancora a **guarda contábil/TCU/Lei 4.320/1964** e a prestação de contas; (II) estudo por órgão de pesquisa (anonimizado quando possível); (III) transferência a terceiro (respeitados requisitos); (IV) **uso exclusivo do controlador, vedado acesso por terceiro, desde que anonimizados**.
- **Regra prática Lunar:** dado sob prazo legal de guarda → **retido via soft-delete/temporalidade** (art. 16, I); fim do prazo sem outra base → **eliminar** ou **anonimizar** (art. 16, IV).

## R2.2 LGPD × LAI (transparência) — ✅ RESOLVIDO

- ✅ **Publicidade é a regra, sigilo é exceção** (LAI 12.527/2011); a divulgação de dado pessoal ainda observa os **princípios da LGPD**.
- ✅ **Instâncias recursais de negativa de acesso:** **CGU** (3ª instância, responde em 5 dias) e **CMRI** — Comissão Mista de Reavaliação de Informações (instância final) — Dec. 7.724/2012, arts. 23-24.
- ✅ No setor público o exercício de direitos observa também **Habeas Data** (Lei 9.507/1997), **Lei do Processo Administrativo** (Lei 9.784/1999) e **LAI** — por força do **art. 23 §3 da LGPD**.

## R2.3 Direitos do titular (arts. 17-22) — ✅ RESOLVIDO

- ✅ **Prazos (art. 19):** acesso/confirmação em **formato simplificado, imediatamente** (I); **declaração clara e completa** (origem, critérios, finalidade) em **até 15 dias** (II).
- ✅ **Art. 18 §4:** se a providência imediata for impossível, o controlador responde justificando (razões de fato/direito) ou indicando o agente correto.
- ✅ **Art. 18, IV e VI:** titular pode pedir **anonimização/bloqueio/eliminação** de dados desnecessários/excessivos/tratados em desconformidade, e **eliminação** dos tratados por consentimento — **ressalvadas as hipóteses do art. 16**.
- ✅ **Art. 20:** direito a **revisão de decisão automatizada** que afete interesses + explicação de critérios. ⚠️ **A exigência de revisão "por pessoa natural" foi VETADA** (Lei 13.853/2019) → a LGPD **não** exige revisão humana obrigatória. Decisões automatizadas são **prioridade de fiscalização da ANPD 2026-2027**.

## R2.4 RIPD (art. 38) — ✅ RESOLVIDO

- ✅ **Quando fazer:** operações que possam gerar **alto risco** (base art. 5º, XVII + art. 38); **especialmente esperado no Poder Público** (art. 32) e com **dados sensíveis**. Recomenda-se elaborar **antes** de iniciar o tratamento. Critérios de alto risco: grande escala **ou** afetação significativa de direitos + (tecnologias emergentes, vigilância de zonas públicas, decisões automatizadas, dados sensíveis/de crianças/idosos).
- ✅ **Conteúdo mínimo (art. 38, par. único):** descrição dos tipos de dados, metodologia de coleta e garantia de segurança, e análise do controlador sobre medidas, salvaguardas e mecanismos de mitigação.
- ✅ **Sem template obrigatório da ANPD** — controlador tem flexibilidade de estrutura. (Há modelos **não-obrigatórios** do PPSI/MGI e da FGV.)

## R2.5 Framework de cibersegurança VIGENTE — ✅ ATUALIZADO

- ✅ **Nova E-Ciber: Decreto nº 12.573, de 04/08/2025** — sucede a E-Ciber de 2020 (expirada). É o **instrumento executório da PNCiber** (Política Nacional de Cibersegurança, **Decreto 11.856/2023**): *"os objetivos da PNCiber... serão alcançados por meio da E-Ciber"* (art. 1º §1º). **4 eixos** (cidadão; resiliência de serviços essenciais/infra crítica; cooperação público-privada; soberania/governança). Governança: **GSI/PR + Comitê Nacional de Cibersegurança (CNCiber)**; **Plano Nacional de Cibersegurança (P-Ciber)** proposto pelo CNCiber.
- ✅ **IN GSI/PR nº 1/2020** (vigente, consolidada com IN 09/2026): **POSIC obrigatória** em todo órgão da APF, aprovada pela autoridade máxima (art. 9º); **9 diretrizes mínimas** (art. 12, IV) — inclui **Controles de Acesso, Gestão de Incidentes, Auditoria e Conformidade**; **revisão ≤ 4 anos** (art. 12 §1º). **Ponto de conexão com a LGPD (art. 19, X):** o **Gestor de SI coopera com o Encarregado (DPO)** quando o tema envolver dados pessoais.

## R2.6 Dados sensíveis (art. 11) + anonimização/pseudonimização — ✅ RESOLVIDO

- ✅ **Dados sensíveis de servidor** (saúde em licença, biometria de ponto, filiação sindical) exigem **base do art. 11** (não do art. 7) + art. 23. **Art. 11, II, "b"** (políticas públicas) é **MAIS RESTRITA** que o art. 7º, III: limitada a políticas **"previstas em leis e regulamentos"** — **não** abrange contratos/convênios. Cessada a finalidade **sem obrigação legal de guarda → eliminar**.
- ✅ **Anonimização (art. 12):** dado anonimizado **não é dado pessoal** e **sai do escopo da LGPD** — salvo se reversível com meios próprios ou esforços razoáveis (fatores: custo, tempo, tecnologia). Retenção para uso exclusivo do controlador só se **anonimizado** (art. 16, IV).
- ✅ **Pseudonimização (art. 13 §4º):** **permanece dado pessoal** (pode ser re-associado via informação adicional mantida separada em ambiente controlado) — é **medida de segurança**, não saída do escopo.

---

## Lacunas ainda abertas (3ª rodada, se necessário)

1. 🔲 **Checklist concreto do art. 46** (Guia de Segurança da ANPD): lista item-a-item de medidas técnicas (cripto, controle de acesso, backup, logs, incidentes). *(Fonte já localizada: Guia de Segurança para Agentes de Pequeno Porte — buscar direto.)*
2. 🔲 **Prazos numéricos** de guarda contábil/TCU (Lei 4.320) e **tabelas de temporalidade CONARQ** por tipo documental (financeiro/RH) — para parametrizar a retenção do Lunar.
3. 🔲 **P-Ciber** (Plano Nacional de Cibersegurança) detalhado — se impõe controles operacionais adicionais.
4. 🔲 **Crosswalk ISO 27001:2022 / 27701** × IN GSI 1/2020 × art. 46 — correspondência formal.

## Fontes (qualidade)

**Primárias:** Planalto (Lei 13.709/2018; Decretos 10.222/2020, 9.637/2018); ANPD — Guia Poder Público, Res. 15/2024, índice de regulamentações, canal CIS, RIPD, Guia de Segurança para pequeno porte; IN01 consolidada (GSI); Política Nacional de SI (gov digital); PGP-Enap.
**Secundárias:** ConJur (LGPD×LAI), Revista CGU, FGV Direito Administrativo, ITS Rio, Ronny Charles, fiquemsabendo, notícia ANPD dosimetria, Anoreg (Res. 18/2024).
