# ADR-0002 — Providers de IA isolados

Providers de chat e extração ficam atrás de ports e adapters para que OpenAI e Gemini não contaminem
domínio ou casos de uso. Requests usam contratos estruturados quando necessários; parser e policies
são a fronteira de confiança antes de qualquer efeito no produto.
