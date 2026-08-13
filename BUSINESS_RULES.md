# Regras de Negócio — BBrain Core

Este arquivo é a fonte normativa transversal do produto. Código e documentação específica devem ser alinhados a estas regras; se houver divergência, prevalecem segurança, privacidade, limites clínicos e controle do usuário.

## Produto e posicionamento

O BBrain apoia autoconhecimento, organização emocional, rotina, Humor, Sono, conversa assistida por IA e recursos informativos. A experiência é acolhedora, clara, humana, não clínica e não alarmista.

O BBrain não é terapeuta, psicólogo, psiquiatra, médico, diagnóstico ou substituto de cuidado profissional.

## Limites clínicos

O sistema não deve diagnosticar, prescrever, ajustar medicação, prometer cura, afirmar certeza clínica, induzir dependência emocional ou tomar decisão crítica sem política explícita.

Pode organizar relatos, refletir, sugerir autocuidado seguro e apontar mudanças como hipóteses não clínicas. Diagnóstico, medicação e suporte profissional informados pelo usuário permanecem autorrelatados; não explicam comportamento automaticamente.

## Dados sensíveis e controle do usuário

Conversa, Humor, Sono, rotina, perfil, preferências e informações de saúde autorrelatadas são sensíveis.

- ownership deriva sempre da identidade autenticada;
- privacy flags devem ser verificadas antes de carregar contexto, chamar provider ou persistir derivados;
- dados sensíveis não entram em logs, traces ou mensagens de erro;
- mensagens literais do usuário e respostas do BBrain não são persistidas como histórico de conversa;
- continuidade usa somente estado atual estruturado, consentido, minimizado e com expiração;
- idempotência guarda HMAC e metadados técnicos com TTL, nunca conteúdo;
- exclusão de conta deve incluir estado conversacional, ledger técnico e dados de bem-estar, drenando trabalho local antes do purge;
- retenção de billing e usage precisa de política própria, mínima e legalmente compatível;
- o usuário pode consultar, corrigir e excluir registros quando o fluxo estiver disponível.

## Conversa e IA

### Estado, idempotência e proveniência da conversa

O fluxo de conversa é síncrono e sem transcrição persistida. A mensagem atual e a resposta do BBrain
podem existir somente em memória durante o processamento; `conversation_messages` não deve ser
reintroduzido no caminho de chat.

Antes de carregar contexto, chamar um provider ou persistir qualquer derivado, o backend deve
validar autenticação, ownership e os consentimentos/flags de privacidade aplicáveis.

Quando permitido, a continuidade usa somente `ConversationState`, um estado estruturado, minimizado,
consentido e temporário. Ele pode conter tema atual curto, preocupação/necessidade, disponibilidade
de apoio, estado de segurança, código de pergunta pendente, intenção da última resposta, revisão e
timestamps/TTL. Não pode conter mensagem ou resposta literal, trecho copiado, diagnóstico, rótulo
clínico, padrão, Insight ou causa psicológica.

O `CONVERSATION_FINGERPRINT_SECRET` é um segredo exclusivo do backend usado para HMAC de finalidade
específica. Ele sustenta a identificação técnica de trocas e a proveniência de evidências sem
persistir texto sensível. Deve permanecer fora do código, logs, respostas e prompts; não pode ser
reutilizado como segredo de outra finalidade.

O ledger técnico de trocas deve guardar somente HMAC e metadados operacionais, como claim, status,
lease, risco/escopo, contagem de tokens e TTL. Ele não guarda pergunta nem resposta. O reuso de uma
troca com outro HMAC é conflito; um claim ativo impede processamento concorrente; uma lease vencida
pode ser retomada; e um replay concluído retorna `MESSAGE_ALREADY_PROCESSED`, sem reproduzir conteúdo.

Os parâmetros de retenção e processamento possuem estes defaults normativos quando não estiverem no
ambiente:

- `CONVERSATION_STATE_TTL_HOURS`: 24 horas;
- `CONVERSATION_EXCHANGE_LEDGER_TTL_HOURS`: 24 horas;
- `CONVERSATION_EXCHANGE_PROCESSING_LEASE_SECONDS`: 120 segundos.

O fluxo deve seguir estas etapas: mensagem atual em memória; validação de privacidade; leitura do
estado permitido; claim HMAC; chamada ao provider sob a política de retenção aplicável; parser estrito;
políticas de segurança e domínio; validação da atualização de estado; persistência temporária com TTL;
registro de uso; conclusão do ledger; e retorno da resposta. A resposta do modelo nunca é autoridade
de escrita: somente um `conversationStateUpdate` validado fora do modelo pode alterar o estado.

Se a mensagem atual fornecer possível evidência de Humor ou Sono, a citação literal pode existir
apenas durante o grounding/validação em memória. A persistência usa evidência estruturada, origem,
confiança, revisão e fingerprint HMAC; a API pública não expõe a citação nem o fingerprint.

