# Arquitetura de IA do BBrain

Este é o único documento arquitetural normativo da implementação atual.

## Estrutura

```text
bbrain-api/src/
  ai/
    prompts/                 prompts centralizados
    providers/               OpenAI e Gemini
    safety/                  limites determinísticos da conversa
    conversation-agent.ts    único agente
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
    wellbeing/                API pública compatível de Humor/Sono
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
      Current Context / Memory e Pattern / Mood / Sleep
```

O `ConversationAgent` não conhece MongoDB. O Context Builder não persiste. O extractor não persiste. Os providers não conhecem consentimento nem regras de domínio. Controllers não montam prompts.

## Continuidade, Current Context e Memory

`conversation_sessions` guarda somente uma janela literal pequena para continuidade imediata: seis mensagens por padrão, máximo configurável de oito e TTL padrão de 24 horas. A collection nunca vira histórico completo.

`current_contexts` possui um documento por usuário e é substituído quando a situação atual muda. `memories` guarda Memory e Pattern consolidados. Seleção é lexical e temporal, sem vector database. Pattern só é persistido após pelo menos duas evidências independentes com dois tópicos normalizados em comum.

## Extração

Uma única chamada estruturada sugere cinco resultados independentes: Current Context, Memory, Pattern, Mood e Sleep. Cada parte pode ser `null`. O JSON passa por validação estrutural e depois pelas regras de confiança, consentimento e domínio. Nenhum resultado livre é persistido diretamente.

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
| billing/usage           | assinatura, cobrança e limites              | regras próprias existentes                                   |

Mood e Sleep expõem juntos o contrato existente de `/wellbeing-history/observations`, adequado a timeline, calendário, gráficos simples, edição e exclusão. Internamente permanecem em collections separadas e não dependem do chat depois de criados.

## Providers e observabilidade

`AI_PROVIDER` escolhe explicitamente OpenAI ou Gemini. `ModelRouter` resolve o modelo do provider ativo para `FAST`, `CONVERSATION` ou `REASONING`. Não existe fallback cross-provider.

Cada chamada registra metadados operacionais: operação, provider, modelo/papel, duração, tokens, sucesso/falha, tentativa e correlation id. O backend não registra mensagens, respostas, prompts completos ou dados emocionais. OpenAI recebe `store: false`; ambos os providers usam timeout e schema JSON.

## Segurança e consentimento

Dados do usuário são enviados como contexto delimitado, nunca concatenados ao system prompt. Diagnósticos são marcados como autorrelato formal. Consentimento é verificado antes do fluxo e novamente depois do provider. Extrações abaixo da confiança configurada, vazias ou inválidas são descartadas.

O BBrain mantém limites não clínicos e não confirma diagnósticos, prescreve, ajusta medicação, promete cura ou incentiva exclusividade emocional.
