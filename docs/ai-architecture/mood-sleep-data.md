# Estratégia de dados de Humor e Sono

## Estado do slice em 20 de julho de 2026

A implementação escolheu um aggregate unificado `WellbeingObservation` e uma coleção aditiva `wellbeing_observations`. Os três `kind` atuais são `mood_event`, `mood_daily_summary` e `sleep_record`; um relato de período de sono é um `sleep_record` com referência temporal `period`, sem materializar noites inexistentes. As interfaces mais detalhadas abaixo continuam sendo vocabulário conceitual/alvo e não devem ser confundidas com coleções já disponíveis.

O documento corrente contém `provenance_history`, `revision_history`, `revision` e o estado canônico. O histórico embutido foi escolhido para o slice por simplicidade e é limitado a 500 revisões; ao atingir o limite, novas mutações falham com indicação de migração em vez de apagar história. Uma coleção append-only passa a ser o próximo passo quando esse limite, auditoria independente ou crescimento de documento se tornarem relevantes. A decisão aplicada está em [ADR-0006](./adr/0006-unified-wellbeing-observations-and-stable-projection.md).

A projeção derivada de Humor usa um slot estável por usuário/dia (`daily-mood-projector.v2`), no mínimo dois eventos, no máximo 12 descritores e 200 fontes. Ela registra ids e revisões das fontes, é atualizada no próprio aggregate e mantém as versões anteriores no histórico; o read path oculta um derivado `current` se as revisões não corresponderem. A listagem Mongo atual é deliberadamente limitada a 500 observações, o que ainda exige paginação/reconciliação antes de escala.

## Decisão central

Não promover o esqueleto `check-in` atual ao MVP. `MoodCheckin` exige um `MoodLevel` numérico e `SleepCheckin` exige `SleepDuration` (`bbrain-api/src/domain/check-in/entities/check-in.entities.ts:32-73`), enquanto as premissas exigem eventos emocionais, registros parciais e ausência sem preenchimento. Esse código pode permanecer como esqueleto até uma migração deliberada; não é fonte de verdade do novo histórico.

O primeiro slice adota um histórico de bem-estar orientado a evidências:

```text
MoodEvent                   evidência emocional primária
SleepRecord                 uma noite/momento específico, parcial
SleepPeriodStatement        relato agregado de período, sem fabricar noites
DailyMoodSummary            projeção derivada ou resumo explícito/manual
RecordRevision              correção, edição, supersession ou tombstone
EvidenceReference           ligação à mensagem/ação de origem
```

Os nomes finais podem acompanhar a linguagem ubíqua escolhida no código; as distinções são invariantes.

## Tipos compartilhados conceituais

```ts
type Origin = 'conversation_extraction' | 'manual_entry' | 'manual_edit';
type RecordStatus = 'active' | 'stale' | 'superseded' | 'deleted';
type Precision = 'exact' | 'approximate' | 'range' | 'vague' | 'unknown';
type TemporalKind = 'instant' | 'day' | 'specific_night' | 'interval' | 'period';

interface EvidenceReference {
  sourceType: 'user_message' | 'manual_action';
  sourceId: string;
  sourceRevision?: number;
  conversationId?: string;
  actor: 'user';
  observedAt: string;
}

interface ExtractionProvenance {
  executionId: string;
  task: 'mood_extraction' | 'sleep_extraction';
  policyVersion: string;
  promptVersion: string;
  schemaVersion: string;
  provider: string;
  modelSnapshot: string;
  extractedAt: string;
}

interface TemporalReference {
  kind: TemporalKind;
  startsAt?: string;
  endsAt?: string;
  localDate?: string;
  timezone: string;
  precision: Precision;
  expression?: string; // opcional e minimizado; não necessário em logs
  unresolvedReason?: string;
}
```

