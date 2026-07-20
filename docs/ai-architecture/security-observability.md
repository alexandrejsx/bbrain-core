# Segurança, privacidade e observabilidade

## Estado implementado do primeiro slice

Ownership vem do JWT em todos os endpoints novos; respostas não expõem `userId`, idempotency key, fingerprint, modelo, prompt ou schema internos. Proveniência visível ao dono preserva fonte, ids de conversa/mensagem e confiança, sem citação literal. O context builder e a captura aplicam privacy flags; a captura revalida conta/consentimento depois da chamada externa. OpenAI e Gemini usam `store: false`, o chat usa estado/ledger com TTL sem transcrição, e account deletion inclui estado, ledger e bem-estar após bloquear/drenar tarefas locais.

Permanecem lacunas de produção: configuração global ainda não é fail-closed, Node 20 está EOL, não há rate limiting/tracing/métricas estruturadas, auditoria de dependências não foi executada, e o coordenador de captura não protege múltiplas réplicas. Persistência, eventos e projeções não usam transação/outbox; a leitura faz checagens defensivas, mas é necessária reconciliação durável. Não houve inspeção de tráfego real de provider nem teste de eliminação distribuída.

## Postura

Dados emocionais, sono, diário, conversas, perfil, diagnósticos/medicação autorrelatados e Insights são altamente sensíveis. O sistema deve operar com privacidade por padrão, menor privilégio, minimização e controle do usuário. Observabilidade mede comportamento técnico sem transformar conteúdo pessoal em telemetria.

Este documento separa fotografia auditada, slice implementado e alvo. Na fotografia inicial não havia tracing, métricas, rate limiting, audit trail ou validação forte de env. Os únicos logs específicos de IA ficavam nos adapters (`openai-chat-agent.ts:86-165`, `gemini-chat-agent.ts:76-171`), e usage agregava tokens (`user-daily-usage.schema.ts:31-44`). O slice não pretende encerrar essas lacunas transversais.

## Classificação de dados

| Classe | Exemplos | Logs operacionais | Tracing | Produto | Dataset de eval |
|---|---|---|---|---|---|
| segredo | JWT secret, API keys, webhook secrets, reset code | nunca | nunca | secret manager/hash quando aplicável | nunca |
| credencial/token | senha/hash, bearer token, reset hash | nunca | nunca | storage mínimo protegido | nunca |
| sensível pessoal | conversa, Humor, Sono, diário, diagnóstico/medicação | nunca por padrão | apenas ids/contagens; conteúdo off | storage de produto com ownership/retention | somente processo explícito, minimizado e consentido |
| identificador | userId, conversationId, eventId | pseudonimizado/permitido conforme ambiente | permitido com controle de acesso | sim | substituir por ids sintéticos |
| billing | customer/subscription/order/payment metadata | ids mínimos; sem payload | ids/status | conforme finalidade e retenção legal | sintético |
| conhecimento oficial | documentos publicados | ids/version/status | ids | corpus isolado | permitido conforme licença |
| editorial | sínteses de livros | ids/version/review status | ids | corpus isolado | conforme direitos e revisão |
| sintético | fixtures sem pessoa real | permitido | permitido | fora de produção | padrão preferido |

Logs, traces, datasets e dados do produto usam storages, IAM e retenções diferentes.

## Threat model resumido

### Ativos

- identidade e credenciais;
- estado conversacional efêmero e perfil reflexivo;
- eventos de Humor/Sono e revisões;
- Insights e evidências;
- documentos/corpus e direitos editoriais;
- secrets de providers/pagamento;
- entitlement e billing;
- prompts/policies/schemas;
- telemetry e datasets de eval.

### Ameaças principais

- IDOR/cross-user query;
- bypass de consentimento/entitlement;
- prompt injection direta ou indireta;
- modelo criar fato falso ou agir como write authority;
- vazamento em log/trace/provider;
- brute force de auth/reset e abuso de custo;
- webhook duplicado/fraudulento;
- replay/duplicação de jobs;
- stale Insight após correção/exclusão;
- corpus errado/obsoleto ou mistura de trust domains;
- supply-chain/runtime EOL;
- configuração insegura silenciosa.

