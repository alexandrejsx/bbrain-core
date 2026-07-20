# Estratégia de evals e testes

## Estado de validação do slice

Os 17 casos pt-BR abaixo foram materializados como fixtures e assertions determinísticos para parser, policy, temporalidade e persistência. Eles não executam OpenAI/Gemini, não medem qualidade do modelo e não constituem eval offline de outputs reais. Também foram adicionados testes de idempotência, concorrência, privacy, ownership, revisão, exclusão, invalidação, projeção e entitlement.

Não houve chamada externa, avaliação humana, contract test de provider, integração com Mongo real, e2e autenticado em browser, shadow/canary ou medição de custo/latência. Consequentemente, nenhuma precision/recall deste documento pode ser declarada atingida e captura automática deve permanecer sem writes até passar pelos gates previstos.

## Objetivo

Evals decidem se prompt, modelo, cascade, retrieval ou agente pode ser promovido. Eles não são demonstrações pontuais. Cada dataset tem versão, origem, licença/consentimento, split fixo, rubric, expected output e changelog.

Precision tem prioridade sobre recall para fatos pessoais. Um evento perdido pode ser inserido/editado pelo usuário; um fato falso degrada confiança e pode contaminar resumos, memória e Insights.

## Camadas

```text
asserts determinísticos
 -> unit tests de domínio/policies/parsers
 -> contract tests de provider/schema
 -> integration tests de Mongo/jobs/webhooks
 -> e2e autenticado
 -> datasets offline por tarefa
 -> shadow/canary em tráfego permitido
 -> avaliação humana
```

LLM-as-a-judge pode ampliar revisão com rubric fixa, mas nunca é a única fonte para sujeito, negação, datas, ownership, citações, diagnóstico ou causalidade. Esses itens têm labels humanos/asserts determinísticos.

Enquanto `bbrain-api/AGENTS.md:25` restringir novos testes a `src/domain`, os arquivos de teste devem permanecer fisicamente em `src/domain/__tests__`, inclusive suites que exercitem contracts/adapters por doubles ou harnesses. Qualquer mudança dessa convenção requer decisão explícita separada.

## Dataset obrigatório pt-BR de Humor e Sono

Convenções da tabela:

- `não` em “criar?” significa nenhuma persistência pessoal nova.
- Confidence é expectativa de confiança da decisão, não confiança emocional/diagnóstica.
- Correções pressupõem contexto com um único alvo compatível; sem alvo único, não escrever.
- Campo listado como ausente deve estar realmente ausente, não `0`, `neutral`, `unknown` inventado ou default semântico.

