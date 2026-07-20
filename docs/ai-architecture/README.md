# Arquitetura de IA do BBrain

Status: auditoria arquitetural e fundação vertical implementada sobre o código existente em 20 de julho de 2026.

Este diretório registra o reconhecimento técnico, a arquitetura-alvo, o slice entregue e os gates de evolução da IA do BBrain. Cada documento distingue o que foi observado, implementado e apenas planejado. A fotografia inicial de backend usada na análise corresponde ao commit `58c2086` de `bbrain-api`; referências de linha da auditoria devem ser lidas nessa fotografia, pois a implementação posterior as deslocou.

## Resumo executivo

O backend já possuía uma borda razoável entre aplicação e provedores, mas persistia transcrições e reunia conversa, classificação e uma proposta de mutação do perfil na mesma resposta do modelo. O slice entregue mantém a resposta síncrona, remove `profileUpdate` e substitui histórico literal por estado estruturado com TTL e ledger HMAC sem conteúdo. A extração pós-resposta de Humor/Sono tem contrato versionado, adapters OpenAI/Gemini/noop, fallback único, parser estrito, validação de domínio, proveniência por fingerprint, idempotência, revisão, CRUD autenticado e projeção diária determinística.

A implementação preserva a única identidade conversacional BBrain e não altera layout, CSS, navegação ou cria páginas/check-ins. O frontend envia `clientMessageId` estável e passa a ler Humor, Sono e o entitlement de Insights pelos endpoints reais. A resposta e a mensagem não são persistidas; uma resposta perdida não pode ser reproduzida. Execução e persistência de observações têm flags independentes: ambas permanecem desligadas por padrão (`AI_OBSERVATION_EXTRACTION_ENABLED=false` e `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED=false`) até existirem evals reais de provider, integração Mongo e gates operacionais. Ligar somente a primeira executa shadow sem writes.

Humor usa eventos emocionais como evidência primária e um slot diário derivado estável e revisável. Sono aceita registros parciais e relatos de período sem inventar noites. Correções conversacionais, edição manual e exclusão mantêm cadeia de revisão; mudança em fonte invalida/reconstrói resumo e o read path rejeita projeção com versões divergentes. Ausência nunca vira neutralidade.

Insights agora tem autorização Pro efetiva no backend e resposta honesta `insufficient_data`; geração longitudinal ainda não foi implementada. Histórico básico, proveniência sanitizada, edição, exclusão e inserção manual são independentes do plano no backend; os controles manuais de frontend foram adiados porque o layout atual não os oferece com segurança. Recursos/RAG permanece estratégia documentada, separado de dados pessoais e sem writes de memória/perfil.

Não se recomenda multiagente, GraphRAG, knowledge graph, memória vetorial pessoal, ou uma fila distribuída apenas por antecipação. Cada item ganha um gatilho objetivo de reavaliação. Também não se recomenda trocar modelos concretos antes de um benchmark reproduzível demonstrar qualidade, custo e latência adequados para cada tarefa.

## Decisões aplicadas ao primeiro slice

- Manter conversa síncrona e não alongar seu caminho crítico com análise longitudinal.
- Separar resposta conversacional de extração de fatos pessoais.
- Aceitar somente mensagem de usuário como evidência de fato pessoal.
- Implementar primeiro um workflow determinístico de Humor/Sono com schemas canônicos, validação e proveniência.
- Fazer writes por casos de uso; nenhum modelo ou adapter escreve no MongoDB.
- Aplicar as preferências de privacidade antes de construir contexto ou persistir derivados.
- Introduzir aliases de capacidade e política de seleção; nomes concretos de modelos permanecem em configuração de infraestrutura.
- Executar processamento pós-resposta local e desligado por padrão; adotar fila/outbox durável quando volume, reconciliação ou múltiplas réplicas exigirem.
- Manter Recursos, material editorial e dados pessoais em trust domains e índices distintos.

## Índice dos entregáveis

- [Estado atual, contextos, dependências e riscos](./current-state.md)
- [Arquitetura-alvo e fluxos](./target-architecture.md)
- [Política das 13 tarefas de IA](./ai-task-policy.md)
- [Prompts, contexto e memória](./prompt-context-memory.md)
- [Dados e workflows de Humor e Sono](./mood-sleep-data.md)
- [Insights, Recursos e RAG](./insights-resources-rag.md)
- [Segurança, privacidade e observabilidade](./security-observability.md)
- [Estratégia de evals e dataset pt-BR](./eval-strategy.md)
- [Plano de migração reversível](./migration-plan.md)
- [Relatório da implementação e validação](./implementation-report.md)
- [ADRs](./adr/)

## Cobertura dos 20 entregáveis

