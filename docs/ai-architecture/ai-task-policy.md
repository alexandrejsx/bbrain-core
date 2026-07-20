# Política das tarefas de IA

## Estado do primeiro slice

Foram implementadas a conversa já existente e a extração estruturada de Humor/Sono com seleção primary/fallback configurável. A classificação de risco/escopo continua integrada ao contrato conversacional. Insights, Recursos/RAG, memória, padrões e relatórios permanecem políticas-alvo, não funcionalidades entregues. Nenhum nome de provider/model foi levado ao domínio; modelos concretos continuam na composição/configuração e não foram trocados sem benchmark.

## Status e regra de promoção

Esta política define capabilities lógicas e budgets iniciais. Ela não muda o provider ou os modelos atuais. Na fotografia auditada, o chat escolhe `gemini | openai | mock` em `bbrain-api/src/modules/conversation.module.ts:38-52`, com aliases concretos em `bbrain-api/src/config.ts:88-92,117-125,163-166`.

Nenhum nome concreto novo deve ser promovido apenas por ser “mais recente”, barato ou poderoso. Uma mudança só ocorre quando o candidato:

1. suporta o schema e os controles de dados necessários;
2. passa o dataset versionado da tarefa;
3. não piora nenhum gate crítico de segurança, sujeito, negação, temporalidade ou faithfulness;
4. cumpre latência e custo no ambiente real;
5. passa shadow/canary e possui rollback de configuração;
6. tem provider/model snapshot, prompt e schema registrados por execução.

O fallback precisa passar os mesmos gates. Em extração de fatos, “não produzir evento” é preferível a degradar para um modelo não avaliado.

## Classes de custo

Os valores são caps iniciais por execução, não preços esperados nem promessa comercial. Devem ser recalculados automaticamente a partir da tabela de preços vigente antes de cada rollout.

| Classe | Cap inicial | Uso pretendido |
|---|---:|---|
| C0 | US$ 0,001 | regra/classificação muito curta |
| C1 | US$ 0,005 | extração ou normalização curta |
| C2 | US$ 0,020 | conversa/memória/resumo limitado |
| C3 | US$ 0,100 | Insight ou síntese fundamentada complexa |

Ao exceder cap, a execução deve encerrar sem persistir output parcial, registrar `budget_exceeded` sanitizado e seguir o fallback definido.

## Matriz A — requisitos e contexto