## Controles obrigatórios por boundary

| Boundary | Controles |
|---|---|
| HTTP | JWT, DTO whitelist, ownership derivado do token, rate limit, body limit, security headers, correlation id, error mapping sem stack |
| Auth/reset | secret forte obrigatório, código nunca logado, hash, TTL, tentativas/cooldown, resposta anti-enumeração, auditoria minimizada, invalidação de sessão conforme policy |
| Application | autorização/entitlement/privacy antes de carregar dados, idempotency, expected revision, policy explícita, transação quando necessária |
| AI gateway | allowlist de task/model/provider, budgets, timeout, retry limitado, `store:false` quando suportado, region/data policy, schema validation, no arbitrary tools |
| Modelo -> domínio | output não confiável; schema + domain validation; modelo não fornece `userId` e não escreve repository |
| MongoDB | queries sempre scoped, least-privilege credentials, TLS/auth em produção, encryption at rest, backups testados, índices via migração |
| Jobs/eventos | envelope validado, idempotency/inbox, least privilege, retry/backoff, DLQ quando necessário, conteúdo por referência |
| RAG | índices fisicamente isolados, allowlist de fonte, review status, metadata filters, injection defense, claim citations |
| Observabilidade | redaction allowlist, acesso restrito, retenção curta, nenhuma mensagem/resposta inteira por padrão |

## Correções imediatas do estado atual

### Configuração fail-closed

`ConfigModule` deve validar env no startup por ambiente. Em produção:

- proibir `JWT_SECRET=local-secret` (`config.ts:141`, `auth.module.ts:41`);
- exigir provider e sua API key/modelo;
- exigir `EMAIL_PROVIDER=resend`, remetente e chave, ou desabilitar reset por código de forma segura;
- exigir webhook secrets para providers habilitados;
- validar URL schemes/hosts, timezone e números positivos;
- rejeitar CORS wildcard/empty inesperado;
- registrar apenas nomes de campos inválidos, nunca valores.

### Email/reset

`EmailService.logEmail` registra destinatário e texto completo (`email.service.ts:57-69`), incluindo reset code produzido em `request-password-reset.use-case.ts:38-58`. Em produção, fallback para log deve ser impossível. Em local/test, o preview deve ser opt-in e claramente isolado; nunca enviado a collector central.

Adicionar tentativas máximas por code/account/IP, cooldown de request, single active code, comparação constante via hash, invalidation após sucesso e eventos de auditoria sem email/código em claro.

### Privacy enforcement

O context builder atual usa perfil sensível e histórico sem consultar `privacy_settings` (`conversation-agent-context-builder.service.ts:47-92`; flags em `user.schema.ts:43-48`). Introduzir `ConversationPrivacyPolicy`/port equivalente antes de qualquer load/serialize. Tests devem provar cada flag.

### Account deletion

`AccountLifecycleService` apaga apenas perfil, mensagens e user (`account-lifecycle.service.ts:50-57`). Criar orchestration idempotente por classe:

```text
produto sensível -> purge
usage técnico ligado ao usuário -> purge/anonymize conforme finalidade
billing -> retenção legal mínima + pseudonimização quando aplicável
telemetria -> remover vínculo/expirar
eval datasets -> processo separado de retirada
provider state -> delete API quando aplicável e contratado
```

Uma falha parcial fica reconciliável; usuário não é apagado definitivamente antes de registrar status de cada etapa, salvo desenho transacional comprovado.

### Runtime e dependências

`Dockerfile:1` usa Node 20, EOL na data da auditoria. Migrar para Node 22 ou 24 LTS, imagem slim/distroless compatível, digest pinning, usuário não-root, `WORKDIR`, `CMD`, healthcheck e SBOM/scanning. Mongo local deve ter versão pinada; produção não pode replicar o compose sem auth/TLS.

## Autorização, entitlement e IDOR