| # | Entregável | Evidência |
|---:|---|---|
| 1 | relatório da arquitetura atual | [current-state.md](./current-state.md) |
| 2 | bounded contexts e dependências | [current-state.md](./current-state.md) e [target-architecture.md](./target-architecture.md) |
| 3 | problemas e riscos | [current-state.md](./current-state.md) e [implementation-report.md](./implementation-report.md) |
| 4 | arquitetura-alvo | [target-architecture.md](./target-architecture.md) |
| 5 | diagramas textuais dos fluxos | [target-architecture.md](./target-architecture.md) |
| 6 | tarefas de IA e políticas de modelo | [ai-task-policy.md](./ai-task-policy.md) |
| 7 | prompts e contexto | [prompt-context-memory.md](./prompt-context-memory.md) |
| 8 | memória | [prompt-context-memory.md](./prompt-context-memory.md) |
| 9 | Humor e Sono | [mood-sleep-data.md](./mood-sleep-data.md) |
| 10 | Insights premium | [insights-resources-rag.md](./insights-resources-rag.md) |
| 11 | Recursos e RAG | [insights-resources-rag.md](./insights-resources-rag.md) |
| 12 | segurança e privacidade | [security-observability.md](./security-observability.md) |
| 13 | observabilidade | [security-observability.md](./security-observability.md) |
| 14 | evals | [eval-strategy.md](./eval-strategy.md) |
| 15 | migração | [migration-plan.md](./migration-plan.md) |
| 16 | ADRs | [adr/](./adr/) |
| 17 | implementação incremental | [implementation-report.md](./implementation-report.md) |
| 18 | testes novos/atualizados | [implementation-report.md](./implementation-report.md) e [eval-strategy.md](./eval-strategy.md) |
| 19 | relatório final | [implementation-report.md](./implementation-report.md) |

## Baseline reproduzível

Executado em `bbrain-api` na fotografia auditada:

```bash
pnpm test --runInBand
pnpm run lint
pnpm exec tsc --noEmit --pretty false
pnpm exec jest --runInBand --coverage \
  --collectCoverageFrom='src/**/*.ts' \
  --collectCoverageFrom='!src/**/*.spec.ts' \
  --coverageDirectory=/tmp/bbrain-api-coverage
```

Baseline anterior ao slice: 15 suites e 95 testes passaram; lint e typecheck passaram. A cobertura global era 33,61% de statements, 27,48% de branches, 38,65% de functions e 34,78% de lines. No snapshot final, a suíte completa passou com 31 suites e 232 testes; cobertura global ficou em 49,12% de statements, 49,58% de branches, 51,60% de functions e 50,15% de lines. Os demais comandos ficam registrados no [relatório da implementação](./implementation-report.md). O comando `pnpm test -- --runInBand` não deve ser usado: o separador extra faz o Jest interpretar `--runInBand` como filtro de arquivo.

Antes de qualquer release, acrescentar ao baseline:

```bash
pnpm install --frozen-lockfile
pnpm build
pnpm audit --prod
```

O audit de dependências exige acesso ao registry e deve ser armazenado como artefato de CI; não foi executado nesta auditoria local.

## Fontes externas atuais

- OpenAI recomenda Responses API para novos projetos e documenta `text.format` para Structured Outputs: [Migrate to the Responses API](https://developers.openai.com/api/docs/guides/migrate-to-responses).
- OpenAI documenta retenção e controles de dados por endpoint: [Data controls in the OpenAI platform](https://developers.openai.com/api/docs/guides/your-data).
- Google informa que Interactions API está GA em `v1` e que `v1beta` pode mudar: [Gemini API versions](https://ai.google.dev/gemini-api/docs/api-versions).
- A página de structured output usada pelo adapter atual chama GenerateContent de API anterior/legacy e recomenda Interactions: [Gemini GenerateContent structured outputs](https://ai.google.dev/gemini-api/docs/generate-content/structured-output?hl=en).
- O modelo configurado `gemini-3.5-flash` não tinha shutdown anunciado na data da auditoria: [Gemini deprecations](https://ai.google.dev/gemini-api/docs/deprecations).
- Node.js 20, usado no Dockerfile e no ambiente auditado, está EOL: [Node.js releases](https://nodejs.org/en/about/previous-releases) e [Node.js EOL](https://nodejs.org/en/about/eol).

## Regra de atualização

Alterações em contratos canônicos, políticas de modelo, eventos, prompts, schemas, privacidade, retenção, corpus ou fluxos assíncronos devem atualizar estes documentos e um ADR quando mudarem uma decisão. Métricas observadas devem substituir hipóteses; números de orçamento aqui são limites iniciais de engenharia, não promessa comercial.
