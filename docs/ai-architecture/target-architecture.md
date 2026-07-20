# Arquitetura-alvo evolutiva

## Estado implementado do primeiro slice

A direção abaixo foi concretizada parcialmente num bounded context `wellbeing-history`: port de extração estruturada, adapters OpenAI/Gemini/noop, router primary/fallback, schema/prompt/parser v3, validação de domínio, aggregate/repository Mongo, CRUD autenticado, captura pós-resposta e projeção diária. A conversa continua síncrona e ganhou idempotência por exchange; a captura fica fora do retorno HTTP. Isso ainda não é o `AiExecutionGateway` universal descrito como alvo, nem um executor durável: coordenação, retries e purge são locais ao processo.

Insights ganhou somente a fronteira de autorização Pro e um read model `insufficient_data`; Resources/RAG, memória tipada, telemetry de execução e geração longitudinal continuam futuros. A captura automática não deve ser promovida sem shadow separado de persistência, eval de provider, Mongo integration e reconciliação.

## Objetivos e não objetivos

A arquitetura-alvo sustenta conversa, extração de Humor/Sono, controle do usuário, Insights e Recursos sem transformar o BBrain em serviço clínico. Ela preserva DDD/Clean Architecture existentes, uma identidade conversacional e os contratos visuais atuais.

Não são objetivos do MVP:

- check-in obrigatório, questionário ou wizard;
- integração com Apple Health, smartwatch, OpenClaw, Hermes ou outra fonte externa;
- multiagente, votação, blackboard, GraphRAG ou knowledge graph;
- envio indiscriminado de perfil, histórico, livros ou documentos;
- diagnóstico, prescrição ou decisão clínica;
- redesign do frontend.

## Separações conceituais propostas

As separações abaixo são boundaries, não uma ordem para criar uma pasta ou microserviço para cada nome.

| Capacidade | Papel predominante | Dono da regra | Estado inicial |
|---|---|---|---|
| Conversa | use case síncrono + capability de geração | application/domain policies | evoluir fluxo existente |
| Política de IA | port + policy de aplicação + adapters | application/composition | nova boundary mínima |
| Perfil/Preferências | aggregate/policies revisáveis | Users/Profile | separar leituras sem migração destrutiva |
| Memória | fatos duráveis com proveniência e retenção | Memory | boundary agora; persistência depois de consentimento/evals |
| Humor | eventos primários + projeção diária | Wellbeing/Mood | primeiro slice funcional |
| Sono | registros parciais + relatos de período | Wellbeing/Sleep | primeiro slice funcional |
| Insights | projeção longitudinal premium | Insights/Pattern Analysis | pós-MVP, boundary preparada |
| Recursos | knowledge retrieval sem memória pessoal | Resources/Knowledge | fase própria |
| Segurança/Governança | policies transversais, sem regra escondida no prompt | application/infrastructure | obrigatória desde o primeiro slice |

Humor e Sono podem começar dentro de um contexto `wellbeing-history` coeso, caso o repositório prefira menos contexts. Eles precisam manter tipos e invariantes distintos, mesmo compartilhando provenance, revisão e invalidation.

## Componentes alvo

```text
HTTP controllers / future job handlers
        |
        v
Application use cases
  - SendChatMessage
  - ExtractWellbeingFromUserMessage
  - Create/Edit/DeleteMoodEvent
  - Create/Edit/DeleteSleepRecord
  - RebuildDailyMoodSummary
  - Generate/ValidateInsight
  - AnswerResourceQuestion
        |
        +--> domain entities/policies/specifications
        |      - provenance and ownership
        |      - temporal/subject/negation validation
        |      - revision and invalidation
        |      - entitlement and privacy
        |
        +--> AI capability ports
        |      - ConversationGeneration
        |      - StructuredExtraction
        |      - GroundedSynthesis
        |      - SafetyClassification
        |
        +--> repository/query/event/job ports
               |
               v
Infrastructure adapters
  - OpenAI / Gemini / Mock
  - Mongo repositories + mappers
  - in-process post-response executor, then queue if required
  - retrieval adapters and isolated indexes
  - sanitized telemetry
```

`AiExecutionPolicy` resolve uma capacidade lógica para provider/model/configuração. Domínio não conhece provider, modelo, prompt ou JSON Schema técnico. Um `AiExecutionRecord` registra política, provider, modelo, prompt, schema, context bundle/eventual skill, tokens, duração e resultado sanitizado; ele não armazena texto integral por padrão.

## Fluxo de Conversa