- `userId` sempre vem do JWT e é aplicado em todo repository/query.
- IDs de record nunca são suficientes: carregar por `(id,userId)` ou validar aggregate ownership antes de retornar/mutar.
- Insights requer capability backend `insights.read/generate`, não apenas plano no frontend.
- CRUD manual de Humor/Sono não é premium.
- Resources não usa user profile para retrieval; locale/jurisdição explícitos não equivalem a condição pessoal.
- Admin/ingestion usa role e credencial separadas, sem acesso automático a conversas.

O padrão positivo atual de payment ownership (`billing.service.ts:112-128`) deve ser reproduzido nos novos records.

## Guardrails

### Entrada

- DTO length/type/enum;
- detecção determinística de payload/arquivo proibido;
- rate limit e abuse scoring;
- safety classifier com hard timeout;
- delimitação de conteúdo como dado.

### Saída

- schema;
- enum/date/size;
- domain policy;
- clínica/causalidade;
- provenance/citation;
- PII leakage e instrução interna;
- localized safe fallback.

Guardrail não deve alterar silenciosamente fato extraído para “parecer válido”. Rejeita candidate e registra reason code.

## Provider data controls

O adapter OpenAI já define `store:false` (`openai-chat-agent.ts:109-133`), coerente com a documentação de data controls. Isso não substitui verificação contratual de abuse monitoring, region, ZDR, subprocessors e retenção. Cada provider policy registra:

```text
allowedDataClasses
region/residency
retentionMode
store flag
training/usage terms reviewedAt
contractOwner
```

Gemini e qualquer provider futuro exigem revisão equivalente antes de receber dados sensíveis. O fallback nunca pode enviar para provider sem a mesma autorização de classe/região.

## Observabilidade alvo

### Request correlation

Gerar/aceitar um `correlationId` validado na borda; propagar por controller, use case, repositories, AI execution e job. Não usar email, user text ou token como correlation id.

### Log estruturado allowlist

```json
{
  "timestamp": "...",
  "level": "info",
  "service": "bbrain-api",
  "environment": "...",
  "event": "ai.execution.completed",
  "correlationId": "...",
  "executionId": "...",
  "task": "mood_extraction",
  "policyVersion": "...",
  "provider": "...",
  "modelSnapshot": "...",
  "status": "succeeded",
  "latencyMs": 1234,
  "inputTokens": 100,
  "outputTokens": 40,
  "retryCount": 0,
  "schemaVersion": "...",
  "promptVersion": "..."
}
```

Não incluir `message`, `reply`, prompt renderizado, profile, document chunk, provider response body, email, telefone, diagnóstico, medicamento, token ou secret.

### Métricas

| Família | Métricas |
|---|---|
| HTTP | requests, latency p50/p95/p99, status, route template, rate-limit blocks |
| AI | calls por task/provider/model/policy, latency, tokens, estimated cost, timeout, retry, fallback, refusal, invalid schema/output |
| Extraction | candidates, accepted/rejected por reason, dedupe, corrections, manual edits/deletes, stale projection delay |
| Safety | categories/action codes, false-positive/negative audit aggregate, over-escalation |
| Jobs | enqueued/started/succeeded/failed, attempts, age, backlog, DLQ, idempotent duplicate |
| RAG | retrieval latency, zero-result, corpus/version, Recall@k offline, unsupported claims, citation coverage |
| Insights | generated/rejected/stale, evidence gate, invalidation latency, cost |
| Privacy | items omitted por policy, blocked unauthorized access, deletion completion; nunca valores sensíveis |

Cardinalidade é controlada. Não usar `userId`, `conversationId`, document URL ou error message como metric label.

### Tracing

Spans sugeridos:

```text
http.request
  auth.verify
  context.build
  ai.execute
    provider.http
    output.parse
    domain.validate
  persistence.commit
  event.publish

job.execute
  source.load
  ai.extract
  domain.validate
  persistence.commit
  derivation.invalidate
```