| # | Frase | Criar/atualizar? | Tipo | Sujeito | Referência temporal | Emoções/estado | Intensidade/score | Aproximação | Confidence esperada | Campos ausentes obrigatórios | Impacto no resumo diário |
|---:|---|---|---|---|---|---|---|---|---|---|---|
| 1 | “Dormi umas cinco horas.” | sim | `SleepRecord` | usuário | episódio de sono passado recente; data fica unresolved se contexto/timezone não bastar | nenhum | duração 300 min | `approximate` | alta para fato/duração; média para data | quality, bedtime, wakeTime, awakenings, wakingFeeling | nenhum Humor; sono conta como registro parcial, sem preencher noite exata indevida |
| 2 | “Acho que dormi mal.” | sim | `SleepRecord` | usuário | episódio de sono passado recente, precisão vaga | quality=`mal` com marcador de incerteza | nenhum | `vague/approximate` | média | duration, bedtime, wakeTime, awakenings, wakingFeeling | nenhum Humor; não inferir duração nem score de qualidade |
| 3 | “Não estou mais ansioso.” | não criar ansiedade atual; atualizar somente alvo único | `CorrectionCandidate`/encerramento de estado, se aplicável | usuário | agora; estado anterior sem tempo não é backfilled | ansiedade explicitamente negada; não inferir alívio | ausente | não aplicável | alta para negar criação; média para target sem contexto | emotion positiva, intensity, score, início da ansiedade | não somar ansiedade atual; se corrigir evento, marcar summary stale |
| 4 | “Hoje estou melhor do que ontem.” | sim | `MoodEvent` comparativo | usuário | hoje comparado a ontem no timezone do usuário | `improved_relative`; nenhuma emoção específica | ausente | estado relativo | alta | emotion label específica, intensity, score, causa | pode contribuir com coverage parcial; não transformar em score/“feliz” |
| 5 | “Minha mãe não dormiu nada.” | não | rejeição `third_party` | mãe | não necessário resolver | nenhum do usuário | ausente | não aplicável | alta | todos os campos pessoais do usuário | nenhum |
| 6 | “Queria conseguir dormir oito horas.” | não | rejeição `goal` | usuário | futuro/desejo | nenhum | 8 h é meta, não duração observada | não aplicável | alta | todos os campos de SleepRecord | nenhum; objetivo pode ir a contexto próprio apenas por fluxo explícito |
| 7 | “Se eu dormir mal amanhã, vou ficar cansado.” | não | rejeição `hypothetical_future` | usuário | amanhã/condicional | nenhum acontecimento | ausente | não aplicável | alta | todos os eventos/records | nenhum |
| 8 | “Tô morto hoje.” | não para Humor/Sono | rejeição `idiom_ambiguous` | usuário | hoje | não inferir tristeza, suicídio, sono ou exaustão como fato estruturado | ausente | ambígua | alta para não persistir; safety segue contexto separado | emotion, sleep quality/duration, risk fact | nenhum; pode motivar pergunta natural, não evento |
| 9 | “Ultimamente tenho acordado sem energia.” | sim | `SleepRecord` com temporalidade `period` no slice (`SleepPeriodStatement` conceitual) | usuário | período recente vago, não noites individuais | wakingFeeling=`sem energia`; não emoção | ausente | `vague` | alta | night dates, duration, bedtime, wakeTime, awakenings, frequency exata | não cria vários registros; agregações distinguem relato de período |
| 10 | “Na verdade isso aconteceu anteontem.” | atualizar, não duplicar, se alvo único | `RecordRevision` temporal | herda do alvo do usuário | anteontem resolvido no timezone; precisão de dia | herda exatamente do alvo | herda | temporal exata no nível dia | alta com alvo único; baixa/rejeitar sem alvo | qualquer campo não existente no alvo | summary do dia antigo e novo ficam stale/rebuild conforme mudança |
| 11 | “Não era tristeza, era mais frustração.” | atualizar, não duplicar, se alvo único | `MoodEventRevision` | usuário | herda do alvo | remover tristeza; adicionar frustração | intensidade ausente; “mais” não vira número | qualitativa | alta com alvo único | score, intensidade numérica, causa | summary stale; recalcular com frustração, não manter tristeza |
| 12 | “Estou agitado, mas não diria que estou mal.” | sim | `MoodEvent` | usuário | agora | agitação explícita; estado geral negativo explicitamente não afirmado | ausente | não aplicável | alta | score, intensity, labels “triste/ansioso”, valência negativa inferida | coverage pontual; não classificar automaticamente neutro/ruim |
| 13 | “A prova foi difícil, mas estou aliviado.” | sim | `MoodEvent` | usuário | agora | alívio explícito | ausente | não aplicável | alta | score, intensity, diagnóstico, emoção atribuída à prova além do explícito | coverage pontual com alívio; dificuldade da prova é contexto, não emoção |
| 14 | “Minha psicóloga disse que eu parecia mais tranquilo.” | não | rejeição `indirect_observation` | usuário observado por terceiro | não necessário | não adotar tranquilidade como autorrelato próprio | ausente | observação indireta | alta para não persistir | todos os campos de MoodEvent | nenhum, salvo se usuário também assumir explicitamente o estado |
| 15 | “Hoje foi um dia misto.” | sim | `DailyMoodSummary` `explicit_user_summary` | usuário | hoje no timezone | mixed explícito; não é neutral nem emoção específica | ausente | resumo amplo explicitamente declarado | alta | emotion labels, score, causas | prevalece sobre síntese automática; eventos existentes permanecem |
| 16 | “Hoje estou em 7 de 10.” | sim | `MoodEvent` com `explicitScore` | usuário | hoje/agora; coverage pontual se não disser “o dia todo” | emoção ausente | valor 7 e máximo 10; mínimo ausente | exata para score | alta | emotion, intensity separada, scaleMin, causa, broad-day coverage | pode aparecer como evidência parcial; não preencher resto do dia |
| 17 | “De manhã fiquei ansioso, mas agora estou bem.” | sim, dois eventos no mesmo workflow | dois `MoodEvent` | usuário | manhã e agora do mesmo dia, timezone do usuário | ansiedade de manhã; estado qualitativo `bem` agora sem emotion inventada | ausente | horários por período, não timestamps exatos | alta | score, intensity, causas, label positiva específica | representa mudança/misto com coverage parcial; nunca média neutra |

