# Relatório da implementação incremental

- Data da auditoria e implementação: 2026-07-20
- Repositórios: `bbrain-core`, `bbrain-api` e `bbrain`
- Estado: fundação vertical implementada e validada localmente; rollout automático não autorizado
- Bloqueio documental: os 232 termos não constam no anexo recebido

## Resultado

O slice escolhido foi conversa sem transcrição → estado efêmero → captura pós-resposta de Humor/Sono → validação → persistência → leitura/CRUD manual → revisão/invalidação/projeção. Ele preserva o chat síncrono e o layout atual, trata saída de modelo como não confiável e mantém os writes sob casos de uso. A captura automática fica desligada por padrão; geração de Insights, Recursos/RAG, padrões longitudinais e memória pessoal durável não foram simulados como se estivessem prontos.

## Implementação no backend

### Conversa e idempotência

- [`SendChatMessageUseCase`](../../bbrain-api/src/use-cases/conversation/send-chat-message.use-case.ts) aceita `clientMessageId`, captura um horário de referência, reivindica o ledger e reserva usage antes do provider.
- [`ConversationExchangeLedger`](../../bbrain-api/src/infrastructure/database/mongodb/schemas/conversation-exchange-ledger.schema.ts) persiste HMAC, claim/lease, status, risco/escopo, uso e TTL, sem mensagem ou resposta.
- Reuso do id com HMAC diferente, execução concorrente e replay concluído geram erros estáveis distintos. Replay não devolve a resposta porque ela não é armazenada.
- Lease vencida pode ser reivindicada; falha de provider/usage libera claim ativa e reserva.
- Schema, mapper, port e repository de `conversation_messages` foram removidos.

Limite conhecido: se o cliente perder a resposta após conclusão, não há recuperação literal; ele recebe `MESSAGE_ALREADY_PROCESSED`. O trade-off prioriza minimização. Eventos/captura continuam sem outbox/reconciliação durável.

### Privacidade, contexto e perfil

- [`ConversationAgentContextBuilderService`](../../bbrain-api/src/use-cases/conversation/conversation-agent-context-builder.service.ts) aplica privacy flags antes de selecionar perfil/estado e não consulta histórico literal.
- Diagnóstico, medicação e suporte profissional autorrelatados não entram no contexto geral.
- Flags legadas ausentes falham fechadas para Insights de Humor e storage sensível.
- [`ConversationState`](../../bbrain-api/src/domain/conversation/entities/conversation-state.entity.ts) guarda apenas snapshot mínimo com TTL. [`ConversationStateUpdatePolicy`](../../bbrain-api/src/domain/conversation/services/conversation-state-update-policy.service.ts) rejeita rótulo clínico e trecho copiado.
- `profileUpdate` e seu service foram removidos. Uma conversa não grava temas, padrões, estratégias ou resumo no perfil reflexivo.
- [`ConversationSafetyReplyPolicy`](../../bbrain-api/src/use-cases/conversation/conversation-safety-reply.policy.ts) impede confirmação de autorrotulação clínica, sintomas acrescentados e exclusividade emocional; o caso “só você” com impulsividade exige checagem de segurança.
- Captura revalida existência/estado da conta e permissões depois da chamada externa, evitando write quando a conta é desativada durante o voo.

### Contrato de extração

- Port: [`observation-extractor.port.ts`](../../bbrain-api/src/use-cases/wellbeing-history/ports/observation-extractor.port.ts).
- Prompt versionado: [`observation-extraction.prompt.ts`](../../bbrain-api/src/infrastructure/wellbeing-history/prompts/observation-extraction.prompt.ts).
- Schema de provider: [`observation-extraction.schema.ts`](../../bbrain-api/src/infrastructure/wellbeing-history/provider-schemas/observation-extraction.schema.ts).
- Parser estrito: [`observation-extraction-response.parser.ts`](../../bbrain-api/src/infrastructure/wellbeing-history/structured-output/observation-extraction-response.parser.ts).
- Adapters: [`openai-observation-extractor.ts`](../../bbrain-api/src/infrastructure/openai/openai-observation-extractor.ts), [`gemini-observation-extractor.ts`](../../bbrain-api/src/infrastructure/gemini/gemini-observation-extractor.ts) e [`noop-observation-extractor.ts`](../../bbrain-api/src/infrastructure/mock/noop-observation-extractor.ts).
- Router: [`observation-extractor-router.ts`](../../bbrain-api/src/infrastructure/wellbeing-history/observation-extractor-router.ts), com primary e no máximo um fallback.

