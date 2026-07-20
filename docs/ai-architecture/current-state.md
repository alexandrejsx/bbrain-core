# Estado atual, bounded contexts e riscos

> Este documento é a fotografia de entrada da auditoria no commit backend `58c2086`. As correções e diferenças introduzidas pelo slice estão consolidadas em [implementation-report.md](./implementation-report.md); referências abaixo não devem ser lidas como o estado pós-implementação.

## Escopo e método

Esta fotografia descreve `bbrain-core` e `bbrain-api` no commit de backend `58c2086`. Foram inspecionados manifests, configuração, módulos NestJS, domínio, casos de uso, adapters, schemas MongoDB, testes, Docker e documentação. `bbrain` foi auditado separadamente para contratos e restrição de layout; este documento trata as fronteiras relevantes ao backend e à IA.

## Stack e composição

- NestJS 11, TypeScript 5.7, MongoDB/Mongoose, `@nestjs/event-emitter`, JWT, bcrypt, Stripe e clients HTTP com `fetch`: `bbrain-api/package.json:8-65`.
- A composição raiz registra MongoDB, eventos, usuários, auth, conversa, planos e perfil: `bbrain-api/src/app.module.ts:13-28`.
- Configuração é carregada de `process.env` com `ignoreEnvFile: true`; o script local usa `env-cmd`: `bbrain-api/src/app.module.ts:15-19`, `bbrain-api/package.json:16`.
- O único container de desenvolvimento é MongoDB: `bbrain-api/docker-compose.yml:1-10`.

## Fluxo real de conversa

```text
POST /chat/message
  -> JwtAuthGuard
  -> SendChatMessageUseCase
      -> ConversationAgentContextBuilderService
          -> UserRepository
          -> ReflectiveProfileRepository
          -> ConversationMessageHistoryPort
      -> UsageService.assertCanSendMessage
      -> ChatAgent.respond
          -> buildChatMessages
          -> OpenAI | Gemini | Mock
          -> parseChatAgentResponse
      -> ProfileUpdateService.apply
      -> salvar perfil + exchange
      -> registrar uso agregado
      -> EventEmitter2.dispatch
  -> reply + riskLevel + scopeStatus
```

Evidências:

- Entrada e ownership por JWT: `bbrain-api/src/controllers/chat.controller.ts:20-35`.
- Orquestração: `bbrain-api/src/use-cases/conversation/send-chat-message.use-case.ts:54-132`.
- Contexto limitado a 20 mensagens, 4.000 caracteres cada: `bbrain-api/src/use-cases/conversation/conversation-agent-context-builder.service.ts:14-17,43-108`.
- Port canônico: `bbrain-api/src/use-cases/conversation/chat-agent.port.ts:18-35`.
- Seleção estática por `AI_CHAT_PROVIDER`: `bbrain-api/src/modules/conversation.module.ts:38-52`.
- OpenAI Responses API, `store: false`, JSON Schema strict: `bbrain-api/src/infrastructure/openai/openai-chat-agent.ts:18,109-133`.
- Gemini GenerateContent `v1beta`, JSON estruturado: `bbrain-api/src/infrastructure/gemini/gemini-chat-agent.ts:18,100-127`.
- Parser estrutural local: `bbrain-api/src/infrastructure/chat/structured-output/chat-agent-response.parser.ts:200-238`.

## Mapa dos bounded contexts