`EvidenceReference` identifica a fonte sem duplicar texto sensível. No MVP, a citação literal existe somente na validação em memória e a persistência usa fingerprint HMAC; explicabilidade pública usa campos estruturados, origem e confiança, sem ecoar a mensagem.

## MoodEvent

```ts
interface MoodEvent {
  id: string;
  userId: string;
  temporal: TemporalReference;
  emotions: Array<{
    label: string;
    assertedByUser: true;
    intensity?: {
      value: number | string;
      scale?: string;
      precision: Precision;
      explicitlyReported: true;
    };
  }>;
  explicitScore?: {
    value: number;
    scaleMin?: number;
    scaleMax: number;
    explicitlyReported: true;
  };
  qualitativeState?: string;
  extractionConfidence?: number;
  origin: Origin;
  evidence: EvidenceReference[];
  provenance?: ExtractionProvenance;
  revision: number;
  status: RecordStatus;
  supersedesId?: string;
  createdAt: string;
  updatedAt: string;
}
```

Invariantes:

- `userId` vem do contexto autenticado, nunca do modelo.
- Toda extração automática tem pelo menos uma evidência `user_message` com `actor=user`.
- `emotions` pode estar vazio quando existe apenas estado comparativo ou score explícito; isso não autoriza inventar label.
- Intensidade só existe se sustentada na mensagem. Nunca mapear emoção automaticamente para escala.
- Score numérico só existe quando o usuário o fornece e inclui pelo menos o limite superior da escala; em “7/10”, `scaleMin` permanece ausente porque zero ou um não foi declarado.
- “Misto” não é “neutro”; pode ser resumo explícito ou coexistência de eventos.
- Confiança de extração não é cobertura do dia.
- Diagnóstico autorrelatado nunca é label emocional nem explicação causal.
- Evento sobre terceiro, futuro, desejo, hipótese, ficção ou pergunta informativa é rejeitado.
- Negação impede criar o estado negado. Uma declaração de término pode corrigir/encerrar evento anterior somente com alvo sustentado.

## SleepRecord

```ts
interface SleepRecord {
  id: string;
  userId: string;
  temporal: TemporalReference;
  durationMinutes?: { value: number; precision: Precision };
  bedtime?: { localTime: string; precision: Precision };
  wakeTime?: { localTime: string; precision: Precision };
  quality?: { value: string; precision: Precision; explicitlyReported: true };
  awakenings?: { count?: number; description?: string; precision: Precision };
  wakingFeeling?: { value: string; explicitlyReported: true };
  extractionConfidence?: number;
  origin: Origin;
  evidence: EvidenceReference[];
  provenance?: ExtractionProvenance;
  revision: number;
  status: RecordStatus;
  supersedesId?: string;
  createdAt: string;
  updatedAt: string;
}
```

Invariantes:

- Nenhum dos campos clínico-descritivos é obrigatório isoladamente; exige-se apenas uma afirmação de sono válida e evidência.
- `durationMinutes` preserva `approximate/range`; “umas cinco horas” não vira exato.
- Bedtime, wake time, awakenings, quality e waking feeling permanecem ausentes se não mencionados.
- `specific_night` pode ter `localDate` sem timestamps exatos.
- Timezone vem do perfil e é registrado junto à resolução; mudança posterior de timezone não reinterpreta silenciosamente o passado.
- “quero dormir oito horas” é goal; “se eu dormir mal” é hipótese. Nenhum cria `SleepRecord`.
- Pergunta sobre sono/medicamento não cria fato pessoal.

## SleepPeriodStatement

```ts
interface SleepPeriodStatement {
  id: string;
  userId: string;
  temporal: TemporalReference & { kind: 'period' };
  reportedPattern: string;
  wakingFeeling?: string;
  quality?: string;
  frequency?: { value: string; precision: Precision };
  origin: Origin;
  evidence: EvidenceReference[];
  provenance?: ExtractionProvenance;
  revision: number;
  status: RecordStatus;
}
```

