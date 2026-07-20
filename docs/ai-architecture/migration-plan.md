# Plano de migração evolutiva e reversível

## Estratégia

Usar `expand -> migrate/shadow -> switch -> contract`. Nenhuma fase remove campo/collection/endpoint existente antes de:

- localizar consumidores;
- introduzir substituto compatível;
- medir leitura/escrita;
- reconciliar dados;
- validar rollback;
- cumprir a janela de compatibilidade.

Cada fase é liberável separadamente e tem kill switch. O primeiro objetivo é uma fundação vertical funcional, não uma árvore de abstrações vazias.

## Estado real após o primeiro slice

| Fase | Estado em 2026-07-20 | Evidência/limite |
|---|---|---|
| 0 | parcial | privacy flags limitam contexto/captura; schema de conversa literal foi removido; purge inclui estado/ledger/bem-estar; fail-closed global, rate limit e runtime LTS continuam pendentes |
| 1 | parcial | contratos de extração e versões existem; gateway universal e execution telemetry ainda não |
| 2 | código aditivo pronto | schemas de `wellbeing_observations`, `conversation_states` e ledger sem conteúdo existem; faltam preflight e teste Mongo real |
| 3 | código shadow implementado; não operado | `AI_OBSERVATION_EXTRACTION_ENABLED=true` com persistência false executa/valida/contabiliza sem writes; não houve provider contract test nem métrica real |
| 4 | protegida por flag própria off | auto-persist requer também `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED=true`; não houve cohort/canary |
| 5 | backend implementado; frontend parcial | GET/POST/PATCH/DELETE, ownership, provenance e revisão existem; frontend só lê, sem controles manuais novos |
| 6 | implementada | projeção diária estável `v2`, stale/rebuild e guarda por revisão no read path |
| 7 | futura | executor/coordenador é apenas in-process; sem outbox, fila, lease, DLQ ou reconciliação distribuída |
| 8 | fundação de entitlement | `/insights` exige Pro efetivo e retorna `insufficient_data`; geração/validação longitudinal não existe |
| 9 | documentada | Recursos/RAG não foi implementado |
| 10 | implementada no código | `profileUpdate` e transcrição saíram do fluxo; estado TTL e ledger HMAC estão ativos. Banco pré-MVP será recriado, sem script de migração |

As fases abaixo continuam sendo o plano de rollout completo. “Implementada” na tabela significa código local e testes determinísticos, não promoção para produção.

## Pré-condições

1. Receber o glossário ausente; isso não bloqueia o slice técnico, mas bloqueia o entregável exato de 232 termos.
2. Congelar fotografia de contratos backend/frontend e coleções.
3. Definir responsáveis de produto, segurança/privacidade e engenharia para gates.
4. Criar baseline de CI com install frozen, lint, typecheck, build, tests e dependency/container scan.
5. Fazer backup e restore testado antes de migrations de dados.

## Fase 0 — segurança operacional

Escopo:

- validação fail-closed de env;
- remover reset code/email body de logs;
- rate limits de auth/reset/chat;
- aplicar privacy flags no context builder e profile update;
- correlation ID e logger allowlist;
- migrar Node 20 EOL para linha LTS suportada;
- completar/instrumentar account deletion.

Evidências que motivam: `config.ts:141`, `email.service.ts:31-33,57-69`, `conversation-agent-context-builder.service.ts:47-92`, `Dockerfile:1`, `account-lifecycle.service.ts:50-57`.

Rollback: config flag por controle, imagem anterior apenas durante janela curta e com risco aceito; a remoção de secret/log inseguro não deve ser revertida em produção.

Gate de saída: tests de startup/privacy/redaction/rate limit passam; scanner não usa runtime EOL; nenhuma credencial/reset code aparece em capture de logs.

## Fase 1 — contratos e policies sem mudança funcional

Adicionar boundaries:

- `AiExecutionGateway`/capabilities lógicas;
- `AiExecutionPolicy` e registry de prompt/schema versionados;
- `PrivacyPolicy` e `EntitlementPort`;
- `Clock`, id generator e temporal resolver ports quando necessários;
- execution metadata/metrics sem conteúdo;
- feature flags no backend.

Adapter de compatibilidade envolve o `ChatAgent` atual, preservando response pública. Não trocar modelo/provider nessa fase.

Rollback: composition volta ao adapter atual; novos tipos não têm writes de produto.

Gate: 95 testes atuais, novos boundary/contract tests, lint/typecheck/build e parity test do chat.

## Fase 2 — expand de persistência