| Contexto | Estado observado | Responsabilidade atual | Dependências/consumidores | Evidência |
|---|---|---|---|---|
| Users/Auth/Profile | operacional | cadastro, login, JWT, perfil de onboarding, reset e exclusão agendada | MongoDB; conversa lê identidade e privacidade; billing lê plano | `src/domain/users`, `src/use-cases/auth`, `src/controllers/auth.controller.ts` |
| Conversation | operacional parcial | resposta do BBrain, escopo, contexto recente, perfil reflexivo e histórico | Users, Usage, providers de IA, EventEmitter | `src/use-cases/conversation`, `src/infrastructure/chat`, `src/infrastructure/openai`, `src/infrastructure/gemini` |
| Reflective Profile | operacional, acoplado à conversa | preferências, temas, padrões, estratégias, diagnósticos/medicação autorrelatados e resumo atual | Context builder e update automático do chat | `src/domain/conversation/entities/reflective-profile.entity.ts:5-161`; schema em `reflective-profile.schema.ts:11-61` |
| Usage | operacional | limites de mensagem/tokens em janela móvel | Users/Plans; chamado antes e depois do provider | `src/domain/usage`, `src/use-cases/conversation/send-chat-message.use-case.ts:72,115` |
| Plans/Billing | operacional | catálogo, checkout, Pix, assinaturas, entitlement de plano | Users, Stripe, Asaas, Usage, MongoDB | `src/use-cases/billing`, `src/modules/plans.module.ts` |
| Events | infraestrutura mínima | publicação síncrona em memória | Auth e Conversation publicam; não há handlers | `src/infrastructure/events/event-dispatcher.adapter.ts:7-13`; `README.md:112-116` |
| Check-in | esqueleto de domínio | entidades escalares de humor/sono/energia/stress/motivação | apenas teste arquitetural; sem repository/use case/controller | `src/domain/check-in`; `bbrain-contexts.spec.ts:72-83` |
| Memory | esqueleto de domínio | memória com categoria, confiança, fonte e retenção | apenas teste; sem persistência/fluxo real | `src/domain/memory/entities/memory.entity.ts:13-59` |
| Pattern Analysis | esqueleto de domínio | Insight com janela, confiança e status | apenas teste; sem entitlement/persistência | `src/domain/pattern-analysis/entities/pattern-insight.entity.ts:10-50` |
| Summary | esqueleto de domínio | resumo de período | apenas teste; sem projeção/persistência | `src/domain/summary/entities/user-summary.entity.ts:11-49` |
| Risk Assessment | esqueleto de domínio | sinais e avaliação determinística | não conectado ao chat; risco atual vem do mesmo output do LLM | `src/domain/risk-assessment`; `chat-agent.port.ts:5,18-23` |
| Journal | esqueleto de domínio | entrada privada e tags | não conectado à aplicação/persistência | `src/domain/journal` |
| Support Plan | esqueleto de domínio | metas, passos e progresso | não conectado | `src/domain/support-plan` |
| Humor | ausente como contexto próprio | hoje aparece apenas como `MoodCheckin` escalar e texto em perfil | sem evento emocional, projeção ou API | `src/domain/check-in/entities/check-in.entities.ts:32-51` |
| Sono | ausente como contexto próprio | hoje aparece apenas como `SleepCheckin.duration` obrigatório | sem parcialidade, temporalidade ou API | `src/domain/check-in/entities/check-in.entities.ts:54-73` |
| Insights premium | ausente operacionalmente | existe esqueleto `PatternInsight`, sem policy de acesso | nenhum controller/repository/job | `src/domain/pattern-analysis` |
| Recursos/RAG | ausente | nenhum corpus, ingestion, retrieval ou citação | nenhum | busca integral em `src`; `README.md:102` |

## Dependências atuais

```text
Controllers
  -> use-cases / application services
      -> domain entities, policies, repositories/ports
      -> ChatAgent port
      -> EventDispatcher port

Modules (composition)
  -> concrete Mongo repositories
  -> OpenAI/Gemini/Mock adapters
  -> Stripe/Asaas adapters
  -> Nest controllers/guards

Infrastructure
  -> domain/application contracts
  -> Mongoose/Nest/external HTTP SDKs
```

A fronteira principal é coerente com Clean Architecture, mas há exceções relevantes:

- `UsageService` está em `src/domain` embora coordene repositories e persistência: `src/domain/usage/services/usage.service.ts:68-158`.
- Casos de uso importam exceções HTTP do Nest, por exemplo `update-user-profile.use-case.ts:1` e `billing.service.ts:1`.
- `ProfileUpdateService` confia em campos produzidos pelo modelo e usa apenas presença textual simples como proteção semântica: `src/use-cases/conversation/profile-update.service.ts:17-61`.
- O mesmo call de IA produz reply, risco, escopo e atualização de perfil, acoplando tarefas com perfis de qualidade diferentes: `chat-agent.port.ts:18-23`.

## Persistência atual

Coleções operacionais:

- `users`: identidade, credencial, onboarding, privacidade e billing agregado.
- `reflective_profiles`: perfil reflexivo sensível.
- `conversation_messages`: texto integral de usuário e assistente.
- `user_daily_usages`: tokens e contadores agregados.
- `payment_orders`, `provider_events`, `user_subscriptions`: billing.

Schemas e filtros persistidos usam majoritariamente `snake_case`, com mappers para domínio, em conformidade com a diretriz global. Não há migration runner, `schema_version`, backfill versionado ou rollback. Compatibilidade legado ocorre de forma local para plano `premium` e `woovi_customer_id`: `src/domain/plans/plan-definition.ts:49`, `src/infrastructure/database/mongodb/mappers/user.mapper.ts:132-146`.

## Testes atuais

- 15 specs, 95 testes passando no baseline.
- A maior parte está sob `src/domain/__tests__`, inclusive testes de casos de uso e boundaries.
- Há um spec de prompt renderer em infraestrutura.
- Cobertura de controllers, guards, Mongo, providers, config e jobs é zero no baseline completo.
- Não há e2e, contract tests externos, eval dataset, LLM-as-a-judge, teste de carga ou teste de migração.

## Observabilidade, jobs e confiabilidade