“Tenho dormido mal nas últimas semanas” cria no máximo um statement de período. Ele não é materializado como 14/21 noites, não entra em média de duração e não preenche dias ausentes.

## DailyMoodSummary

```ts
interface DailyMoodSummary {
  id: string;
  userId: string;
  localDate: string;
  timezone: string;
  summaryKind: 'derived' | 'explicit_user_summary' | 'manual_override';
  state: 'current' | 'stale' | 'superseded' | 'insufficient_data';
  description?: string;
  labels?: string[];
  explicitScore?: { value: number; scaleMin?: number; scaleMax: number };
  coverage: 'point_in_time' | 'partial_day' | 'broad_day' | 'unknown';
  sourceEventIds: string[];
  sourceRevisions: number[];
  projectionVersion: string;
  generatedAt?: string;
  manuallyEditedAt?: string;
  revision: number;
}
```

Regras de precedência:

```text
manual_override
> explicit_user_summary
> derived
```

- Sem evidência suficiente: `insufficient_data`, sem label/score fabricado.
- Dois momentos diferentes podem resultar em `mixed`/mudança, nunca média neutra automática.
- Um evento isolado normalmente tem coverage `point_in_time` ou `partial_day`, ainda que sua extração tenha alta confiança.
- Resumo manual não destrói eventos; ele apenas prevalece na apresentação.

## Revisões, edição e exclusão

Não sobrescrever silenciosamente o documento em fluxo sensível. Duas opções foram consideradas:

1. aggregate com histórico de revisões embutido; simples no MVP se o histórico for curto;
2. coleção append-only `wellbeing_record_revisions`; melhor para auditoria e concorrência.

Decisão do slice: documento corrente + revisões embutidas, validadas e limitadas a 500. A coleção append-only permanece evolução planejada e obrigatória antes de ultrapassar esse limite ou de exigir auditoria independente. A API usa `expectedRevision`; uma mutação concorrente falha em vez de sobrescrever silenciosamente.

```ts
interface RecordRevision {
  recordId: string;
  recordType: 'mood_event' | 'sleep_record' | 'sleep_period_statement' | 'daily_mood_summary';
  fromRevision: number;
  toRevision: number;
  operation: 'create' | 'manual_edit' | 'source_correction' | 'supersede' | 'delete';
  actor: 'user' | 'system';
  actorUserId?: string;
  changedFields: string[];
  reasonCode?: string;
  sourceId?: string;
  occurredAt: string;
}
```

Para não duplicar dados sensíveis, a revisão guarda patch/campos necessários em storage protegido e a trilha operacional guarda apenas metadados.

Commands usam optimistic concurrency:

```text
EditMoodEvent(recordId, expectedRevision, patch)
DeleteMoodEvent(recordId, expectedRevision)
CreateManualMoodEvent(...)
EditSleepRecord(recordId, expectedRevision, patch)
DeleteSleepRecord(recordId, expectedRevision)
CreateManualSleepRecord(...)
```

Um tombstone preserva que a evidência foi removida sem manter o conteúdo além da retenção permitida. APIs nunca permitem selecionar outro `userId`; ownership deriva do JWT.

## Correções conversacionais

Correção automática precisa de alvo. O extractor pode devolver:

```ts
interface CorrectionCandidate {
  operation: 'correct_existing';
  targetHints: {
    recordType: string;
    sourceMessageId?: string;
    relativePosition?: 'previous_related_event';
    previousTemporal?: TemporalReference;
  };
  patch: unknown;
  confidence: number;
}
```

A aplicação busca um conjunto pequeno de registros do próprio usuário e só aplica se houver alvo único e policy aprovar. Ambiguidade vira `needs_review`/nenhuma alteração. “Na verdade foi anteontem” atualiza a referência do evento correto; não cria evento adicional. “Não era tristeza, era frustração” cria revisão que remove tristeza e adiciona frustração mantendo a fonte/cadeia.