```text
request autenticado
  -> validar DTO, ownership, rate limit e privacy policy
  -> montar contexto mínimo e tipado
  -> policy resolve CHAT_BALANCED
  -> gerar output estruturado de conversa
  -> validar schema + regras de escopo/segurança
  -> persistir exchange idempotente
  -> registrar usage/telemetria sanitizada
  -> responder ao frontend
  -> passar a mensagem atual apenas ao executor local em memória; envelope durável futuro contém ids, nunca texto integral
       -> workflow de Humor/Sono recarrega somente a mensagem do usuário
```

Decisões:

- O reply não espera Insights, consolidação de memória ou resumo diário.
- Classificação de segurança pode ter uma camada determinística antes/depois do modelo; risco alto nunca depende apenas de texto livre.
- A atualização do perfil atual deve ser desativada, reduzida ou executada por workflow separado antes de se tornar fonte de fatos pessoais.
- Streaming é `CONDITIONAL_ON_EVALS`: adotar quando p95 de first-token/tempo percebido justificar e o contrato do frontend aceitar sem redesign. Structured output completo continua necessário para etapas internas.

## Fluxo de Humor

```text
UserMessageStored(messageId, userId, conversationId)
  -> idempotency key = messageId + extractorVersion + schemaVersion
  -> eligibility policy
       somente role=user; consentimento; escopo pessoal; não ficção/hipótese/terceiro
  -> StructuredExtraction(MOOD_SLEEP_EXTRACTOR_PRECISE)
  -> schema validation
  -> deterministic domain validation
       sujeito, negação, temporalidade, campos explícitos, duplicação
  -> persist MoodEvent(s) com EvidenceRef e revision
  -> mark DailyMoodSummary(date,user) stale
  -> rebuild projection quando apropriado
  -> emitir telemetria sem conteúdo
```

Edição/exclusão:

```text
comando autenticado + expectedRevision
  -> ownership + privacy + validação
  -> append revision/tombstone; nunca sobrescrever evidência sem trilha
  -> invalidar resumo diário e Insights dependentes
  -> recalcular sob demanda/job
```

O resumo diário é uma projeção, não o evento primário. Uma edição manual e um resumo explícito do usuário prevalecem sobre derivação. Sem eventos suficientes, o estado é `insufficient_data`, não `neutral`.

## Fluxo de Sono

```text
mensagem do usuário
  -> eligibility + extractor estruturado
  -> temporal resolver usa timezone do perfil apenas como contexto
  -> classificar referência:
       specific_night | moment | interval | period_statement | goal_or_hypothesis
  -> rejeitar goal/hypothesis/third_party
  -> validar somente campos explicitamente sustentados
  -> persist SleepRecord parcial OU SleepPeriodStatement
  -> revision/invalidation em correção
```

`SleepRecord` não exige duração, qualidade, hora de dormir, hora de acordar, despertares ou sensação. Valores têm precisão `exact | approximate | range | unknown`. Relato “nas últimas semanas” gera um statement de período, nunca várias noites artificiais.

## Fluxo de Insights

```text
request ou job autorizado
  -> backend entitlement PRO + feature flag + privacy policy
  -> selecionar eventos ativos e projeções válidas em janela explícita
  -> calcular cobertura e agregações determinísticas
  -> gate de evidência mínima
  -> gerar candidate Insight com INSIGHT_REASONER
  -> validar com regras determinísticas + INSIGHT_VALIDATOR se eval justificar
  -> persistir evidências, versões, limitações e status
  -> servir apenas se todas as referências continuam ativas
```

Um Insight declara período, cobertura, evidências e limitações. Nunca diagnostica nem afirma causalidade. Alteração/deleção de evidência marca o Insight `stale` antes de qualquer exposição. A geração é inicialmente sob demanda/cacheada ou pós-resposta, não no caminho crítico da conversa.

## Fluxo de Recursos

```text
pergunta classificada como factual/Recursos
  -> não executar update de perfil/memória pessoal
  -> policy seleciona corpus e filtros de confiança
  -> busca lexical/semântica (híbrida somente se retrieval eval vencer)
  -> rerank opcional condicionado a eval
  -> validar autoridade, jurisdição, versão, data, seção e permissão
  -> montar evidence pack não executável
  -> síntese fundamentada
  -> verificação claim -> citation -> trecho
  -> resposta com limitações e fontes
```

Trust domains fisicamente separados:

```text
official_resources_index
editorial_book_derivatives_index
internal_product_content_index
personal_user_data (nunca corpus de Recursos)
```

Material recuperado é dado não confiável, não instrução. Prompt injection indireta é neutralizada por delimitação, allowlist de ferramentas, filtros e validação de citações.

## Síncrono, pós-resposta e assíncrono