Criar preflight explícito para as coleções aditivas descritas em [mood-sleep-data.md](./mood-sleep-data.md) e no [ADR-0007](./adr/0007-transcript-free-conversation-state.md). O slice já contém schemas, mappers, repositories e declarações de índice, mas isso não substitui validação Mongo real. Neste ambiente pré-MVP não haverá migração de transcrição: o responsável pelo produto recriará o banco.

Migration ledger conceitual:

```text
id, checksum, applied_at, application_version, status, rollback_id
```

Procedimento por índice:

1. verificar duplicatas/shape em read-only preflight;
2. criar índice em background/mecanismo compatível;
3. validar uso/uniqueness;
4. só então ativar writer.

Rollback: desligar writers e manter collections; dropar índice/collection somente depois de confirmar ausência de dados necessários e backup, em mudança separada aprovada.

Gate: migration forward/rollback em snapshot sanitizado, sem lock/degradação inaceitável.

## Fase 3 — extração shadow

Fluxo:

```text
mensagem atual em memória
 -> post-response executor
 -> extraction candidate
 -> schema/domain validation
 -> execution metrics + candidate counters
 -> descartar candidate de produto
```

Não persistir evento pessoal nem alterar resposta/perfil. O código entregue mantém essa separação: `AI_OBSERVATION_EXTRACTION_ENABLED` controla execução e `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED` controla writes. Se amostra humana for aprovada, usar processo separado e minimizado.

Rollback: flag `wellbeing_extraction_shadow=false`.

Gate: gates de schema, precision e critical set em [eval-strategy.md](./eval-strategy.md); custo/latência dentro de budget; privacy review.

## Fase 4 — auto-persist controlada

Ativar para cohort pequeno:

- idempotency reservation;
- persistência de candidates aprovados;
- provenance/revision;
- zero frontend dependency;
- reconciliation job/read check;
- kill switch independente do chat.

Não fazer backfill de conversas históricas por padrão. Backfill de conteúdo sensível exige finalidade, consentimento/base, janela, modelo/prompt version, custo, revisão e rollback próprios.

Rollback:

1. desligar `wellbeing_auto_persist`;
2. manter records marcados com rollout/policy version;
3. invalidar ou tombstonar records do cohort se incident review determinar;
4. nunca reescrever como “manual”.

Gate: duplicação zero, critical false fact zero, correction/edit rate aceitável e deletion propagation.

## Fase 5 — leitura e controle manual

Adicionar use cases/endpoints compatíveis:

- listar records/eventos por período;
- inserir manualmente;
- editar com expected revision;
- excluir/tombstone;
- recuperar provenance em forma segura.

Frontend só adapta clientes, tipos, stores/hooks, loading/error e botões já existentes. Se o layout atual não expuser uma capacidade, backend permanece pronto e a limitação é documentada.

Rollback: endpoints novos podem ser ocultados por flag, sem remover dados. Contratos usam campos opcionais durante rollout.

Gate: ownership/IDOR/e2e, concurrency, accessibility do fluxo existente e nenhuma alteração de CSS/layout.

## Fase 6 — projeção diária de Humor

Ativar `DailyMoodSummary`:

- build determinístico;
- `insufficient_data` em ausência;
- manual/explicit precedence;
- stale/invalidation/rebuild;
- read path verifica source revisions.

Não gerar linha contínua em dias ausentes.

Rollback: desligar projection/read alias; eventos primários permanecem. Rebuild com versão anterior é possível.

Gate: absent-vs-neutral, mixed, coverage, correction/invalidation tests 100% nos fixtures.

## Fase 7 — executor assíncrono durável, se necessário

Manter `JobPort` desde a fase 3. Migrar de executor local para fila somente se ocorrer um gatilho objetivo:

- mais de uma réplica processando pós-resposta;
- job perdido/duplicado acima de 0,1%;
- necessidade de retry além do ciclo do processo;
- backlog sustentado > 1 minuto;
- requisito operacional de DLQ/replay.

Implementar inbox/idempotency, retry/backoff, DLQ metadata, concurrency limit e reconciliation.

Rollback: pausar consumer, drenar/reconciliar e voltar ao executor anterior com a mesma idempotency key. Nunca executar ambos sem ownership/partition explícita.

## Fase 8 — Insights premium

Pré-condições:

- eventos/revisões/projeções estáveis;
- coverage observável;
- entitlement backend;
- invalidate-on-edit/delete;
- evals de segurança/evidência aprovados.

Rollout:

1. candidates offline/shadow;
2. validator e evidence gate;
3. cohort interno;
4. premium canary;
5. expansão.

Rollback: `insights_generation/read=false`, marcar candidates da policy afetada stale; histórico básico permanece acessível.

## Fase 9 — Recursos/RAG

Executar como produto separado:

