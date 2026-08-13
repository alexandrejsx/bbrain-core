# ADR-0001 — Captura determinística pós-resposta

A resposta de chat é entregue primeiro. A captura de Humor e Sono é iniciada depois, sem bloquear a
experiência. Ela usa a mensagem atual em memória, parser e policies de domínio; resultados válidos
são persistidos de forma idempotente e atualizam projeções aplicáveis.

Falhas não criam dados parciais nem afetam a resposta. O fluxo não persiste transcrições literais.