## Invalidação de derivados

```text
MoodEventChanged(recordId, revision)
  -> DailyMoodSummary do localDate = stale
  -> Insights com evidenceId/revision anterior = stale
  -> memória/padrões derivados relacionados = review_required

SleepRecordChanged(recordId, revision)
  -> agregações de sono na janela = stale
  -> Insights dependentes = stale
```

O handler é idempotente. Até haver infraestrutura durável, o read path também verifica revisions para impedir exibição de derivado stale mesmo se um evento de invalidação se perder.

## Workflow de extração

```text
1. persistir a troca conversacional idempotente e responder ao usuário
2. aplicar feature/privacy/account policy antes de chamar o extractor
3. serializar captura por usuário no processo atual
4. executar extração estruturada com primary e no máximo um fallback
5. validar JSON Schema, parser estrito e regras de domínio
6. reaplicar privacy/account policy depois da chamada externa
7. deduplicar candidates e reservar o slot idempotente na persistência
8. criar/corrigir a observação com proveniência e contabilizar tokens auxiliares
9. invalidar e reconstruir a projeção diária quando aplicável
```

Chave inicial:

```text
conversation:{conversationId}:{sourceMessageId}:{kind}:{evidenceFingerprintHmac[0..31]}:v4
```

O hash é identificador técnico, não substitui criptografia nem torna conteúdo anônimo.

## Persistência aditiva implementada

- `wellbeing_observations`, com shape persistido em `snake_case` por mapper;
- estado, proveniência e revisão no mesmo aggregate;
- nenhuma coleção vetorial pessoal;
- `ai_executions`, inbox/outbox e fila durável permanecem futuras.

Índices iniciais, implantados por migração explícita:

```text
unique(user_id, idempotency_key)
(user_id, kind, created_at desc)
(user_id, kind, data.source_observation_ids)
unique parcial(user_id, kind, temporal_reference.local_date)
  quando kind=mood_daily_summary, summary_source=derived e status=current
```

Não criar índice vetorial pessoal neste slice.

## Contratos de leitura

Read models devem distinguir:

```text
source = automatic | manual
status/revision
coverage
approximation
provenance disponível ao usuário em linguagem simples
stale/processing state
```

Campos novos devem ser opcionais nos contratos frontend durante rollout. O backend suporta CRUD e provenance mesmo quando o layout atual não consegue exibir todos os detalhes; nenhuma nova página/CSS é exigida por esta arquitetura.

## Retenção e exclusão

- Exclusão de conta inclui todos os records, revisions permitidas, summaries e executions vinculados; billing segue policy legal de retenção/anonymização separada.
- Exclusão de conta bloqueia e drena capturas locais antes de remover `wellbeing_observations`; coordenação distribuída entre réplicas ainda requer fila/lease durável.
- Exclusão de mensagem fonte deve invalidar/excluir derivados, conforme escolha de produto e consentimento; esse vínculo ainda não tem orchestration completa.
- Samples para eval não são copiados automaticamente do produto; exigem pipeline separado, minimizado e auditado.
- Proveniência não justifica retenção indefinida do conteúdo fonte.

## Critérios de aceite do slice

- Schema e domain validation rejeitam terceiro, futuro, desejo, ficção e negação.
- Registro parcial de sono funciona sem duração.
- Score emocional só aparece quando explícito.
- Reprocessamento idêntico não duplica.
- Edição/exclusão manual usa ownership e revision.
- Correção atualiza o alvo ou não faz write se ambígua.
- Summary não preenche ausência e respeita precedência manual.
- Alteração invalida todo derivado referenciado.
- Provenance e versões são recuperáveis, mas conteúdo não aparece em logs.

Os critérios acima são cobertos por testes determinísticos. Não houve chamada a provider real, avaliação humana, integration test Mongo nem canary; por isso a captura automática continua desligada por padrão.