| # | Tarefa / capability | Qualidade exigida | Latência alvo inicial | Volume esperado | Sensibilidade | Reasoning | Output | Contexto mínimo | Ferramentas |
|---:|---|---|---|---|---|---|---|---|---|
| 1 | Conversa principal / `CHAT_BALANCED` | acolhimento, escopo, idioma e segurança; sem diagnóstico; continuidade sem inventar memória | p95 <= 8 s; timeout total 30 s | uma por mensagem elegível | muito alta | médio; maior apenas em escalada avaliada | `{reply, scopeStatus, riskLevel, conversationStateUpdate}` validado | mensagem atual, estado efêmero estruturado e preferências permitidas; nunca transcrição ou perfil integral | nenhuma no MVP; capabilities específicas são chamadas pela aplicação, não pelo chat |
| 2 | Roteamento de intenção / `INTENT_FAST` | alta precisão nas rotas `conversation`, `resources`, `account_action`, `out_of_scope`; Recursos não pode atualizar perfil | p95 <= 800 ms | uma por mensagem se regras não resolverem | alta | baixo | enum + confidence + razões codificadas | mensagem atual e route anterior apenas se necessária | regras determinísticas antes do modelo |
| 3 | Extração de Humor / `MOOD_EXTRACTOR_PRECISE` | precisão acima de recall; zero tolerância no set crítico para sujeito/negação/hipótese/terceiro | fora do reply; p95 <= 8 s | até uma por mensagem de usuário | muito alta | médio | schema de `MoodExtractionCandidate[]` | uma mensagem do usuário, ids, timezone e data de recebimento; correções próximas somente quando referenciadas | temporal resolver; lookup limitado de eventos candidatos à correção |
| 4 | Extração de Sono / `SLEEP_EXTRACTOR_PRECISE` | preservar parcialidade/aproximação; não fabricar noite/duração/horário/qualidade | fora do reply; p95 <= 8 s | até uma por mensagem de usuário | muito alta | médio | `SleepRecordCandidate[]` e `SleepPeriodStatementCandidate[]` | mesma boundary da extração de Humor | temporal resolver; lookup limitado de records candidatos |
| 5 | Resolução temporal / `TEMPORAL_RESOLVER` | determinística quando possível; preservar ambiguidade; timezone correto | p95 <= 300 ms determinístico; <= 3 s com modelo | por candidato temporal | alta | baixo/médio apenas em linguagem ambígua | intervalo canônico + precision + unresolved reason | expressão, instante de recebimento, timezone IANA, contexto correção mínimo | biblioteca de tempo/tz; calendário sem acesso externo |
| 6 | Consolidação de memória / `MEMORY_CONSOLIDATOR` | somente fatos/preferências duráveis, consentidos, revisáveis e com provenance; sem diagnóstico inferido | assíncrona; p95 <= 15 s | muito menor que mensagens; acionada por candidato elegível | muito alta | médio | candidate + category + retention + evidence ids | fatos validados, preferências, mensagem fonte mínima; nunca conversa inteira | lookup de memória por usuário/categoria, sem escrita direta |
| 7 | Resumo diário de Humor / `DAILY_MOOD_PROJECTOR` | distinguir misto, neutro explícito e ausência; manual vence derivado; cobertura transparente | sob demanda/job; p95 <= 5 s | no máximo um rebuild por usuário/dia após debounce | muito alta | regra primeiro; modelo opcional baixo/médio | `DailyMoodSummaryCandidate` | eventos ativos do dia e revisão manual | agregador determinístico; modelo só para texto curto se eval mostrar ganho |
| 8 | Geração de Insights / `INSIGHT_REASONER` | associação, não causalidade; evidência/coverage/limitações; não diagnosticar | fora do reply; p95 <= 45 s | baixo, premium, sob demanda/cache | muito alta | alto quando longitudinal | candidate com claims, evidence ids, period e limitations | agregações e eventos ativos mínimos da janela; diagnóstico autorrelatado excluído como explicação automática | query ports de eventos/agregações; sem banco geral |
| 9 | Validação de Insight / `INSIGHT_VALIDATOR` | bloquear afirmação não sustentada, causalidade, diagnóstico, stale evidence e ancoragem | p95 <= 15 s | um por candidate; zero se regra já rejeitar | muito alta | médio/alto | verdict tipado + violation codes + claim mapping | candidate + evidence pack exato + rubric version | regras determinísticas primeiro; verifier model independente se eval justificar |
| 10 | Recuperação de Recursos / `RESOURCE_RETRIEVER` | autoridade, seção, versão, jurisdição, idioma e permissão corretos; alta Recall@k | p95 <= 1,5 s; <= 4 s com rewrite/rerank | por pergunta de Recursos | alta, mas sem dados pessoais | baixo; retrieval é majoritariamente determinístico | chunks + metadata + scores | query sanitizada, locale/jurisdição e corpus; nenhum perfil emocional | busca textual; semântica/híbrida/rerank apenas conforme retrieval eval |
| 11 | Síntese fundamentada / `GROUNDED_SYNTHESIZER` | cada afirmação factual sustentada; comunicar insuficiência; não personalizar tratamento | p95 <= 20 s; timeout 30 s | por pergunta com evidence pack suficiente | alta | médio/alto | resposta em blocos + claim ids/citation ids | pergunta e chunks aprovados; instruções separadas de documentos | apenas evidence pack; nenhuma ferramenta de escrita |
| 12 | Verificação de citações / `CITATION_VERIFIER` | cobertura e entailment por claim; bloquear citações inexistentes/obsoletas | p95 <= 8 s | por síntese factual | alta | médio | claim verdicts + unsupported spans | claims e trechos exatos com metadata | matching determinístico; verifier sem acesso externo opcional |
| 13 | Moderação/classificação de segurança / `SAFETY_CLASSIFIER` | recall alto para risco imediato sem alarmismo; resposta segura; não usar como diagnóstico | p95 <= 1 s, hard timeout 2,5 s no caminho crítico | por mensagem e, quando necessário, por output | máxima | baixo/médio; cascade controlada | category, level, confidence, action code | mensagem atual; contexto mínimo somente para desambiguação | regras críticas locais + classifier aprovado; sem tool use |

## Matriz B — execução, fallback, budgets e avaliação

