# ADR-0005 — Contexto privacy-first e telemetria sanitizada

- Status: Proposta
- Data: 2026-07-20
- Escopo: contexto, memória, providers, logs, métricas e auditoria

## Contexto

O perfil já contém preferências de privacidade, mas o builder de contexto e a persistência reflexiva atuais não as aplicam como gates. Logs podem incluir erro controlado por provider, e o fluxo de recuperação de senha permite registrar código quando email está desabilitado. Não há correlação, tracing, métricas por tarefa/versão nem audit trail próprio para leitura e mutação de dados sensíveis.

## Decisão

Aplicar `PrivacyPolicy` antes da seleção de contexto, chamada externa e persistência. O builder recebe uma autorização tipada e monta o menor contexto necessário em camadas: regras do produto, estado da conversa, fatos consentidos pertinentes e evidence pack específico da tarefa. `false` ou preferência ausente falha fechada para memória/derivação nova. Revogação impede novas leituras/escritas e aciona o processo de retenção/deleção definido.

Conteúdo de usuário, diário, emoção, sono, prompts montados, outputs brutos, tokens de autenticação, códigos de reset e trechos recuperados não entram em logs por padrão. Telemetria usa allowlist e identificadores opacos: tarefa, versões, provider/modelo, duração, contagem de tokens, estado do schema, classe de erro, retry, cache, decisão de policy e flags. Hashes/fingerprints devem ser não reversíveis e não reutilizados entre finalidades sem necessidade.

Um audit trail separado registra quem/qual serviço leu, alterou, exportou ou excluiu dado sensível, com finalidade e resultado, sem copiar o conteúdo. Métricas e traces têm correlação por `requestId`/`executionId` e retenção menor que dados de produto. Dados enviados a provider respeitam minimização, região/controles contratuais disponíveis e configuração sem treino/retenção quando aplicável.

## Alternativas consideradas

- Filtrar depois de montar o prompt: dados já podem ter sido acessados ou copiados; rejeitado.
- Logar prompts/outputs para depuração: útil no curto prazo e incompatível com privacidade por padrão.
- Um opt-in global indiferenciado: não representa finalidades distintas como contexto, memória, Insights e melhoria.
- Confiar somente nos controles do provider: eles não substituem autorização e minimização locais.

## Consequências

- Preferência do usuário vira regra executável, não metadado decorativo.
- Diagnóstico operacional passa a depender de métricas, códigos e replay controlado em dataset sintético, não de texto de produção.
- Builders, repositórios, providers e jobs precisam aceitar um contexto de autorização consistente.
- Investigações excepcionais de conteúdo exigem processo explícito, acesso mínimo, justificativa e auditoria.

## Gates e rollback

Nenhuma feature que use perfil/memória é habilitada sem testes de consentimento negado, revogado e cross-user. Scanner de logs deve provar ausência de secrets e conteúdo dos casos sintéticos. Se uma versão de contexto ou telemetria vazar campo proibido, desligar a flag, revogar/rotacionar credenciais quando necessário, interromper exportação e executar resposta a incidente; rollback não reativa a coleta anterior.

## Evidência e fontes

Ver [prompts, contexto e memória](../prompt-context-memory.md) e [segurança e observabilidade](../security-observability.md). A política de retenção por endpoint deve acompanhar a documentação oficial do provider, incluindo [Data controls in the OpenAI platform](https://developers.openai.com/api/docs/guides/your-data).
