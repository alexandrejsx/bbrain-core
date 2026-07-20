# ADR-0001 — Workflow determinístico pós-resposta

- Status: Parcialmente superada pelo ADR-0007
- Data: 2026-07-20
- Escopo: Conversa, Humor, Sono e projeções derivadas

## Contexto

O fluxo atual concentra reply, classificação de risco/escopo e sugestão de alteração do perfil no retorno único de `ChatAgent`. A aplicação persiste a conversa e pode atualizar o perfil no mesmo caso de uso. Isso torna a resposta interativa dependente de responsabilidades com requisitos diferentes e não oferece idempotência, proveniência ou uma política explícita para transformar fala em dado pessoal.

Humor e Sono precisam preservar evidência, sujeito, negação, temporalidade, campos ausentes e correções. Insights e resumos não precisam bloquear o reply. Ainda não há fila durável, consumer, inbox/outbox ou coordenação distribuída no backend; introduzir um broker antes de demonstrar necessidade aumentaria a superfície operacional.

## Decisão

Manter o reply conversacional síncrono e iniciar um workflow determinístico pós-resposta. A decisão original previa persistir e recarregar mensagem/resposta; essa parte foi substituída pelo [ADR-0007](./0007-transcript-free-conversation-state.md). O executor local recebe a mensagem atual apenas em memória:

1. concluir a claim técnica sem conteúdo;
2. passar a mensagem atual diretamente ao executor local, sem repository de transcrição;
3. aplicar consentimento e elegibilidade antes de chamar IA;
4. extrair para schema versionado;
5. validar sujeito, negação, temporalidade, suporte de cada campo e duplicação no domínio;
6. persistir por caso de uso com chave idempotente;
7. invalidar e reconstruir projeções derivadas;
8. registrar telemetria sanitizada.

O primeiro executor pode ser local, atrás de `JobPort`, em implantação de uma única réplica. O envelope, a chave idempotente e a semântica de retry devem ser compatíveis com uma fila futura. Adotar fila durável quando houver mais de uma réplica, perda/reprocessamento observado, backlog acima de um minuto ou retry durável como requisito.

## Alternativas consideradas

- Extrair antes de responder: aumenta latência e acopla disponibilidade de uma função derivada ao chat.
- Manter reply e extração no mesmo output: economiza chamada, mas mistura confiança, schemas, orçamento e autorização; rejeitada como contrato persistente.
- Introduzir imediatamente Redis/BullMQ: oferece durabilidade, porém não resolve sozinho idempotência e adiciona operação antes de existir carga medida.
- Multiagente/autonomia aberta: incompatível com a previsibilidade e auditabilidade exigidas para dados emocionais.

## Consequências

Positivas:

- reply tem caminho crítico menor e falhas de extração não o anulam;
- cada etapa pode ter schema, política, timeout, retry e eval próprios;
- reprocessamento e shadow mode tornam a migração reversível.

Custos e riscos:

- consistência é eventual; a UI precisa tolerar pequeno atraso;
- executor local perde trabalho em crash sem uma inbox/outbox durável;
- uma futura topologia com múltiplas réplicas exige fila ou coordenador real.

## Gates e rollback

O workflow só pode escrever após aprovação do dataset de extração, idempotência concorrente, autorização e telemetria. Antes disso opera em `shadow`. Rollback desliga a feature flag de persistência e mantém a conversa atual; registros shadow podem ser descartados porque não são fonte do usuário. Falha persistente acima do SLO, duplicação ou criação de fato não sustentado interrompe automaticamente a escrita derivada.

## Evidência relacionada

Ver [estado atual](../current-state.md), [arquitetura-alvo](../target-architecture.md), [dados de Humor e Sono](../mood-sleep-data.md) e [migração](../migration-plan.md).