Attributes seguem allowlist de metadata. Content capture fica `false` por padrão. Qualquer sampling de conteúdo para incident/eval requer approval, justificativa, minimização, encryption, access log, TTL e vínculo de retirada.

### Audit trail

Audit é separado de log operacional e registra ações de controle:

- consent/privacy change;
- visualização/edição/exclusão de record sensível quando exigido;
- entitlement decision;
- geração/invalidação de Insight;
- corpus publish/retire;
- prompt/model/policy activation;
- export/deletion request;
- acesso administrativo.

Audit não guarda conteúdo anterior/novo por padrão; usa record/revision/action/result.

## SLOs iniciais e alertas

São objetivos de engenharia a validar, não SLA comercial.

| Fluxo | SLI/SLO inicial | Alerta |
|---|---|---|
| conversa | disponibilidade 99,5%; p95 <= 8 s excluindo indisponibilidade upstream definida | erro > 2%/5 min, p95 > 10 s/15 min, fallback > 10% |
| safety | 99,9% execução/fallback seguro; p95 <= 1 s | qualquer bypass de critical policy |
| extração assíncrona | 99% concluída em 60 s; 99,9% sem duplicação | oldest job > 2 min, failure > 1%, duplicate write > 0 |
| invalidação | stale não servido; 99% marcado em 10 s | stale read > 0 |
| Recursos | p95 retrieval <= 1,5 s; 100% docs approved | cross-corpus leak ou unapproved doc > 0 |
| exclusão de conta | todas as etapas dentro da janela declarada | qualquer etapa failed além do retry budget |

## Retry, circuit breaker e fallback

- Retry apenas em timeout, 429 e 5xx reconhecidos; exponential backoff com jitter.
- Respeitar deadline/cost budget total; não reiniciar budget a cada tentativa.
- Sem retry automático de violação de domínio.
- Circuit breaker por provider/capability, não global.
- Fallback somente aprovado pela mesma policy/data class.
- Extração falha sem write; conversa pode usar resposta segura; Resources comunica insuficiência.
- Registrar reason code, não response body.

## Jobs

O timer atual de exclusão roda em todas as réplicas e descarta promise (`account-lifecycle.service.ts:18-27`). Curto prazo: ownership único configurado, lock e log de resultado. Médio prazo, se gatilhos ocorrerem: job runner/fila com envelope:

```text
jobId, type, version, idempotencyKey
subjectId/referenceId (mínimo)
createdAt, notBefore, attempt, correlationId
```

Payload não carrega transcript; worker recarrega por id com autorização técnica mínima. DLQ armazena metadata e reason code. Reconciliação compara fonte com derived state, não apenas estado da fila.

## Feature flags e rollback

- Flags no backend e avaliadas após auth/privacy.
- Rollout por cohort aleatório não baseado em atributo sensível.
- Kill switch de auto-persist separada de chat.
- Desligar tracing de conteúdo é configuração imutável em produção.
- Policy/model/prompt rollback preserva execution records.

## Testes de segurança mínimos

- cross-user IDs em todos os CRUDs;
- privacy flags em todas as combinações relevantes;
- JWT secret/config startup;
- reset request/confirm rate/attempts e ausência de código em logs;
- provider error com PII/secrets não vaza;
- prompt injection direta/indireta;
- webhook replay/concurrency/crash window;
- duplicate job e optimistic concurrency;
- delete cascade/anonymization/reconciliation;
- stale Insight após edição/exclusão;
- cross-corpus retrieval;
- SSRF/file-type/malware protections no ingestion;
- dependency/container scan no CI.

## Critérios de aceite

- Nenhum conteúdo sensível aparece em logs/traces padrão.
- Config insegura impede startup de produção.
- Privacy policy é testada antes de contexto e escrita.
- Todo record é scoped por usuário e revision.
- Todo call de IA tem budgets, versões e status observáveis.
- Webhooks/jobs são idempotentes sob concorrência e crash simulado.
- Deletion report cobre todas as classes de dados.
- Runtime usa linha Node suportada e imagens pinadas.
