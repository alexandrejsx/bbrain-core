# Segurança e observabilidade

Dados emocionais, de Humor e de Sono são sensíveis. O backend aplica autenticação, ownership,
consentimento, HMAC por finalidade, idempotência e respostas de erro seguras.

Logs não incluem mensagens, respostas, prompts renderizados, cotações de evidência, perfis ou
segredos. A telemetria registra somente provider, modelo, duração, tokens, contagens agregadas e
tipo de falha sanitizado para diagnosticar problemas reais.

Uma extração falha sem criar write parcial e sem afetar o chat. A persistência revalida o
consentimento imediatamente antes das mutações.