OpenAI e Gemini recebem `store: false`. O schema evita keywords não suportadas pelo structured output do Gemini. O parser exige JSON com chaves exatas, enumerações válidas, datas de calendário possíveis e offset explícito para instantes/intervalos. A citação precisa ser trecho literal da mensagem atual durante a validação, mas é substituída por `evidenceFingerprint` HMAC antes do aggregate/repository. Sujeito, negação, relato de terceiro, desejo, hipótese e correção são validados novamente pela aplicação. Score, intensidade, limites de escala e números de Sono falsificáveis exigem grounding literal conservador na evidência, não apenas uma citação qualquer. Campos qualitativos continuam dependentes de prompt, parser, policy e evals; por isso shadow é obrigatório antes de writes.

Correções usam `removeFields` para distinguir ausência de remoção explícita. “Na verdade foi anteontem” pode corrigir apenas a temporalidade; “não era tristeza, era frustração” remove/adiciona campos sem duplicar o registro. Observações recentes entram somente quando a mensagem atual contém marcador determinístico de correção e são dados não confiáveis.

### Aggregate, persistência e revisions

- Aggregate: [`wellbeing-observation.entity.ts`](../../bbrain-api/src/domain/wellbeing-history/entities/wellbeing-observation.entity.ts).
- Tipos e invariantes: [`wellbeing-observation.types.ts`](../../bbrain-api/src/domain/wellbeing-history/value-objects/wellbeing-observation.types.ts) e [`wellbeing-observation.validators.ts`](../../bbrain-api/src/domain/wellbeing-history/value-objects/wellbeing-observation.validators.ts).
- Policy de candidate: [`wellbeing-candidate-validation.policy.ts`](../../bbrain-api/src/domain/wellbeing-history/services/wellbeing-candidate-validation.policy.ts).
- Mapper/repository/schema Mongo: [`wellbeing-observation.mapper.ts`](../../bbrain-api/src/infrastructure/database/mongodb/mappers/wellbeing-observation.mapper.ts), [`mongo-wellbeing-observation.repository.ts`](../../bbrain-api/src/infrastructure/database/mongodb/repositories/mongo-wellbeing-observation.repository.ts) e [`wellbeing-observation.schema.ts`](../../bbrain-api/src/infrastructure/database/mongodb/schemas/wellbeing-observation.schema.ts).

Kinds implementados:

```text
mood_event
mood_daily_summary
sleep_record
```

Temporalidade suporta momento, dia, noite específica, intervalo, período e desconhecido, com timezone e precisão exata/aproximada. Sono preserva a precisão por campo e aceita registros parciais. Humor separa rating geral de intensidade; “7/10” mantém `scaleMin` ausente. “Misto” usa `isMixed: true`, nunca label localizada fabricada.

A coleção única `wellbeing_observations` usa mapper central para `snake_case`, idempotência única por usuário, consulta por tipo/data, lineage por source ids e slot derivado diário único parcial. Proveniência e revisions ficam embutidas, validadas, contíguas e limitadas a 500; o limite é hard stop para migração, não truncamento silencioso.

### Captura, projeção e exclusão

- [`CaptureWellbeingObservationsUseCase`](../../bbrain-api/src/use-cases/wellbeing-history/capture-wellbeing-observations.use-case.ts) aplica policy antes/depois do provider, valida, deduplica, persiste/corrige e contabiliza tokens auxiliares sem contar outra mensagem do usuário.
- A idempotency key automática inclui conversa, mensagem fonte, kind e prefixo do fingerprint HMAC; candidatos semanticamente duplicados no mesmo output são colapsados.
- [`DailyMoodSummaryProjectorService`](../../bbrain-api/src/use-cases/wellbeing-history/daily-mood-summary-projector.service.ts) exige pelo menos dois eventos, limita a 12 descritores/200 fontes e mantém um slot derivado estável por usuário/dia.
- Fonte manual ou resumo explícito prevalece sobre derivado. O read path verifica a composição e as versões atuais das fontes elegíveis dentro dos caps de 500 observações/200 fontes; divergência ou fonte nova ainda não projetada oculta o resumo.
- Exclusão/correção de fonte invalida e tenta reconstruir derivados. Excluir o próprio resumo não o recria automaticamente.
- [`WellbeingCaptureCoordinator`](../../bbrain-api/src/use-cases/wellbeing-history/wellbeing-capture-coordinator.service.ts) serializa captura por usuário no processo e permite que account purge bloqueie/drene tarefas antes de apagar dados.