- Providers registram modelo, duração e erro sanitizado parcialmente; não há tracing, métricas ou correlation ID: `openai-chat-agent.ts:86-165`, `gemini-chat-agent.ts:76-171`.
- Tokens são agregados sem provider/modelo/tarefa/custo/latência: `user-daily-usage.schema.ts:31-44`.
- A exclusão agendada usa `setInterval` em cada instância: `account-lifecycle.service.ts:18-27`.
- Eventos são síncronos e voláteis: `event-dispatcher.adapter.ts:10-13`.
- Webhooks usam uma tabela de idempotência, mas o efeito ocorre antes do marcador processado: `billing.service.ts:203-214`.
- Chat e usage não compartilham transação ou idempotency key: `send-chat-message.use-case.ts:100-126`.

## Riscos priorizados

| Severidade | Risco | Evidência | Impacto | Resposta recomendada |
|---|---|---|---|---|
| crítica | código de reset no log quando email usa/faz fallback para `log` | `email.service.ts:31-33,57-69`; `request-password-reset.use-case.ts:38-58` | tomada de conta e exposição de PII | falhar fechado em produção; nunca registrar corpo/código |
| crítica | JWT aceita `local-secret` | `config.ts:141`; `auth.module.ts:41` | falsificação de token | validação de startup por ambiente e secret obrigatório |
| crítica | flags de privacidade não governam contexto ou update automático | `user.schema.ts:43-48`; `conversation-agent-context-builder.service.ts:47-92` | envio/persistência contra preferência do usuário | policy explícita antes de leitura, envio e escrita |
| alta | login/reset/chat sem rate limit; reset sem limite de tentativas | `auth.controller.ts:26-43`; `confirm-password-reset.use-case.ts:35-46` | brute force, abuso e custo | throttling por IP/conta e lockout temporário auditável |
| alta | exclusão não cobre usage/billing nem define retenção/anonymização | `account-lifecycle.service.ts:50-57` | dados órfãos e descumprimento de expectativa | inventário por classe, purge/anonymize policy e reconciliação |
| alta | Node 20 EOL | `Dockerfile:1`; fonte oficial no README deste diretório | vulnerabilidade e drift de ecossistema | migrar para Node 22/24 LTS com testes |
| alta | fatos pessoais produzidos no mesmo call do reply | `chat-agent.port.ts:18-23`; `send-chat-message.use-case.ts:86-93` | falso fato, acoplamento e difícil rollback | workflow separado, schema/version/proveniência e shadow mode |
| alta | writes e webhooks não atomicamente idempotentes | `billing.service.ts:203-214`; `send-chat-message.use-case.ts:100-126` | duplicação ou estado parcial | reservation/inbox/outbox mínima e operações atômicas |
| média | Gemini usa GenerateContent `v1beta` legacy | `gemini-chat-agent.ts:18,117-127`; docs oficiais | mudança futura e acesso desigual a features | adapter compatível com Interactions `v1`, após contract/evals |
| média | aliases de modelo/prompt/schema não versionados | `config.ts:88-92`; `prompt-registry.ts:3-6` | mudança silenciosa de comportamento | execution record e versões imutáveis |
| média | ausência de migration runner/index deployment explícito | `mongodb.module.ts:9-23`; schemas | rollout irreversível ou índice falhar em produção | ledger de migração, expand/migrate/contract e rollback |
| média | in-process timer/event emitter em múltiplas réplicas | `account-lifecycle.service.ts:18-27`; `event-dispatcher.adapter.ts:10-13` | concorrência e perda de eventos | single-owner job inicialmente; fila quando gatilhos ocorrerem |
| média | cobertura baixa e infraestrutura sem testes | baseline de cobertura | regressões em bordas sensíveis | contracts, integration e e2e priorizados por risco |

## Lacunas de produto relevantes à arquitetura

- Não existem contratos de evento emocional, registro parcial de sono, revisão, correção ou proveniência.
- `MoodCheckin` exige escala numérica e `SleepCheckin` exige duração, incompatíveis com as premissas desta evolução; não devem ser promovidos ao fluxo apenas por já existirem.
- `current_context_summary` é enviado como contexto sem versão, validade ou expiração: `conversation-agent-context-builder.service.ts:86-92`.
- O perfil reflexivo mistura preferências, objetivos, fatos autorrelatados, saúde, memória e contexto temporário em um único documento: `reflective-profile.schema.ts:18-55`.
- Não há entitlement de Insights, feature flags de backend, RAG, corpus, citações ou ingestion.
- Não há endpoint de edição/exclusão/inserção manual de Humor/Sono.

## Conclusão do reconhecimento

O primeiro incremento não deve reutilizar automaticamente o check-in escalar nem ampliar o `profileUpdate` atual. Deve criar uma fundação vertical pequena, aditiva e reversível: contrato canônico de evidência, extração em shadow, validação determinística, collections versionadas, leitura/edição/exclusão manual e invalidação de uma projeção diária. Fila distribuída, Insights e RAG vêm depois que esse fluxo mede precisão, correções, latência e custo.
