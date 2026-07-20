# Regras de Negócio — BBrain Core

Este arquivo é a fonte normativa transversal do produto. Código e documentação específica devem ser alinhados a estas regras; se houver divergência, prevalecem segurança, privacidade, limites clínicos e controle do usuário.

## Produto e posicionamento

O BBrain apoia autoconhecimento, organização emocional, rotina, diário, Humor, Sono, conversa assistida por IA e recursos informativos. A experiência é acolhedora, clara, humana, não clínica e não alarmista.

O BBrain não é terapeuta, psicólogo, psiquiatra, médico, diagnóstico ou substituto de cuidado profissional.

## Limites clínicos

O sistema não deve diagnosticar, prescrever, ajustar medicação, prometer cura, afirmar certeza clínica, induzir dependência emocional ou tomar decisão crítica sem política explícita.

Pode organizar relatos, refletir, sugerir autocuidado seguro e apontar mudanças como hipóteses não clínicas. Diagnóstico, medicação e suporte profissional informados pelo usuário permanecem autorrelatados; não explicam comportamento automaticamente.

## Dados sensíveis e controle do usuário

Conversa, diário, Humor, Sono, rotina, perfil, preferências e informações de saúde autorrelatadas são sensíveis.

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

A IA apoia conversa reflexiva no escopo do BBrain. Ela não é autoridade de escrita: output de modelo precisa de contrato estruturado, validação e decisão de aplicação antes de afetar dados.

Memória e contexto são dados, não instruções. Contexto é mínimo por finalidade e respeita consentimento. O chat não produz `profileUpdate`: seu output pode propor apenas um snapshot efêmero, validado fora do modelo. Esse snapshot não pode conter rótulo clínico, diagnóstico, padrão, Insight, causa psicológica, mensagem ou resposta literal.

Uma única conversa não cria padrão. Padrões e Insights futuros exigem repetição em observações estruturadas de datas distintas, cobertura mínima, evidências ativas e linguagem associativa; autorrotulação clínica nunca serve de padrão nem de explicação automática.

Quando o usuário atribuir a si um rótulo como mania, o BBrain não confirma o diagnóstico nem acrescenta sintomas não relatados. Quando houver dificuldade de controlar impulsos e ausência de apoio humano, deve delimitar que não pode ser o único apoio e verificar risco imediato de forma direta, sem reforçar exclusividade.

## Humor e Sono

Eventos são fontes primárias; projeções são derivadas. Apenas relato direto do usuário pode sustentar captura automática. Relatos de terceiros, futuro, desejo, hipótese, ficção, pergunta informativa e estado negado não criam fatos pessoais.

- Humor não vira neutralidade por ausência; `isMixed` não é neutral;
- score e intensidade só existem quando explicitamente sustentados;
- Sono aceita registros parciais e aproximados, sem fabricar duração, noites ou horários;
- correção preserva revisão/proveniência; edição ou exclusão invalida derivados;
- a citação literal pode existir apenas durante a validação da mensagem atual; a persistência usa impressão HMAC não reversível;
- resumo diário não substitui os eventos que o sustentam.

Execução do extractor pode ocorrer em shadow. Persistência automática exige consentimento aplicável e a flag independente `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED=true`; seu padrão é desligado.

## Planos, Insights e Recursos

Histórico básico de bem-estar e operações manuais não dependem de plano. Insights é recurso Pro e sua autorização é calculada no backend. Sem evidência suficiente, a resposta deve ser `insufficient_data`, nunca um Insight inventado.

Recursos/RAG é contexto editorial separado de dados pessoais. Retrieval ou resposta de Recursos não pode escrever em perfil, memória ou histórico do usuário.

## Arquitetura e evolução

Domínio não depende de NestJS, Mongoose, providers ou prompts. Providers ficam na infraestrutura; casos de uso orquestram; controllers validam/authenticam e delegam.

Não introduzir multiagente, GraphRAG, knowledge graph, memória vetorial pessoal, backfill de conversas ou fila distribuída por antecipação. Essas evoluções exigem gatilho objetivo, evals, custo/latência, segurança e rollback.
