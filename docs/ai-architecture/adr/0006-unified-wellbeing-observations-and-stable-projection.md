# ADR-0006 — Aggregate unificado e projeção diária estável

- Status: Aceita e implementada no primeiro slice
- Data: 2026-07-20
- Escopo: persistência de Humor/Sono, revisões e resumo diário de Humor
- Refina: ADR-0003

## Contexto

A proposta inicial separava eventos, registros, resumos e revisões em coleções. Para o primeiro slice isso criaria vários repositories, migrations e operações de consistência antes de validar o modelo de produto. Ao mesmo tempo, criar um novo documento derivado a cada correção faria o número de resumos crescer com o histórico e entraria em conflito com a necessidade de um estado diário atual inequívoco.

## Decisão

Usar um aggregate `WellbeingObservation` com três discriminadores (`mood_event`, `mood_daily_summary`, `sleep_record`) e uma coleção aditiva `wellbeing_observations`. Relatos de sono de período são `sleep_record` com temporalidade `period`; não geram noites artificiais.

O aggregate mantém estado corrente, `provenanceHistory`, `revisionHistory` e revisão otimista. O histórico embutido tem limite estrito de 500 entradas; ao atingir o limite a mutação falha e exige migração, sem truncar silenciosamente. Exclusão explícita remove o documento corrente conforme a política de eliminação, em vez de preservar conteúdo sensível indefinidamente num tombstone.

O resumo derivado usa um slot estável por usuário e data local, protegido por índice único parcial. `daily-mood-projector.v2` cria somente com pelo menos dois eventos, limita fontes/descriptores, registra revisões das fontes e atualiza o aggregate existente. Mudança de fonte o marca stale e tenta reconstruí-lo; mesmo diante de falha parcial, o read path não serve uma projeção cujas versões de origem divergiram.

A idempotência de extração reserva um slot por conversa, mensagem fonte, tipo e fingerprint HMAC da evidência validada. A citação literal não é persistida. Entrada manual usa `clientRequestId`; reutilizar o mesmo id com payload diferente é conflito.

## Alternativas consideradas

- Coleção por tipo mais coleção append-only de revisões: separação forte e auditoria independente, mas custo operacional excessivo para o slice.
- Novo resumo imutável por rebuild: preserva cada projeção como documento, mas cresce sem limite e dificulta definir o atual.
- Atualização destrutiva sem histórico: menor código, porém perde explicabilidade e rollback.
- Event sourcing completo: não proporcional ao estágio atual.

## Consequências e limites

- Um mapper centraliza `camelCase` de domínio para `snake_case` persistido.
- A listagem atual tem cap de 500; paginação e consulta por janela temporal precisam anteceder escala.
- O limite de 500 revisões é um gatilho explícito para migrar a história a storage append-only.
- A projeção e a invalidação não são transacionais entre vários documentos; estabilidade do slot, idempotência, stale e validação no read path reduzem, mas não eliminam, a janela de falha.
- A fronteira foi verificada com doubles/unit tests; falta teste de integração contra Mongo real e migração/preflight operacional.

## Rollback e evolução

Desligar `AI_OBSERVATION_EXTRACTION_ENABLED` interrompe writes automáticos sem afetar o chat nem os registros já criados. Endpoints/readers podem ser retirados do rollout mantendo a coleção aditiva. Uma migração futura pode copiar `revision_history` para coleção append-only, validar checksum/contagem e somente depois reduzir o documento corrente; nenhuma remoção ocorre no mesmo deploy da cópia.