Não há transação/outbox entre observation, eventos e projection. Uma falha pode deixar trabalho pendente, embora idempotência, slot estável, stale e guarda de leitura evitem servir parte dos estados inconsistentes. Fila/lease/reconciliation distribuída é necessária antes de várias réplicas.

### API e autorização

[`WellbeingHistoryController`](../../bbrain-api/src/controllers/wellbeing-history.controller.ts) expõe, sempre com JWT e ownership derivado do token:

```text
GET    /wellbeing-history/observations
POST   /wellbeing-history/observations
PATCH  /wellbeing-history/observations/:observationId
DELETE /wellbeing-history/observations/:observationId?expectedRevision=N
```

POST aceita `clientRequestId`; reuso com payload diferente retorna conflito. PATCH aplica semântica JSON Merge Patch: campo omitido permanece e `null` remove, sempre com `expectedRevision`. Datas, offsets, timezone, precisão e limites de domínio passam por parser estrito. A resposta do dono inclui uma cadeia de proveniência sanitizada, sem `userId`, idempotency key, modelo, prompt ou schema internos.

[`InsightsController`](../../bbrain-api/src/controllers/insights.controller.ts) expõe `GET /insights`. O backend exige plano Pro efetivo; free/standard/expirado recebem 403 estável. Pro recebe `status=insufficient_data`, lista vazia e policy version — nenhum Insight é inventado.

Histórico e CRUD manual não são premium.

## Implementação no frontend

- O chat envia UUID criptograficamente estável por tentativa em [`chat/page.tsx`](../../bbrain/src/app/chat/page.tsx).
- [`wellbeing-history.service.ts`](../../bbrain/src/services/api/wellbeing-history.service.ts), [`useWellbeingHistory.ts`](../../bbrain/src/hooks/useWellbeingHistory.ts), tipos e utilitários consomem dados reais.
- Humor escolhe no máximo um rating geral explícito por dia. Só normaliza e calcula a média semanal quando mínimo e máximo foram declarados; não preenche dias ausentes nem interpreta escala sem limite inferior como 0. Ratings como “7/10” aparecem separadamente no valor original.
- Sono mostra apenas registros reais na janela, preserva parciais/aproximações e não usa fallback antigo fora do período.
- Insights usa o endpoint real; o mock foi removido e a cópia de entitlement foi alinhada ao plano Pro.
- Preferências de privacidade existentes expõem os quatro flags e avisam que edição/exclusão pela interface ainda está em preparação.

Não foram alterados layout, CSS ou navegação; não foram criadas páginas nem check-ins. O CRUD manual ficou disponível no backend, mas sem controles novos de frontend porque o layout atual não oferece um encaixe seguro sem redesenho.

## Manifesto de alterações

O snapshot contém 143 arquivos alterados/adicionados/removidos: 22 arquivos em `bbrain-core`, 103 arquivos de aplicação/teste/configuração em `bbrain-api` e 18 arquivos de consumo/tipos/i18n no `bbrain`. Não houve alteração de lockfile, dependência, CSS, layout ou navegação. A documentação agrupa os arquivos por responsabilidade; o inventário exato é reproduzível com `git diff --name-only` mais `git ls-files --others --exclude-standard` em cada repositório.

## Configuração e rollout

As variáveis estão documentadas em [`bbrain-api/.env.example`](../../bbrain-api/.env.example). Provider/model são configuração de infraestrutura; nomes concretos não entram no domínio. `AI_OBSERVATION_EXTRACTION_ENABLED=true` liga execução/validação shadow; writes só ocorrem se `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED=true` também estiver habilitada. As duas são `false` por padrão, de modo que medir o extractor não implica persistência sensível.

Procedimento antes de produção:

1. atualizar o runtime Node para uma linha LTS suportada;
2. criar migration/preflight explícito para collection e índices, checando duplicatas;
3. executar tests de integração contra Mongo e rollback em snapshot sanitizado;
4. executar contract tests reais de OpenAI/Gemini com `store:false` verificado;
5. rodar shadow sem writes, medir os 17 casos e dataset ampliado;
6. revisar falsos fatos/correções com responsável humano e privacy/security review;
7. adicionar outbox/ledger/reconciliation e coordenação multi-réplica;
8. ativar persistência somente em cohort pequeno com kill switch independente;
9. monitorar schema invalid, false facts, duplicação, custo, p95 e exclusão;
10. expandir somente após os gates de [eval-strategy.md](./eval-strategy.md).

Rollback: desligar execução/persistência automática sem afetar chat; manter a coleção aditiva; retirar readers do rollout se necessário; não apagar dados ou índices no mesmo deploy. Registros de rollout afetado devem ser corrigidos/excluídos por política explícita do usuário/produto.

