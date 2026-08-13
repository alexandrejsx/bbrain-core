# Estratégia de evals

O harness executa datasets sintéticos contra parser e policies reais para detectar regressões de
extração. Ele mede qualidade, rejeições, latência e uso de tokens sem guardar conteúdo pessoal.

Contract tests verificam requests, structured outputs, timeout, recusa, respostas incompletas,
autenticação e indisponibilidade dos providers. Chamadas reais são opt-in e usam apenas
conteúdo sintético.

Os gates do harness são critérios de desenvolvimento: ajudam a responder se uma mudança melhorou,
manteve ou piorou a qualidade. Eles não alteram a execução da aplicação.
