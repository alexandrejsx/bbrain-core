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

Depois da resposta, processamento local assíncrono pode extrair Current Context, Memory e propostas de Pattern. Mood e Sleep nunca são criados ou atualizados a partir da conversa comum. Cada candidato passa por structured output, validação de schema, regras de negócio, consentimento, ownership e persistência idempotente. Falha nessa etapa não altera a resposta do chat.

Sem informação suficiente, o resultado é vazio e nenhum registro é criado. Não persistir texto bruto da conversa. Proveniência usa origem, `capturedAt`, `sessionId`, `sourceEventId` e versões do extractor/prompt quando úteis.

## Humor e Sono

`mood_event`, `mood_daily_summary` manual e `sleep_record` são registros independentes e editáveis, com revisão otimista e proveniência. Suas únicas fontes de criação são registro manual ou Daily Check-in guiado.

- o Daily Check-in normaliza Mood em inteiro `0..10`, sem interpretação clínica e somente quando há evidência explícita na resposta à pergunta de humor;
- Sono preserva duração, aproximação, qualidade subjetiva, despertares, tempo acordado durante a noite e sensação de descanso como dimensões independentes;
- campos subjetivos e temporais permanecem ausentes quando não informados;
- uma observação semanal é um período, não vários dias fictícios;
- ausência de dado não é neutralidade, score ou noite de sono;
- criação manual e histórico básico independem de plano;
- correção e exclusão pertencem somente ao usuário autenticado.

## Planos e Insights

Autorização de plano é calculada no backend. O endpoint atual de Insights apenas informa elegibilidade ou dados insuficientes; não existe Insight Agent nem geração de Insight neste estágio.

O Daily Check-in guiado por IA possui trial de sete dias desde a criação da conta e, depois, requer plano pago efetivo. Esse acesso não altera a disponibilidade do registro manual nem do histórico básico. Respostas e chamadas de IA do check-in não passam pela reserva nem pela contagem de mensagens do chat.

## Evolução

Simplicidade precede abstração. Código, configuração, teste e documentação sem utilidade atual devem ser removidos. Infraestrutura futura só entra quando uma necessidade funcional concreta do produto atual justificar seu custo e risco.
