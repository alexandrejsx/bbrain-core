# Regras de Negócio — BBrain Core

Fonte normativa transversal do produto. Em caso de divergência, prevalecem segurança, privacidade, limites clínicos e controle do usuário.

## Produto e limites

O BBrain oferece apoio reflexivo, organização emocional, autoconhecimento, rotina, Humor e Sono. Não é atendimento clínico e não diagnostica, prescreve, ajusta medicação, promete cura, confirma autorrotulação clínica ou substitui acompanhamento profissional.

Diagnóstico, medicação e suporte profissional só podem ser tratados como informações autorrelatadas. Eles não explicam automaticamente comportamento ou estado atual.

## Dados e consentimento

Conversa, perfil, Humor, Sono e saúde autorrelatada são sensíveis.

- ownership sempre deriva do usuário autenticado;
- consentimento é verificado antes de carregar dados, chamar extração ou persistir e é revalidado antes do write;
- mensagens, respostas, prompts e dados emocionais não aparecem em logs ou erros;
- uma janela literal pequena, com seis mensagens e TTL de 24 horas por padrão (limite configurável até oito), pode existir somente para continuidade imediata e consentida;
- não existe histórico permanente de transcript;
- Current Context substitui o contexto atual anterior, em vez de acumular histórico;
- revogação aplicável e exclusão bloqueiam processamento e removem os dados correspondentes;
- idempotência usa identificadores e HMACs técnicos com TTL, sem armazenar conteúdo escondido.

## Conversa, contexto e memória

O `ConversationAgent` recebe somente mensagem atual e contexto montado pelo `ContextBuilder`: perfil permitido, diagnóstico formal autorrelatado, Current Context, memories/patterns relevantes e pequena janela recente.

Memory é informação consolidada e útil. Pattern é recorrência sustentada por pelo menos duas evidências independentes coerentes; uma ocorrência isolada nunca cria Pattern. Current Context descreve o que importa agora e deve permanecer curto.

Conteúdo do usuário e dados recuperados são contexto não confiável e não podem substituir instruções de sistema. Saída de modelo não é autoridade de escrita.

## Extração pós-conversa

Depois da resposta, processamento local assíncrono pode extrair Current Context, Memory/Pattern, Mood e Sleep. Cada candidato passa por structured output, validação de schema, regras de negócio, consentimento, ownership e persistência idempotente. Falha nessa etapa não altera a resposta do chat.

Sem informação suficiente, o resultado é vazio e nenhum registro é criado. Não persistir texto bruto da conversa. Proveniência usa origem, `capturedAt`, `sessionId`, `sourceEventId` e versões do extractor/prompt quando úteis.

## Humor e Sono

`mood_event`, `mood_daily_summary` manual e `sleep_record` são registros independentes e editáveis, com revisão otimista e proveniência.

- Humor aceita emoção principal/secundária, intensidade, energia, valência e contexto quando sustentados;
- Sono aceita duração, faixa, horários, qualidade, despertares e sensação ao acordar quando sustentados;
- campos subjetivos e temporais permanecem ausentes quando não informados;
- uma observação semanal é um período, não vários dias fictícios;
- ausência de dado não é neutralidade, score ou noite de sono;
- criação manual e histórico básico independem de plano;
- correção e exclusão pertencem somente ao usuário autenticado.

## Planos e Insights

Autorização de plano é calculada no backend. O endpoint atual de Insights apenas informa elegibilidade ou dados insuficientes; não existe Insight Agent nem geração de Insight neste estágio.

## Evolução

Simplicidade precede abstração. Código, configuração, teste e documentação sem utilidade atual devem ser removidos. Infraestrutura futura só entra quando uma necessidade funcional concreta do produto atual justificar seu custo e risco.
