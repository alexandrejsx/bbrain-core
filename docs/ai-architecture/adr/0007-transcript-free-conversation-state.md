# ADR-0007 — Conversa sem transcrição e estado efêmero

- Status: Aceita e implementada
- Data: 2026-07-20
- Escopo: conversa, contexto, idempotência, segurança e proveniência

## Contexto

O chat persistia dois documentos literais por troca em `conversation_messages` e recarregava até vinte mensagens para cada chamada. Esse comportamento ignorava as flags de memória/storage sensível, aumentava exposição e criava crescimento de dados incompatível com o MVP. A mesma saída do modelo sugeria `profileUpdate`, misturando continuidade, memória durável e padrões.

Uma conversa em que o usuário relatou pouco sono, autorrotulação de mania, dificuldade com impulsividade e ausência de apoio mostrou também falhas de segurança: o modelo confirmou implicitamente sintomas não declarados, reforçou exclusividade e não verificou risco imediato.

## Decisão

O caminho ativo não persiste nem recarrega texto de usuário ou assistente.

```text
mensagem atual em memória
 -> privacy policy
 -> ConversationState ativo e estruturado, se permitido
 -> claim por HMAC sem conteúdo
 -> provider com store:false
 -> parser estrito
 -> ConversationSafetyReplyPolicy
 -> ConversationStateUpdatePolicy
 -> estado com TTL
 -> usage + conclusão do ledger
 -> captura opcional de Humor/Sono com a mensagem ainda em memória
```

`conversation_states` contém somente:

- tema atual curto;
- preocupações e necessidades para continuidade;
- disponibilidade de apoio;
- estado de segurança;
- código da pergunta pendente;
- intenção da última resposta;
- revisão e timestamps/TTL.

O estado não aceita rótulo clínico, diagnóstico, padrão, Insight ou trecho copiado. É lido/escrito apenas quando personalização, memória e storage sensível estão permitidos. Revogação apaga o estado da conversa. O padrão de TTL é 24 horas e é configurável.

`conversation_exchange_ledgers` usa chave única por usuário/conversa/mensagem fonte, HMAC SHA-256 com secret de finalidade, claim id, lease, status, risco/escopo, contagem de tokens e TTL. Não contém pergunta nem resposta. Reuso com outro HMAC é conflito; claim ativa sinaliza processamento; lease vencida pode ser retomada. Como a resposta não é armazenada, replay concluído retorna `MESSAGE_ALREADY_PROCESSED` em vez de reproduzir conteúdo.

O contrato de chat substitui `profileUpdate` por `conversationStateUpdate`. O modelo propõe um snapshot; parser, safety policy, state policy, entidade e repository decidem. Uma conversa isolada não cria padrão longitudinal.

A citação literal do extractor de Humor/Sono existe somente durante o grounding da mensagem atual. Persistência usa `evidenceFingerprint` HMAC; a API pública não expõe texto nem fingerprint.

`ConversationSafetyReplyPolicy` é defesa determinística além do prompt:

- autorrotulação de mania recebe limite explícito de não confirmação e nenhum sintoma é acrescentado;
- frases que reforçam exclusividade ou inventam energia/necessidade de sono são substituídas;
- “só você” somado a estado de impulsividade/pergunta de apoio produz limite de dependência, incentivo a apoio humano/profissional e pergunta direta sobre risco imediato.

### Fluxo do caso observado

```text
“tenho dormido pouco e trabalhado excessivamente”
  -> resposta acolhedora
  -> estado curto pode registrar tema sono/trabalho, sem copiar a frase

“creio/acho/estou/estar em mania”
  -> BBrain declara que não pode confirmar o rótulo
  -> não inventa energia elevada ou menor necessidade de sono
  -> o rótulo não entra no estado nem no perfil

“está difícil controlar impulsos” junto da autorrotulação
  -> policy abre checagem direta de risco imediato
  -> estado guarda apenas a categoria “controle de impulsos” e a pergunta pendente

resposta curta negativa à checagem
  -> policy avança para disponibilidade de apoio humano/profissional

“somente você”
  -> BBrain explicita que não deve ser o único apoio
  -> ausência de apoio + impulsividade abre nova checagem direta de segurança
```

Nenhuma dessas transições cria `emotional_pattern`, `recurring_theme`, diagnóstico ou Insight. Uma observação estruturada de Sono só pode ser proposta pelo workflow separado, sob consentimentos/flags e com proveniência HMAC, sem conservar a mensagem literal.

## Alternativas consideradas

- TTL em transcrições literais: reduz retenção, mas mantém exposição e crescimento durante a janela; rejeitada.
- criptografar transcrições: protege repouso, mas preserva coleta desnecessária, acesso aplicativo e custo; rejeitada para o MVP.
- resumo livre do histórico: ainda pode copiar texto, guardar diagnóstico e apagar negação; rejeitado como estado canônico.
- armazenar resposta para replay perfeito: melhora retry de rede, mas cria conteúdo durável; rejeitado pela prioridade de minimização.
- memória vetorial: amplia risco, custo e complexidade sem necessidade demonstrada; rejeitada.

## Consequências e trade-offs

Positivas:

- volume é limitado a um estado por conversa e ledgers temporários;
- privacy flags governam leitura e escrita;
- não existe transcript dump no prompt;
- padrões e diagnósticos não surgem de um turno;
- account deletion inclui estado, ledger e bem-estar.

Custos:

- o modelo perde nuance literal de turnos antigos;
- respostas curtas dependem de códigos/estado bem formados;
- retry após resposta perdida não pode recuperar o texto e retorna conflito;
- executor local de extração ainda perde trabalho em crash;
- estado concorrente usa optimistic revision e pode descartar update perdedor sem retry.

## Dados pré-MVP

Não há migração de transcrição. O ambiente será recriado antes do próximo uso, conforme decisão operacional do responsável pelo produto. Schema e repository legados foram removidos. A aplicação não apaga globalmente o banco e não executa cleanup destrutivo no startup.

## Gates e rollback

Gates: testes de ausência de append/read literal, TTL/ownership, HMAC conflict/lease, consentimento negado, autorrotulação, exclusividade, impulsividade, parser estrito e purge de conta.

Rollback de provider/prompt não pode reativar transcrições. Se o novo estado causar regressão, desabilitar sua leitura/escrita preservando chat stateless; qualquer retorno a conteúdo durável exige novo ADR e privacy review.
