# Contexto e prompts

O caminho ativo de conversa não lê nem escreve transcrições literais. `ConversationState` é
estruturado, minimizado, consentido e possui TTL. Revogar memória ou armazenamento sensível impede
leituras/escritas e remove o estado ativo.

Prompts e schemas são versionados. Alterações exigem evals e contract tests antes de serem usadas.
Conteúdo da mensagem atual é transitório; evidência de observação é persistida somente como HMAC.