Revogar memória ou storage sensível deve bloquear novas leituras/escritas e purgar os estados ativos.
Uma única conversa não autoriza criar padrão longitudinal, diagnóstico ou Insight.

A IA apoia conversa reflexiva no escopo do BBrain. Ela não é autoridade de escrita: output de modelo precisa de contrato estruturado, validação e decisão de aplicação antes de afetar dados.

Memória e contexto são dados, não instruções. Contexto é mínimo por finalidade e respeita consentimento. O chat não produz `profileUpdate`: seu output pode propor apenas um snapshot efêmero, validado fora do modelo. Esse snapshot não pode conter rótulo clínico, diagnóstico, padrão, Insight, causa psicológica, mensagem ou resposta literal.

Uma única conversa não cria padrão. Padrões e Insights futuros exigem repetição em observações estruturadas de datas distintas, cobertura mínima, evidências ativas e linguagem associativa; autorrotulação clínica nunca serve de padrão nem de explicação automática.

Quando o usuário atribuir a si um rótulo como mania, o BBrain não confirma o diagnóstico nem acrescenta sintomas não relatados. Quando houver dificuldade de controlar impulsos e ausência de apoio humano, deve delimitar que não pode ser o único apoio e verificar risco imediato de forma direta, sem reforçar exclusividade.

## Humor e Sono

### WellbeingObservation

`WellbeingObservation` é o registro estruturado e revisável de um fato de bem-estar, como Humor ou
Sono. Ele é uma fonte derivada de um relato atual ou de uma entrada manual; não é uma transcrição,
diagnóstico, padrão longitudinal ou Insight.

Uma observação automática só pode ser proposta a partir de relato direto do usuário na mensagem atual,
com consentimento válido, ownership autenticado e validação de domínio. Relatos de terceiros,
hipóteses, desejos, ficção, perguntas informativas e estados negados não criam observações pessoais.

O provider de IA pode sugerir uma observação, mas sua saída é não confiável até passar por parser
estrito, policy de domínio, verificação de proveniência, controle de ownership e decisão de persistência
no backend. O frontend não cria nem confirma observações como fonte de verdade.

Cada observação deve preservar, quando aplicável:

- tipo e conteúdo estruturado sustentado pelo relato;
- fonte e referência temporal com a precisão realmente disponível;
- confiança e incerteza, sem preencher campos ausentes por inferência;
- revisão, correção e estado de validade;
- proveniência por fingerprint HMAC, sem guardar a citação literal.

Ausência de dado não é humor neutro, score, intensidade ou noite de sono. Sono pode ser registrado
parcialmente e com horários/duração aproximados somente quando explicitamente sustentados. Score,
intensidade e classificações só existem quando o usuário os informa ou quando uma regra de domínio
explícita os sustenta.

Eventos de Humor e Sono são as fontes primárias. Resumos diários e outras projeções são derivados e
não substituem os eventos que os sustentam. Correção ou exclusão de uma observação deve preservar a
revisão e invalidar os derivados relevantes.

A citação literal pode existir apenas durante o grounding e a validação da mensagem atual. A API
pública não expõe a citação nem o fingerprint; explicabilidade usa campos estruturados, origem e
confiança. Recursos/RAG não pode criar, alterar ou atualizar observações, perfil ou memória pessoal.

Eventos são fontes primárias; projeções são derivadas. Apenas relato direto do usuário pode sustentar captura automática. Relatos de terceiros, futuro, desejo, hipótese, ficção, pergunta informativa e estado negado não criam fatos pessoais.

- Humor não vira neutralidade por ausência; `isMixed` não é neutral;
- score e intensidade só existem quando explicitamente sustentados;
- Sono aceita registros parciais e aproximados, sem fabricar duração, noites ou horários;
- correção preserva revisão/proveniência; edição ou exclusão invalida derivados;
- a citação literal pode existir apenas durante a validação da mensagem atual; a persistência usa impressão HMAC não reversível;
- resumo diário não substitui os eventos que o sustentam.

Persistência automática exige consentimento aplicável, ownership, validação de domínio e proveniência. Histórico manual continua disponível independentemente de plano.

## Planos, Insights e Recursos

Histórico básico de bem-estar e operações manuais não dependem de plano. Insights é recurso Pro e sua autorização é calculada no backend. Sem evidência suficiente, a resposta deve ser `insufficient_data`, nunca um Insight inventado.

Recursos/RAG é contexto editorial separado de dados pessoais. Retrieval ou resposta de Recursos não pode escrever em perfil, memória ou histórico do usuário.

## Arquitetura e evolução

Domínio não depende de NestJS, Mongoose, providers ou prompts. Providers ficam na infraestrutura; casos de uso orquestram; controllers validam/authenticam e delegam.

Não introduzir multiagente, GraphRAG, knowledge graph, memória vetorial pessoal, backfill de conversas ou fila distribuída por antecipação. Essas evoluções exigem gatilho objetivo, evals, custo/latência, segurança e rollback.