1. schema de documentos/chunks e rights metadata;
2. corpus oficial allowlisted;
3. lexical/filter baseline;
4. retrieval eval;
5. synthesis/citation verifier;
6. injection/security tests;
7. endpoint que não dispara memory/profile writes;
8. corpus editorial somente após rights/review workflow.

Rollback: corpus alias para versão anterior; desligar retrieval/synthesis e comunicar indisponibilidade. Nunca fazer fallback para memória do modelo sem fonte.

## Fase 10 — contract/cleanup

Somente após janela medida:

- manter removido o `profileUpdate` acoplado ao reply e versionar o contrato de estado efêmero;
- migrar consumidores do `current_context_summary` para state/memory tipados;
- decidir destino do esqueleto `check-in` incompatível;
- eliminar campos legado `premium`/`woovi` após backfill e zero reads;
- retirar adapter Gemini GenerateContent legacy após parity/evals do Interactions `v1`;
- remover flags encerradas.

Cada remoção é PR/migration separada com rollback. Não combinar cleanup destrutivo com feature rollout.

## Migrações específicas de dados existentes

### Reflective profile

Não mover inicialmente. Introduzir dual-read:

```text
preferência nova tipada -> nova fonte
senão -> reflective_profile legacy
```

Dual-write apenas quando necessário e com reconciliation. `current_context_summary` ganha validade/versão antes de qualquer substituição. Diagnósticos/medicação permanecem autorrelatados.

### Plan `premium`

Mapear explicitamente para plano válido por migration report; contar documentos antes/depois. O schema atual aceita legado em `user.schema.ts:121-126`, e o domínio o reconhece em `plan-definition.ts:49`. Remover somente após zero ocorrências e rollback snapshot.

### Woovi -> Asaas

O mapper atual faz fallback (`user.mapper.ts:132-146`). Backfill `asaas_customer_id` a partir de legado, validar por provider e manter fallback por uma janela. Nunca apagar o campo legado no mesmo deploy do backfill.

### Conversation history

Não reprocessar nem migrar transcrições. Neste ambiente pré-MVP, o banco será recriado antes do próximo uso e o schema legado não faz mais parte da aplicação. Se o usuário editar/apagar uma fonte estruturada futura, derived records são invalidados por source id.

## Matriz de compatibilidade

| Mudança | Write | Read durante rollout | Rollback |
|---|---|---|---|
| novas collections Humor/Sono | flag/cohort | endpoints novos; legado intacto | desligar writer/read |
| prompt/model policy | execution version | resolver active + previous | mudar alias de policy |
| daily summary | projection version | somente `current` e revisions válidas | alias/rebuild anterior |
| Insights | feature + entitlement | stale bloqueado | kill switch, sem afetar events |
| corpus RAG | índice versionado | alias atômico | alias anterior |
| Gemini Interactions | adapter/canary | contrato canônico comum | voltar adapter legacy |
| Node runtime | imagem versionada | canary/health | imagem anterior por janela curta |

## Verificação por fase

```bash
pnpm install --frozen-lockfile
pnpm run lint
pnpm exec tsc --noEmit --pretty false
pnpm test --runInBand
pnpm build
```

Adicionar conforme existir:

```text
migration:status / migration:up / migration:verify / migration:down
eval:<task> --dataset-version ...
contract:<provider>
integration:mongo
e2e:privacy
security:scan
```

O build atual escreve `dist`; executar em CI/worktree apropriado, não durante auditoria read-only.

## Critérios de abort/rollback

- qualquer false personal fact no critical set/canary;
- qualquer cross-user/cross-corpus leak;
- conteúdo sensível em logs/traces;
- stale Insight servido;
- schema invalid >1%;
- fallback não aprovado usado;
- correction rate dobra versus baseline;
- custo excede cap em >1% das execuções;
- p95 ultrapassa target por 15 minutos;
- migration reconciliation diverge;
- erro de exclusão/retention.

## Ordem recomendada de PRs/conjuntos semânticos

1. config/redaction/privacy/rate-limit/runtime;
2. versioned AI execution contracts e telemetry;
3. schemas/domain de provenance e records + tests;
4. Mongo migrations/repositories/mappers + integration harness;
5. shadow extractor + eval runner/dataset;
6. auto-persist canary;
7. manual CRUD + invalidation;
8. daily projection;
9. queue somente se gatilho;
10. Insights;
11. Resources ingestion/retrieval/synthesis;
12. cleanup legado.

## Riscos restantes

- Sem glossário, a classificação conceitual de 232 itens continua bloqueada.
- Sem dados reais minimizados e review, volumes/custos são estimativas.
- Texto de consentimento e retenção precisa de validação de produto/jurídica.
- Provider data residency e contratos precisam de revisão organizacional.
- Regras clínicas/medicamentos exigem revisão editorial apropriada antes de Recursos em produção.
