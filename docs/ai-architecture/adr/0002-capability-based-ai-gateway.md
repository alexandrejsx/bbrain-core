# ADR-0002 — Gateway de IA orientado a capacidades

- Status: Proposta
- Data: 2026-07-20
- Escopo: seleção de provider/modelo, fallback e execução de IA

## Contexto

O domínio já depende de uma porta `ChatAgent`, mas a composição escolhe diretamente um adapter e o contrato retorna reply, risco, escopo e alteração de perfil. Novas tarefas têm requisitos distintos de qualidade, latência, volume, sensibilidade, raciocínio, contexto, ferramentas e custo. Espalhar nomes de modelos em casos de uso impediria avaliar, substituir e observar cada tarefa isoladamente.

## Decisão

Casos de uso solicitam uma capacidade lógica, não provider ou modelo concreto. Uma política versionada de aplicação/composição resolve a capacidade para uma configuração de infraestrutura e registra a decisão em `AiExecutionRecord`.

Capacidades iniciais incluem `CHAT_BALANCED`, `INTENT_FAST`, `EXTRACTOR_PRECISE`, `TEMPORAL_RESOLVER`, `MEMORY_CONSOLIDATOR`, `INSIGHT_REASONER`, `INSIGHT_VALIDATOR`, `RETRIEVAL_QUERY_REWRITER`, `GROUNDED_SYNTHESIZER`, `CITATION_VERIFIER` e `SAFETY_CLASSIFIER`. A lista de tarefas e seus budgets permanece na [política das 13 tarefas](../ai-task-policy.md).

Cada execução carrega, no mínimo, `executionId`, tarefa/capacidade, `policyVersion`, provider, modelo, `promptVersion`, `schemaVersion`, timeout, tentativa, tokens, duração e resultado sanitizado. Outputs estruturados passam por validação de schema e domínio. Fallback ocorre somente entre opções previamente avaliadas para a mesma tarefa; safety e escrita de fato pessoal falham fechadas.

Nomes concretos de modelos ficam em configuração de infraestrutura. Nenhuma troca é promovida sem eval reproduzível, custo observado e canary. O adapter OpenAI atual pode continuar usando Responses API. O adapter Gemini deve ficar atrás desta boundary para permitir migrar de GenerateContent `v1beta` para Interactions `v1` sem alterar domínio ou casos de uso.

## Alternativas consideradas

- Um único modelo para tudo: simples, mas força a mesma relação qualidade/custo/latência em tarefas incompatíveis.
- Roteamento livre decidido por LLM: flexível, mas não oferece orçamento e comportamento determinísticos.
- Referenciar provider/modelo no domínio: reduz abstrações no curto prazo e cria acoplamento técnico e migrações difusas.
- Trocar modelos agora por ranking público: rejeitado; benchmarks genéricos não demonstram qualidade no dataset BBrain.

## Consequências

- Provedores e modelos podem evoluir sem alterar regras de produto.
- Custo, latência e falha passam a ser atribuíveis por tarefa e versão.
- Há uma camada adicional de configuração, registry e testes de contrato.
- Fallback não garante equivalência; cada par precisa de eval próprio.

## Gates e rollback

Uma policy version só recebe tráfego após contratos dos adapters, eval offline, limite de custo e canary. Rollback reponta o alias para a versão anterior; dados gravados carregam versões e não são reinterpretados silenciosamente. Circuit breaker impede cascata sem limite.

## Evidência e fontes

Ver [estado atual](../current-state.md) e [política das tarefas](../ai-task-policy.md). Referências oficiais: [OpenAI Responses API](https://developers.openai.com/api/docs/guides/migrate-to-responses), [controles de dados OpenAI](https://developers.openai.com/api/docs/guides/your-data) e [versões da Gemini API](https://ai.google.dev/gemini-api/docs/api-versions).
