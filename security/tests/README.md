# tests/

⭐ **Testes de SEGURANÇA — suíte SEPARADA da app.**

Rodam em alvo/config próprio, nunca misturados à suíte de testes da aplicação. Ex.: testes adversariais de RBAC/IDOR, resistência de auth, anti-XFF-spoof, verificação de headers. Assim o resultado de segurança é independente e não some no meio dos testes de feature.