## Testes e verificações

Baseline anterior: 15 suites, 95 testes; cobertura 33,61% statements, 27,48% branches, 38,65% functions e 34,78% lines.

Validação do snapshot final:

```text
bbrain-api  pnpm test --runInBand                         31 suites / 232 testes passaram
bbrain-api  pnpm run build                                passou
bbrain-api  pnpm run lint                                 passou
bbrain-api  pnpm exec tsc --noEmit --pretty false         passou
bbrain-api  Jest com coverage                             31 suites / 232 testes passaram
bbrain      pnpm run lint                                 passou, com warning não bloqueante de baseline-browser-mapping
bbrain      pnpm exec tsc --noEmit --incremental false    passou
bbrain      pnpm build                                    passou; 22 rotas geradas
bbrain-core git diff --check + links Markdown             passaram
```

Cobertura final do backend: 49,12% statements, 49,58% branches, 51,60% functions e 50,15% lines. Isso representa aumento sobre o baseline, mas não substitui os testes de integração ausentes. Os 17 fixtures são testes determinísticos, não evals de modelo. Não foram executados provider real, Mongo integration, browser e2e, dependency audit ou produção.

## Riscos e débitos explícitos

- anexo sem a lista dos 232 termos;
- crash durante/depois do provider mantém a claim até a lease expirar; uma retomada pode repetir a chamada, e crash após conclusão não permite recuperar a resposta literal;
- usage, estado, conclusão do ledger, eventos e captura ainda não formam uma transação/outbox reconciliável;
- coordination/purge apenas in-process;
- observation/event/projection sem transação ou outbox;
- deduplicação conservadora existe no mesmo output/retry, mas repetição semântica em mensagens diferentes ainda pode gerar dois eventos;
- cap de 500 na listagem pode omitir fontes antigas;
- múltiplos resumos `user_explicit`/`manual_override` do mesmo dia ainda podem coexistir; precedência de leitura é determinística, mas falta constraint/policy de slot único;
- máximo de 500 revisions embutidas exige migração futura;
- variação da fronteira da citação pelo modelo pode produzir outro fingerprint/slot idempotente;
- controles manuais ainda não existem no frontend;
- captura automática sem resultados de provider/eval real;
- grounding literal cobre números falsificáveis, mas labels qualitativos, horários e resolução temporal ainda dependem de model/prompt/eval;
- nenhuma medição de custo ou latência em produção;
- usage não reserva previamente um orçamento estimado de tokens para a chamada auxiliar; fallback pode consumir acima da primeira tentativa;
- fallback configurável pode reenviar a mensagem sensível a um segundo provider, o que exige allowlist/data-policy explícita por ambiente;
- Insights é apenas entitlement/read foundation;
- Recursos/RAG é somente estratégia, sem corpus/retrieval/synthesis;
- GenerateContent do Gemini é legado suportado e deve migrar só após parity/evals;
- Node 20 está EOL e deve ser atualizado;
- account deletion apaga bem-estar e drena tarefas locais, mas usage/billing ainda exigem uma política explícita de purge/anonymização/retenção legal;
- privacy é rechecada antes do write de bem-estar, mas o write de `ConversationState` ainda usa flags capturadas antes do provider; revogação durante a requisição pode disputar com esse write, mitigado pelo TTL, porém ainda requer revalidação.

## Custos e latência

Não há medida observada. Com captura habilitada, uma mensagem elegível adiciona uma chamada estruturada e tokens auxiliares depois da resposta; o usuário não espera essa chamada no caminho HTTP, mas o custo por conversa aumenta. Primary/fallback limita a no máximo duas tentativas e timeouts configurados impõem teto técnico. Projeção/CRUD são operações Mongo pequenas, mas listagem e rebuild atuais carregam até 500 documentos.

Antes de definir orçamento, medir por task/provider/model: input/output tokens, duração p50/p95/p99, taxa de fallback, schema invalid, candidate/write rate e custo por usuário ativo. Qualquer número sem shadow/canary é hipótese de engenharia, não previsão comercial.

## Decisões adiadas ou rejeitadas

Adiado com gatilho: outbox/fila durável, append-only revision store, execution telemetry, memory tipada, Insights longitudinal, Resources/RAG, migração Gemini Interactions e paginação.

Rejeitado no slice: multiagente, GraphRAG, knowledge graph, memória vetorial pessoal, check-in obrigatório, backfill automático de conversas e troca de modelo sem benchmark. Essas escolhas evitam complexidade e exposição de dados antes de existir evidência operacional.
