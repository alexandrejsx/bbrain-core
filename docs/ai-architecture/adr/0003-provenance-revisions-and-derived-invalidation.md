# ADR-0003 — Proveniência, revisões e invalidação de derivados

- Status: Aceita, com persistência concretizada pelo ADR-0006
- Data: 2026-07-20
- Escopo: Humor, Sono, memória e Insights

## Contexto

Um fato pessoal extraído de conversa é uma interpretação sujeita a ambiguidade e correção. Sobrescrever valores remove a explicação de origem; apagar somente o registro primário deixa resumos e Insights contraditórios. O modelo atual de perfil não representa evidência por campo, precisão temporal, revisão ou dependência entre dado primário e projeção.

## Decisão

Todo `MoodEvent`, `SleepRecord` ou `SleepPeriodStatement` derivado de IA referencia evidência imutável por `EvidenceRef`, incluindo identificadores, role, intervalo de caracteres quando disponível e hash. Somente mensagem do usuário pode sustentar fato pessoal; texto do assistente, conteúdo recuperado e instruções do sistema não podem.

Criação, edição, correção e exclusão usam revisão otimista. Correção não apaga a história: cria `RecordRevision`; exclusão aplica tombstone e respeita o fluxo de eliminação de dados. O valor atual é uma visão materializada das revisões ativas. Alteração manual do usuário tem precedência sobre reextração automática.

Resumo diário de humor e Insights são derivados. Eles armazenam referências e versões dos inputs. Qualquer input editado, excluído ou substituído os marca `stale` antes de nova leitura; rebuild idempotente produz uma nova versão. Ausência ou cobertura insuficiente gera `insufficient_data`, nunca um valor neutro inventado.

Reprocessamento por novo extractor/schema cria uma proposta ou revisão comparável; não substitui silenciosamente um registro confirmado pelo usuário. A composição concreta da idempotency key e o storage de revisões são refinados no [ADR-0006](./0006-unified-wellbeing-observations-and-stable-projection.md).

## Alternativas consideradas

- Estado mutável sem histórico: implementação curta, mas não auditável nem corrigível com segurança.
- Event sourcing completo para todo o produto: oferece histórico forte, porém é mudança ampla e desnecessária para o slice.
- Guardar apenas o texto integral como proveniência: aumenta exposição de conteúdo sensível e dificulta minimização.
- Recalcular tudo a cada leitura: simples conceitualmente, mas caro e ainda exige versionamento para consistência.

## Consequências

- Usuário pode entender, corrigir e excluir a origem de uma interpretação.
- Projeções não continuam aparentando validade após mudança de evidência.
- Persistência precisa de índices, concorrência otimista, lineage e jobs idempotentes.
- Tombstones e trilha de auditoria devem obedecer retenção e direito de eliminação; não podem virar arquivo eterno de conteúdo sensível.

## Gates e rollback

Writes exigem testes de ownership, concorrência, duplicação, correção, deleção e invalidação. A migração é aditiva. Rollback desabilita novos extractors e leitores voltam às fontes anteriores; coleções novas permanecem isoladas até decisão de retenção. Não há backfill destrutivo automático.

## Evidência relacionada

Ver [contratos de Humor e Sono](../mood-sleep-data.md), [Insights e RAG](../insights-resources-rag.md), [estratégia de evals](../eval-strategy.md) e [plano de migração](../migration-plan.md).