| # | Primário lógico | Fallback/cascade | Tokens máximos iniciais (entrada/saída) | Custo | Timeout | Retry | Métricas e evals principais |
|---:|---|---|---:|---:|---:|---|---|
| 1 | `CHAT_BALANCED` | provider alternativo que passou suite; depois resposta segura localizada sem update | 12.000 / 1.200 | C2 | 30 s | 1 em 429/5xx/timeout com jitter e budget restante; nunca repetir write | safety violation, scope accuracy, user-rated usefulness, p50/p95, tokens, custo, fallback rate, schema failure |
| 2 | regras + `INTENT_FAST` | regra conservadora `conversation`; `resources` exige confiança mínima | 1.000 / 80 | C0 | 2 s | no máximo 1 em erro transitório | precision/recall por rota, Resources false route, profile-update leakage, latency |
| 3 | `MOOD_EXTRACTOR_PRECISE` | segundo modelo aprovado ou retry assíncrono; senão nenhum evento | 2.000 / 800 | C1 | 10 s | 1 transitório; 1 schema retry no máximo | precision, recall, FP/FN, sujeito, negação, tempo, intensidade/score inventado, duplicação, edição/exclusão/correção por campo |
| 4 | `SLEEP_EXTRACTOR_PRECISE` | igual à tarefa 3 | 2.000 / 900 | C1 | 10 s | igual à tarefa 3 | mesmas métricas + duração/horário/qualidade/despertares inventados e period-to-nights error |
| 5 | parser determinístico | `TEMPORAL_RESOLVER` aprovado; se ambíguo, `unresolved` | 1.500 / 300 | C0 | 4 s | sem retry de semântica; 1 transitório | exact/range accuracy, timezone/date rollover, correction target, unresolved calibration |
| 6 | regras + `MEMORY_CONSOLIDATOR` | fila de revisão/retry; nunca persistir candidate inválido | 6.000 / 800 | C2 | 20 s | 2 assíncronos com backoff | precision de memória, provenance coverage, consent violation, stale-memory rate, delete propagation |
| 7 | agregador determinístico | texto template localizado; modelo opcional aprovado | 4.000 / 600 | C2 | 12 s | 2 assíncronos; rebuild idempotente | absent-vs-neutral, mixed accuracy, coverage, stale duration, manual precedence, reproducibility |
| 8 | `INSIGHT_REASONER` | sem Insight; mostrar “dados insuficientes” se apropriado | 16.000 / 2.000 | C3 | 50 s | 1 transitório; não reparar claim inseguro automaticamente | evidence sufficiency, usefulness humana, unsupported claim, causality/diagnosis rate, coverage disclosure, custo |
| 9 | regras + `INSIGHT_VALIDATOR` | rejeitar candidate; revisão humana em casos editoriais | 8.000 / 800 | C2 | 20 s | 1 transitório | false accept/false reject, claim-evidence entailment, stale evidence, anchoring, inter-rater agreement |
| 10 | lexical/filter search | semantic -> hybrid -> rerank somente se cada estágio vencer baseline | 1.000 / 120 se houver rewrite | C0/C1 | 5 s | 1 para serviço de busca; circuit breaker | Recall@k, MRR/nDCG, section/authority/version accuracy, zero cross-corpus leak, latency |
| 11 | `GROUNDED_SYNTHESIZER` | resposta extrativa/template ou comunicar insuficiência | 12.000 / 1.500 | C3 | 35 s | 1 transitório; schema retry único | faithfulness, factuality, citation coverage, unsupported claims, safety/personal-treatment violation, latency/cost |
| 12 | regras + `CITATION_VERIFIER` | remover claim não verificado; nunca manter sem fonte | 8.000 / 800 | C2 | 12 s | 1 transitório | claim precision, citation entailment, stale/wrong section, coverage, false accept |
| 13 | regras + `SAFETY_CLASSIFIER` | modelo de escalada aprovado; na indisponibilidade aplicar resposta conservadora definida | 2.000 / 250 | C1 | 2,5 s | no máximo 1 dentro do hard timeout | recall crítico, precision por categoria, over-escalation, latency, availability, locale parity |

## Regras comuns de execução

Toda request ao gateway inclui:

```text
executionId
task
policyVersion
schemaVersion
promptVersion
contextBundleVersion
locale
privacyClass
tokenBudget
costBudget
timeoutMs
idempotencyKey (quando aplicável)
canonicalInput
```

Toda resposta canônica inclui:

```text
status = succeeded | refused | invalid_output | timed_out | provider_error | budget_exceeded
validatedOutput?
usage { inputTokens, outputTokens, totalTokens, source=provider|estimated }
provider/model snapshot (somente application record, nunca domínio)
latencyMs
retryCount
```

Outputs são não confiáveis até passarem por:

1. parsing técnico;
2. schema estrito e versão suportada;
3. validação de enums, datas, limites e campos obrigatórios;
4. policy de domínio da tarefa;
5. ownership/consentimento;
6. idempotência e controle de concorrência.

Recusa, truncamento, finish reason inesperado, tool call não permitido, timeout e JSON reparado devem ter estados distintos. Reparar trailing comma pode ajudar compatibilidade técnica, mas nunca transforma semântica inválida em fato confiável.

## Seleção e canary

1. Registrar baseline com provider/modelo atual.
2. Rodar candidates offline no mesmo dataset e seeds quando disponíveis.
3. Eliminar qualquer candidate que falhe gate crítico, mesmo que média geral seja maior.
4. Comparar custo e latência somente entre candidates seguros.
5. Shadow em tráfego consentido, sem persistência de novos fatos.
6. Revisão humana de amostra minimizada e auditável.
7. Canary 1% -> 5% -> 25% -> 100%, com rollback automático por métrica.

Rollbacks alteram apenas `policyVersion`/configuração; não requerem deploy do domínio. Outputs já persistidos mantêm o execution record original e podem ser invalidados/reprocessados, nunca reatribuídos ao novo modelo.

## Quando combinar chamadas

Combinar Humor e Sono numa única extração pode reduzir custo e latência. Só é aceito se o eval mostrar que:

- a precisão crítica de cada domínio não piora;
- o schema combinado não aumenta invalid output;
- a presença de um domínio não induz campos no outro;
- correções e métricas continuam separáveis.

Conversa, Insights e Recursos não devem compartilhar um call apenas para economizar, pois têm trust domains, contexto e falhas diferentes.
