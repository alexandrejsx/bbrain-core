# BBrain Core

Diretrizes globais do ecossistema BBrain. O frontend (`bbrain`) e o backend (`bbrain-api`) são projetos independentes e possuem instruções específicas próprias.

## Produto

O BBrain apoia acompanhamento emocional, autoconhecimento, rotina e desenvolvimento pessoal assistidos por IA. A experiência deve ser acolhedora, clara, segura, privada, moderna, humana, não clínica, não alarmista e não infantilizada.

O produto não é terapeuta, psicólogo, psiquiatra, médico, ferramenta de diagnóstico ou substituto de acompanhamento profissional. A IA não diagnostica, prescreve, ajusta medicação, promete cura, confirma autorrotulação clínica nem incentiva dependência emocional.

## Princípios

- simplicidade, legibilidade, manutenção, segurança e consistência;
- privacidade por padrão, menor privilégio e minimização de dados;
- frontend como referência do contrato e da experiência atual;
- backend como autoridade de autenticação, autorização, regras, persistência, IA e integrações sensíveis;
- `camelCase` na aplicação e `snake_case` em documentos MongoDB, com conversão localizada nos repositories/mappers;
- `pt-BR`, `en-US` e `es-ES`, com `pt-BR` como fallback;
- código morto e abstrações sem uso são removidos, não preservados por precaução.

## Arquitetura atual do backend

O backend usa Clean Architecture pragmática, organizada principalmente por feature. Conceitos de domínio só existem quando protegem uma regra real; não há DDD cerimonial, interfaces para tudo ou camadas antecipadas.

Existe um único agente: `ConversationAgent`. Memory, Current Context, Pattern, Mood e Sleep são dados ou operações especializadas, não agentes. O `ContextBuilder` é o único lugar que monta o contexto conversacional. Extrações de IA usam structured output, validação de schema, política de consentimento e persistência separada.

OpenAI e Gemini ficam atrás de um único provider configurável. A seleção de modelos usa somente os papéis `FAST`, `CONVERSATION` e `REASONING`. MongoDB é a persistência.

A conversa completa não é memória permanente. Quando há consentimento, apenas uma janela recente pequena e temporária é mantida para continuidade imediata; informação antiga útil é consolidada em Current Context, Memory, Pattern, Mood ou Sleep. Prompts, mensagens completas e dados emocionais não entram em logs de produção.

Antes de adicionar um novo agente, camada arquitetural, framework, banco, pipeline ou abstração, deve existir uma necessidade funcional concreta no produto atual que justifique sua inclusão.

## Dados e IA

- diagnósticos entram no contexto apenas quando informados formalmente pelo próprio usuário;
- ausência de dado não vira score, humor neutro ou noite de sono;
- uma ocorrência não cria Pattern; recorrência exige múltiplas evidências independentes;
- Mood e Sleep aceitam informação parcial e aproximada sem fabricar precisão;
- extração automática requer consentimento válido, ownership, proveniência e revalidação antes do write;
- `sourceEventId` suporta idempotência sem armazenar a frase original;
- revogação e exclusão impedem novas escritas e removem dados aplicáveis;

## Fontes de verdade

1. `BUSINESS_RULES.md` para regras normativas do produto;
2. código para o estado realmente implementado;
3. `docs/ai-architecture/README.md` para o fluxo técnico atual;
4. `bbrain/AGENTS.md` e `bbrain-api/AGENTS.md` para regras específicas.

Não declarar como implementado algo apenas planejado. Em conflitos, preservar limites clínicos, segurança, privacidade, autorização e coerência do produto.