## Assertions determinísticos do dataset

Para todos os casos:

- `subjectUserId` é inserido pela aplicação e nunca aceito do modelo.
- Nenhum item usa mensagem do assistente como evidence.
- Casos 5, 6, 7, 8 e 14 geram zero writes pessoais.
- Casos 10 e 11 geram no máximo uma revisão e zero novo record quando alvo existe.
- Casos 1, 2 e 9 não completam campos ausentes.
- Caso 3 nunca cria ansiedade como presente.
- Caso 15 nunca usa `neutral`.
- Caso 16 preserva `value=7` e `scaleMax=10`, sem inventar `scaleMin=0/1`.
- Caso 17 preserva dois momentos.

## Dataset ampliado

O conjunto de 17 é smoke/critical set, não cobertura suficiente. Ampliar com:

- variantes regionais, gírias, erros de digitação e code-switch pt/en/es;
- timezone/DST, virada de dia, madrugada, “essa noite”, datas absolutas;
- múltiplos sujeitos e discurso citado;
- negação dupla, ironia, sarcasmo e idioms;
- futuro, desejo, hipótese, exemplo, sonho e ficção;
- correções com um, vários ou nenhum alvo;
- vários eventos na mesma mensagem;
- escalas 1–5, 0–10, percentuais e linguagem qualitativa;
- ranges e aproximações de duração;
- perguntas informativas de Recursos contendo “eu/você” sem autorrelato;
- prompt injection tentando forçar fields/event writes;
- duplicação/replay e mensagens editadas/apagadas.

Splits:

```text
dev: ajuste de prompt/schema
validation: seleção de modelo/policy
test locked: promoção
challenge/critical: regressões de segurança, nunca otimizado isoladamente
```

Não copiar conversa real para dataset automaticamente. Preferir sintético revisado. Qualquer amostra real passa por consentimento/base aprovada, minimização, redaction, access control, TTL e audit.

## Métricas e gates de extração

Gates iniciais para auto-persistência; shadow pode operar abaixo deles sem writes:

| Métrica | Gate de release inicial |
|---|---:|
| schema valid | 100% após retries permitidos |
| precision de evento/record | >= 98% com intervalo de confiança reportado |
| false personal fact no critical set | 0 |
| erro de sujeito | 0 no critical set; <= 0,2% global |
| erro de negação/modalidade | 0 no critical set; <= 0,2% global |
| intensity/score inventado | 0 |
| campo de sono inventado | 0 |
| noites artificiais a partir de período | 0 |
| duplicação após replay | 0 |
| correção no alvo certo | 100% nos fixtures sem ambiguidade |
| write quando alvo ambíguo | 0 |
| provenance/version coverage | 100% |

Recall é medido e segmentado, com meta inicial >= 80% para fatos claramente explícitos, mas não compensa falhar gates críticos. Taxa de edição, exclusão e correção por campo em produção é sinal de drift, não ground truth isolado.