| Etapa | Decisão inicial | Por quê | Gatilho para fila durável |
|---|---|---|---|
| reply conversacional | síncrono | experiência principal | não mover enquanto interação for request/response |
| safety pre/post check | síncrono | pode mudar resposta | manter orçamento estrito; circuit breaker local |
| extração Humor/Sono | pós-resposta, executor local idempotente em uma réplica | menor complexidade no primeiro slice | mais de uma réplica, perda/reprocessamento observado, backlog > 1 min ou necessidade de retry durável |
| resumo diário | sob demanda ou job idempotente | derivado e barato | volume/latência exigir batch |
| Insights | sob demanda cacheado ou job | caro, premium, não crítico ao reply | p95 > SLA, batch periódico aprovado ou volume sustentado |
| ingestion Recursos | job administrado | versionamento e revisão | desde o início pode ser CLI/job; fila só com ingestão contínua |

Antes de fila, o executor deve expor o mesmo `JobPort`, idempotency key e envelope que uma implementação distribuída usará. Isso prepara a boundary sem operar Redis/BullMQ prematuramente.

## Consistência e idempotência

- Cada mensagem recebe identificador estável; exchange não pode ser anexado duas vezes.
- Cada execução de IA recebe `executionId`, `task`, `policyVersion`, `schemaVersion`, `promptVersion` e `inputFingerprint` não reversível.
- Writes de extração usam unique key por mensagem/versões e transaction/atomic upsert quando disponível.
- Evento derivado nunca é gravado diretamente pelo adapter.
- Inbox marca job recebido antes de efeitos; outbox mínima publica invalidações após commit quando houver consumidores assíncronos.
- Reprocessamento cria revisão quando a versão mudou; não substitui silenciosamente decisão manual.

## Model routing, fallback e custo

O domínio usa capabilities lógicas. Composition/config resolve, por exemplo:

```text
CHAT_BALANCED
INTENT_FAST
EXTRACTOR_PRECISE
TEMPORAL_RESOLVER
MEMORY_CONSOLIDATOR
INSIGHT_REASONER
INSIGHT_VALIDATOR
RETRIEVAL_QUERY_REWRITER
GROUNDED_SYNTHESIZER
CITATION_VERIFIER
SAFETY_CLASSIFIER
```

Fallback só é permitido entre modelos que passaram o mesmo eval e suportam o mesmo schema. Para extração de fatos pessoais, indisponibilidade resulta em retry/fila ou ausência de evento, não degradação para parser não validado. Para conversa, fallback pode retornar resposta segura reduzida. Limites detalhados estão em [ai-task-policy.md](./ai-task-policy.md).

## Feature flags mínimas

- `wellbeing_extraction_shadow`
- `wellbeing_auto_persist`
- `wellbeing_manual_crud`
- `daily_mood_projection`
- `insights_generation`
- `resources_retrieval`
- `ai_policy_<version>` para rollout controlado

Flags são avaliadas no backend e não substituem entitlement, consentimento ou autorização.

## Decisões adiadas com gatilhos objetivos

| Conceito | Agora | Reavaliar quando |
|---|---|---|
| fila distribuída | evitar inicialmente | múltiplas réplicas ou >0,1% jobs perdidos/duplicados, backlog sustentado, retry durável obrigatório |
| streaming | condicional | p95 do reply exceder meta e teste de UX demonstrar ganho sem quebrar validação |
| router aprendido de modelos | evitar | três ou mais modelos elegíveis e roteamento estático perder custo/qualidade em eval controlado |
| reranker LLM | condicional | retrieval eval mostrar ganho relevante sobre hybrid/filtering que compense latência/custo |
| query rewriting | condicional | queries ambíguas/coloquiais reduzirem Recall@k abaixo do gate |
| vector store pessoal | evitar | busca lexical/estruturada de memória falhar em caso de uso aprovado, com consentimento e threat model |
| GraphRAG/knowledge graph | evitar | corpus oficial grande e consultas multi-hop comprovadamente falharem em retrieval evals |
| agentic RAG | evitar | workflow determinístico não resolver decomposição multi-etapa e eval mostrar ganho líquido |
| multiagente/debate | evitar | eval cego demonstrar ganho de segurança/faithfulness superior ao aumento de custo e superfície de falha |

## Estado final pretendido do primeiro slice

```text
mensagem de usuário armazenada
 -> elegibilidade
 -> extração estruturada em shadow/flag
 -> validação determinística
 -> persistência idempotente com provenance
 -> GET de eventos/records
 -> criação, edição e exclusão manual
 -> invalidação/rebuild de resumo diário
 -> testes, evals e telemetria sanitizada
```

Esse slice é vertical e utilizável sem criar Insights, Recursos, agentes ou novas telas.
