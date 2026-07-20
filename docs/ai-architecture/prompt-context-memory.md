# Prompts, contexto e memória

## Estado implementado do primeiro slice

O extractor de Humor/Sono tem prompt, provider schema e parser explicitamente versionados em v3; outputs são tratados como não confiáveis e passam por validação estrutural e de domínio antes do repository. Apenas a mensagem atual do usuário é evidência de novo fato; até dez observações estruturadas recentes podem entrar somente como contexto de correção quando há marcador determinístico. OpenAI e Gemini recebem `store: false`. A citação literal é transitória e a persistência usa `evidenceFingerprint` HMAC.

O context builder da conversa não consulta transcrições. Ele respeita privacy flags e pode carregar apenas `ConversationState` estruturado e não expirado. Diagnósticos, medicação, suporte profissional e `current_context_summary` não entram no contexto geral. O contrato não possui mais `profileUpdate`; o modelo propõe apenas `conversationStateUpdate`, validado e temporário. Uma memória durável tipada com proveniência não foi implementada.

## Estado atual

O prompt principal está em um registry de infraestrutura, ainda sem versão explícita. O renderer cria:

1. system prompt do BBrain;
2. idioma e adaptação de tom;
3. um segundo bloco `USER_CONTEXT` rotulado como dado, não instrução;
4. um `ConversationState` estruturado quando permitido;
5. a mensagem atual.

Nenhuma mensagem anterior do usuário ou do BBrain é carregada do banco.

Na fotografia auditada, o context builder limitava listas e comprimentos, mas enviava diagnósticos autorrelatados, medicação, suporte profissional e `current_context_summary` sempre que o perfil reflexivo existia: `bbrain-api/src/use-cases/conversation/conversation-agent-context-builder.service.ts:71-92`. Os privacy flags não participavam dessa decisão; o slice corrigiu essa lacuna conforme o estado acima.

O perfil reflexivo mistura categorias com ciclos de vida distintos:

- preferência: `preferred_tone`;
- objetivos: `analysis_goals`;
- hipóteses/padrões derivados: `recurring_themes`, `emotional_patterns`;
- fatos/estratégias: `routine_notes`, `helpful_strategies`, `unhelpful_strategies`, `boundaries`;
- saúde autorrelatada: `reported_formal_diagnoses`, `reported_medication`, `professional_support`;
- resumo temporário: `current_context_summary`.

Evidência: `bbrain-api/src/infrastructure/database/mongodb/schemas/reflective-profile.schema.ts:18-55`.

## Princípios

- Contexto é uma seleção por finalidade, não um dump de banco.
- Memória e conteúdo recuperado são dados; nunca ganham autoridade de instrução.
- Mensagem atual e correção explícita vencem resumo/memória anterior.
- Fato autorrelatado não vira fato confirmado; diagnóstico autorrelatado não explica comportamento automaticamente.
- Dado derivado deve permanecer distinguível de declaração do usuário.
- Dados sensíveis entram somente quando necessários, permitidos e proporcionais à tarefa.
- Cada item tem origem, validade, confiança, sensibilidade e expiração.
- Resumo não pode apagar negação, sujeito, temporalidade ou correção.

## Prompt registry versionado

O registry alvo registra metadados imutáveis:

```ts
interface PromptDefinition {
  key: string;
  version: string;          // semver ou hash publicado
  task: AiTask;
  supportedSchemaVersions: string[];
  locales: SupportedLocale[];
  contentHash: string;
  status: 'draft' | 'shadow' | 'active' | 'retired';
  reviewedAt: Date;
  safetyReviewId?: string;
}
```

O conteúdo concreto continua na infraestrutura. Regras invariantes também devem existir em policies/validators, não apenas no prompt. Uma execução armazena `promptKey`, `promptVersion` e `contentHash`, não necessariamente o prompt inteiro.

Mudança de prompt segue o mesmo gate de model change: dataset offline, shadow, canary e rollback. Hot edit sem versão é proibido.

### Context files e skills futuros

O backend auditado não possui runtime de skills. Se futuramente uma capability carregar arquivo procedural, rubric, taxonomia ou skill, esse artefato entra no registry como dado imutável com `artifactKey`, `artifactVersion`, `contentHash`, finalidade, permissões e status editorial. Uma publicação produz `contextBundleVersion`, que fixa exatamente o conjunto de artefatos permitido para uma tarefa. A execução registra essa versão e os hashes usados; apontar para “latest” em produção é proibido.

Arquivos editoriais derivados de livros seguem registry e revisão próprios, descritos em [Insights, Recursos e RAG](./insights-resources-rag.md). Eles não ganham autoridade de system prompt por serem context files.