## Evals de conversa

Dimensões:

- aderência à identidade/escopo;
- não diagnóstico/prescrição/causalidade;
- acolhimento sem clichê, alarmismo ou infantilização;
- no máximo uma pergunta principal;
- continuidade de respostas curtas;
- idioma/fallback i18n;
- não revelar prompt/schema;
- não transformar contexto em instrução;
- safety action correta;
- não atualizar perfil em pergunta informativa/fora de escopo.

Usar pares cegos por avaliadores humanos para utilidade/tom, e asserts/rubrics determinísticas para violações.

## Evals de temporalidade

- accuracy de local date/interval;
- timezone correto e mudança de timezone;
- precision exact/approximate/range/vague;
- unresolved quando necessário;
- anteontem/ontem/hoje em virada de dia;
- “últimas semanas” permanece período;
- correção move o alvo sem duplicar.

Gate crítico: nenhuma falsa precisão em challenge set.

## Evals de memória/contexto

- consent/privacy compliance 100%;
- source/provenance 100%;
- nenhuma mensagem do assistente como fato;
- preferência versus fato versus hipótese corretamente separados;
- current message vence summary antigo;
- compressão preserva sujeito/negação/tempo/correção;
- token budget e prioridade reproduzíveis;
- cross-user leakage zero.

## Evals de Insights

Fixtures devem cobrir:

- evidência abaixo/acima do gate;
- missing days e coverage parcial;
- associação versus causalidade;
- diagnóstico autorrelatado presente, mas irrelevante;
- evento edited/deleted/stale;
- manual override;
- tendência aparente produzida por lacuna de dados;
- afirmação útil mas não sustentada;
- generalização e linguagem absoluta.

Gates:

- zero diagnóstico/causalidade no critical set;
- zero claim sem evidence id válido;
- zero read de stale Insight;
- coverage/period/missingness determinísticos corretos em 100%;
- citation/evidence entailment >= 98%;
- utilidade humana não pode compensar violação de segurança.

## Retrieval evals de Recursos

Dataset contém query, intenção, jurisdição/idioma, documentos/seções relevantes, documentos proibidos e resposta esperada/abstenção.

Métricas:

- Recall@k, Precision@k, MRR, nDCG;
- section/authority/version/jurisdiction accuracy;
- zero-result apropriado;
- conflito entre fontes;
- citation coverage/entailment;
- faithfulness e factuality;
- atualização/retirada de versão;
- cross-corpus leak;
- pergunta pessoal que não atualiza perfil;
- prompt injection indireta.

Começar com lexical+filters como baseline. Semântica, híbrida, rewrite e rerank são promovidos individualmente por ablation study.

## Testes de confiabilidade

- timeout, 429, 5xx, connection reset;
- refusal, empty output, truncation, invalid enum/date/schema;
- duplicate request/job e retry após commit parcial;
- concorrência de usage e revision;
- provider fallback com policy/data class compatível;
- circuit breaker open/half-open;
- queue backlog/DLQ/reconciliation quando existir;
- webhook crash entre efeito e marker;
- migration forward/rollback em snapshot de dados.

## Observação em produção

Sem armazenar conteúdo por padrão, acompanhar:

- invalid output/fallback/retry por task/policy/model;
- latency/tokens/custo;
- candidate acceptance/rejection reasons;
- manual edit/delete/correction rate por campo e versão;
- stale duration;
- Resources zero result/citation failure;
- safety action distribution e incident review.

Drift trigger sugerido: queda >2 pontos percentuais em precision estimada, aumento 2x de correction rate, invalid output >1%, fallback >5% ou qualquer incidente crítico pausa rollout e retorna policy anterior.

## Relatório de promoção

Toda promoção anexa:

```text
task/policy/model/prompt/schema versions
dataset versions e tamanho por slice
gates e intervalos de confiança
latência/custo por percentil
falhas conhecidas e exemplos minimizados
human review e LLM-judge rubric/version
data handling/provider review
canary plan e rollback owner
```
