# Arquitetura de IA do BBrain

Este é o único documento arquitetural normativo da implementação atual.

## Estrutura

```text
bbrain-api/src/
  ai/
    prompts/                 prompts centralizados
    providers/               OpenAI e Gemini
    safety/                  limites determinísticos da conversa
    conversation-agent.ts    agente do chat comum
    daily-check-in-agent.ts  agente breve de Humor/Sono (FAST)
    model-router.ts           FAST / CONVERSATION / REASONING
    post-conversation.extractor.ts
    structured-output.ts
  modules/
    auth/                     autenticação e ciclo de conta
    billing/                  planos, uso, cobrança e webhooks
    chat/                     fluxo, contexto, janela recente e pós-processamento
    insights/                 contrato atual de elegibilidade/insuficiência
    memory/                   Current Context, Memory e Pattern
    mood/                     domínio e coleção de humor
    sleep/                    domínio e coleção de sono
    users/                    perfil, consentimento e persistência do usuário
    daily-check-in/           sessão, trial, orquestração e API próprias
    wellbeing/                API pública manual de Humor/Sono
  infrastructure/
    database/mongodb/         usuários, uso e billing existentes
    payments/                 Stripe e Asaas
```

Autenticação, perfil, planos e billing mantêm os contratos existentes. A parte conversacional e de bem-estar está organizada por feature; regras de billing/usage permanecem como domínio porque protegem invariantes reais.

## Percurso de uma mensagem

```text
POST /chat/message
  → autenticação e limite de uso
  → claim idempotente por user/session/sourceEvent
  → ContextBuilder
      perfil permitido + diagnóstico formal autorrelatado
      Current Context
      até 6 Memories e 3 Patterns relevantes
      janela recente de até 6 mensagens
  → ConversationAgent (papel CONVERSATION)
  → safety determinística
  → registro de uso + atualização da janela + resposta
  → agendamento local best-effort
      extractor estruturado (papel FAST)
      revalidação de consentimento
      Current Context / Memory / Pattern
```

O `ConversationAgent` não conhece MongoDB. O Context Builder não persiste. O extractor não persiste. Os providers não conhecem consentimento nem regras de domínio. Controllers não montam prompts.

```text
GET /daily-check-in/status
POST /daily-check-in/start | /daily-check-in/dismiss | /daily-check-in/answer
  → autenticação, consentimento e entitlement (trial de 7 dias ou plano pago)
  → adiamento diário persistido no mesmo draft por usuário/data
  → claim idempotente por usuário/data/request
  → DailyCheckInAgent (papel FAST)
  → structured output + confidence por campo
  → validação e limite determinístico de até 5 perguntas
  → Mood / Sleep nas coleções existentes
```

O check-in não usa `ConversationAgent`, endpoints de chat ou reserva de uso. Portanto não incrementa mensagens nem consome quota do chat. Seu draft persiste apenas o estado estruturado, a pergunta atual, o adiamento opcional do dia e identificadores técnicos; não existe transcript do check-in. A abertura automática pertence somente à Home; o Chat apenas pode consumir posteriormente o contexto estruturado concluído.

## Continuidade, Current Context e Memory

`conversation_sessions` guarda somente uma janela literal pequena para continuidade imediata: seis mensagens por padrão, máximo configurável de oito e TTL padrão de 24 horas. A collection nunca vira histórico completo.

`current_contexts` possui um documento por usuário e é substituído quando a situação atual muda. `memories` guarda Memory e Pattern consolidados. Seleção é lexical e temporal, sem vector database. Pattern só é persistido após pelo menos duas evidências independentes com dois tópicos normalizados em comum.

## Extração

Uma única chamada pós-conversa sugere três resultados independentes: Current Context, Memory e Pattern. Cada parte pode ser `null`. Mood e Sleep foram removidos integralmente desse contrato.

O `DailyCheckInAgent` possui schema próprio para Mood `0..10` e dimensões independentes de Sleep. Valores abaixo de `AI_EXTRACTION_MIN_CONFIDENCE` são descartados pela aplicação. O agente sugere a próxima pergunta, mas a aplicação impõe o limite e aceita dados parciais em vez de inventar campos.

O scheduler usa `setImmediate` e mantém tarefas ativas por usuário apenas para permitir drain seguro na exclusão. Não há fila, Redis ou job framework.

## MongoDB

| Collection              | Conteúdo                                    | Retenção/idempotência                                        |
| ----------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| `users`                 | autenticação, plano e perfil/consentimentos | vida da conta                                                |
| `conversation_sessions` | pequena janela recente                      | TTL; unique user/session                                     |
| `chat_requests`         | HMAC/status técnico da troca                | TTL; unique user/session/sourceEvent                         |
| `current_contexts`      | situação atual curta                        | um documento por usuário                                     |
| `memories`              | Memory e Pattern consolidados               | sourceEvent único para Memory; chave de tópicos para Pattern |
| `mood_records`          | eventos e resumos manuais de Humor          | request/sourceEvent idempotente; revisão                     |
| `sleep_records`         | observações parciais de Sono                | request/sourceEvent idempotente; revisão                     |
| `daily_check_ins`       | draft estruturado e coordenação diária      | unique user/data; request idempotente                        |
| billing/usage           | assinatura, cobrança e limites              | regras próprias existentes                                   |

Mood e Sleep expõem juntos o contrato existente de `/wellbeing-history/observations`, adequado a timeline, calendário, gráficos simples, edição e exclusão. Registros guiados usam proveniência `guided_checkin`; registro manual continua disponível em qualquer plano. O check-in concluído entra no `ContextBuilder` como contexto read-only do dia, carregado dos registros que permanecem como fonte de verdade.

## Providers e observabilidade

`AI_PROVIDER` escolhe explicitamente OpenAI ou Gemini. `ModelRouter` resolve o modelo do provider ativo para `FAST`, `CONVERSATION` ou `REASONING`. Não existe fallback cross-provider.

Cada chamada registra metadados operacionais: operação, provider, modelo/papel, duração, tokens, sucesso/falha, tentativa e correlation id. O backend não registra mensagens, respostas, prompts completos ou dados emocionais. OpenAI recebe `store: false`; ambos os providers usam timeout e schema JSON.

## Segurança e consentimento

Dados do usuário são enviados como contexto delimitado, nunca concatenados ao system prompt. Diagnósticos são marcados como autorrelato formal. Consentimento é verificado antes do fluxo e novamente depois do provider. Extrações abaixo da confiança configurada, vazias ou inválidas são descartadas.

O BBrain mantém limites não clínicos e não confirma diagnósticos, prescreve, ajusta medicação, promete cura ou incentiva exclusividade emocional.