## Hierarquia de instruções e dados

```text
1. políticas imutáveis de produto e segurança
2. contrato e objetivo da tarefa
3. schema de output e limites de ferramenta
4. mensagem atual do usuário (dado e intenção local)
5. estado canônico recente necessário
6. fatos/preferências consentidos e relevantes
7. conteúdo externo recuperado, sempre não confiável
8. resumos/derivados antigos, com validade explícita
```

Prioridade aqui não permite que a mensagem do usuário sobrescreva segurança; indica precedência factual entre dados. Exemplo: “na verdade foi anteontem” deve vencer um summary antigo que dizia “ontem”.

Cada bloco é serializado com envelope, sem interpolação como instrução:

```json
{
  "kind": "reported_fact",
  "source": "user_message",
  "sourceId": "...",
  "effectiveAt": "...",
  "validUntil": null,
  "sensitivity": "high",
  "content": "..."
}
```

## Context builder dinâmico

### Entrada

```text
task
userId
conversationId?
currentMessageId?
locale
tokenBudget
privacy policy snapshot
referenceTime + timezone
```

### Pipeline

```text
autorizar usuário/tarefa
 -> aplicar consentimento e entitlement
 -> carregar apenas fontes candidatas permitidas
 -> remover expirados/stale/superseded/deleted
 -> resolver relevância e recência
 -> aplicar precedência e deduplicação
 -> estimar tokens por bloco
 -> alocar orçamento por prioridade
 -> serializar dados delimitados
 -> produzir manifest de contexto sanitizado
```

### Alocação inicial de tokens para conversa

| Bloco | Cap inicial | Regra de corte |
|---|---:|---|
| policy + task + schema | 3.000 | nunca truncar no meio; escolher versão menor aprovada |
| mensagem atual | 4.000 | DTO já limita; rejeitar acima do plano em vez de truncar silenciosamente |
| estado conversacional | 500 | snapshot estruturado; omitir quando não consentido/expirado |
| preferências/objetivos | 700 | somente relevantes à resposta |
| fatos/memórias | 900 | top-k estruturado, consentido e não stale |
| summary recente | 400 | omitir se conflitar ou estiver vencido |

O total deve respeitar o budget de `CHAT_BALANCED`. Caps são ajustados por eval, não por intuição.

### Context manifest

Para debug e auditoria sem conteúdo:

```text
contextManifestId
task
item counts por kind/source
source ids ou hashes internos permitidos
token estimate por bloco
items omitted por privacy/stale/budget/conflict
builderVersion
contextBundleVersion
artifact keys/versions/hashes, sem conteúdo
```

Não registrar valores emocionais/textos no manifest operacional.

## Taxonomia de estado e memória

| Tipo | Exemplo | Autoridade | Persistência/expiração | Pode alimentar |
|---|---|---|---|---|
| preferência explícita | “prefiro respostas diretas” | declaração do usuário | até revisão/exclusão | estilo de conversa |
| objetivo | “quero organizar minha rotina” | declaração do usuário | revisável; expiração opcional | contexto de ajuda |
| fato autorrelatado | diagnóstico ou uso informado | autorrelato, não confirmação | alta sensibilidade; consentimento e retenção | somente tarefa estritamente relevante |
| evento temporal | ansiedade de manhã | evidência pontual | imutável + revisões/tombstone | resumos e Insights |
| registro parcial | dormiu aproximadamente 5 h | evidência pontual | imutável + revisões | histórico/Insights |
| resumo temporal | resumo diário/semanal | derivado ou manual | invalidável/versionado | UI/Insights, nunca como evidência primária |
| padrão derivado | tema recorrente | hipótese derivada | expira/requer evidência | contexto somente com rótulo derivado |
| Insight | associação longitudinal | derivado premium | referências + stale state | seção Insights, não identidade permanente |
| estado conversacional | tópico, necessidade, pergunta pendente | snapshot efêmero validado | TTL padrão de 24 h | continuidade local |
| conhecimento externo | documento oficial | fonte externa | ciclo de corpus | Recursos, nunca perfil |
| instrução procedural | policy/prompt | aplicação | versionada | comportamento do sistema |

`state` é o conjunto mínimo necessário para continuar um workflow. `conversation history` seriam mensagens, mas não é persistido pelo MVP. `memory` é informação selecionada para uso posterior e ainda não foi implementada. Esses conceitos não compartilham coleção ou interface genérica.

## Proveniência de memória

Uma memória futura precisa de:

```text
memoryId, userId, category, canonicalValue
sourceType, sourceId, sourceRevision
assertedBy = user | derived
confidence (somente se derived)
createdAt, effectiveAt, expiresAt?
retentionPolicy, sensitivity
status = active | stale | superseded | deleted
schemaVersion, derivationVersion?
```

O esqueleto atual já possui `category`, `confidence`, `source`, `relevance` e `retentionPolicy` (`memory.entity.ts:13-24`), e `MemoryReference` já aponta para `sourceId/sourceType` (`memory-reference.entity.ts:4-30`). Ele pode inspirar a linguagem, mas não deve ser ligado a produção antes de ownership, consentimento, revisão, repository e testes estarem definidos.

## Consolidação de memória

Workflow determinístico:

```text
evento/fato validado
 -> consent policy
 -> durabilidade e utilidade
 -> buscar memória semanticamente equivalente dentro da mesma categoria
 -> decidir create | reinforce evidence | supersede | ignore
 -> validar que conteúdo não extrapola evidência
 -> persistir por use case
```

O modelo pode sugerir a operação; policy decide. “Perguntou sobre medicamento” não é “usa medicamento”. Uma resposta informativa não atualiza o perfil. Mensagem do BBrain nunca é fonte de memória do usuário.

## Correções e conflitos

Ordem de precedência factual:

```text
edição manual mais recente
> correção explícita do usuário ligada à evidência
> declaração explícita mais recente
> evento extraído validado
> resumo derivado recente
> padrão/Insight derivado
> summary antigo
```

Conflito não resolvido permanece representado. O builder não escolhe silenciosamente a afirmação “mais conveniente”. Para dados sensíveis, conflito pode omitir ambos do contexto e solicitar esclarecimento naturalmente quando necessário.

## Compressão e sumarização

Compressão é `CONDITIONAL_ON_EVALS`. Antes de habilitar, a suite deve testar:

- preservação de negação: “não estou mais ansioso”;
- sujeito: “minha mãe não dormiu”;
- temporalidade: hoje/ontem/anteontem, intervalos e timezone;
- modalidade: desejo, hipótese, plano, ficção;
- correção: tristeza -> frustração;
- origem: fala do usuário versus fala do assistente;
- incerteza/aproximação;
- limites e preferências.

Um summary gerado inclui `sourceMessageIds`, `coversUntil`, `generatedAt`, `schemaVersion`, `promptVersion`, `status` e `expiresAt`. Ao editar/apagar uma mensagem fonte, ele fica stale.

## Proteção contra prompt injection

- Nunca concatenar conteúdo recuperado ao bloco de instrução.
- Serializar dados com tipo/origem e delimitadores estáveis.
- Não permitir que perfil, memória, documento ou mensagem altere tools, schema ou destino da tarefa.
- Tools com allowlist por capability e argumentos validados.
- Resources retrieval é read-only; nenhum documento pode acionar escrita.
- Detectar e registrar apenas código de tentativa, sem conteúdo integral.
- Incluir casos de injection direta/indireta nos evals.
- Verificar output por policy fora do modelo.

## Privacidade por finalidade

| Flag/policy | Efeito mínimo |
|---|---|
| `allowPersonalization=false` | não incluir preferências/fatos no chat; usar apenas identidade estritamente necessária |
| `allowMemory=false` | não criar/consolidar memória, não recuperar memória longa e não ler/escrever estado conversacional |
| `allowMoodInsights=false` | não gerar projeções/Insights de Humor; definir separadamente se histórico básico ainda pode ser criado conforme consentimento do produto |
| `allowSensitiveDataStorage=false` | não persistir fatos derivados nem estado; remover o estado ativo da conversa |

As semânticas exatas precisam de texto de consentimento e revisão jurídica/produto. Até lá, a implementação deve falhar para a opção mais restritiva e nunca inferir consentimento pelo uso do chat.

## Dados que não entram automaticamente

- perfil completo;
- todos os diagnósticos/medicações;
- todas as conversas;
- eventos antigos sem relevância;
- Insights stale;
- livros ou documentos integrais;
- conteúdo de outro usuário;
- dados de Recursos em memória pessoal;
- traces/eval samples como contexto de produto.

## Critérios de aceite

- Tests provam que privacy flags removem cada classe de contexto.
- Context manifest explica inclusão/omissão sem conteúdo sensível.
- Nenhuma mensagem de assistente origina fato/memória pessoal.
- Correção atual supera summary anterior.
- Outputs derivados são rotulados e rastreáveis.
- Token budget é respeitado deterministicamente.
- Prompt/schema/model/context-bundle e eventuais skill/artifact versions constam no execution record.
