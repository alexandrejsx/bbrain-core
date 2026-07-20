# BBrain Core

Repositório central do ecossistema BBrain.

Este documento define as diretrizes globais de todo o ecossistema BBrain.

---

# 1) Escopo global

O `bbrain-core` é o repositório central/orquestrador do ecossistema BBrain.

Pode conter scripts, automações, documentação e diretrizes globais.

Os projetos `bbrain` e `bbrain-api` continuam sendo projetos independentes, com histórico de commits, dependências e AGENTS específicos.

---

# 2) Visão de produto

O BBrain é uma plataforma de acompanhamento emocional, autoconhecimento, rotina e desenvolvimento pessoal assistida por IA.

A experiência deve ser:

- acolhedora
- clara
- segura
- privada
- moderna
- humana
- não clínica
- não alarmista
- não infantilizada

O BBrain não deve se posicionar como terapeuta, psicólogo, psiquiatra, médico, ferramenta de diagnóstico ou substituto de acompanhamento profissional.

---

# 3) Estrutura do ecossistema

O repositório central agrupa projetos com responsabilidades distintas.

## bbrain

Responsável por interface, experiência e fluxos de uso:

- UI e navegação
- formulários
- dashboard
- chat
- diário
- sono
- humor
- rotina
- insights
- recursos

Diretrizes específicas em `bbrain/AGENTS.md`.

## bbrain-api

Responsável por backend e regras de produto:

- autenticação
- regras de negócio
- persistência
- integrações externas
- IA
- processamento sensível

Diretrizes específicas em `bbrain-api/AGENTS.md`.

---

# 4) Princípios globais

Toda implementação deve priorizar:

1. Simplicidade
2. Legibilidade
3. Manutenibilidade
4. Segurança
5. Consistência

Evitar:

- complexidade desnecessária
- overengineering
- dependências sem justificativa

---

# 5) Arquitetura e fronteiras

Cada projeto mantém autonomia técnica e pode evoluir de forma independente.

- O frontend deve focar experiência do usuário, apresentação, formulários, navegação, acessibilidade e consumo de APIs.
- O backend deve ficar com autenticação, regras de negócio, persistência, integrações sensíveis, IA, pagamentos, segurança e controle de acesso.
- O frontend não deve acessar diretamente provedores de IA, gateways de pagamento, banco de dados ou outros serviços sensíveis.

No backend:

- domínio, entidades, value objects, DTOs e casos de uso em `camelCase`
- esquemas, documentos, filtros e objetos persistidos em `snake_case`
- conversões centralizadas em mappers da infraestrutura
- não espalhar conversão manualmente em controllers, serviços ou casos de uso

---

# 6) i18n global

O ecossistema deve suportar internacionalização desde o início.

Idiomas atuais:

- `pt-BR`
- `en-US`
- `es-ES`

Diretrizes:

- textos visíveis não devem ficar soltos em componentes sem controle;
- adicionar idioma novo deve exigir baixo esforço;
- idioma preferido deve ficar persistido no perfil;
- `pt-BR` é fallback padrão;
- preservar tom acolhedor/seguro/não clínico em todas as línguas;
- termos sensíveis de saúde mental requerem tradução e revisão cuidadosas;
- evitar tradução automática para mensagens críticas, de segurança, privacidade, crise ou pagamento.

---

# 7) Segurança e privacidade

Dados emocionais, diário, humor, sono, rotina, perfil, diagnósticos formais informados pelo usuário e preferências pessoais são sensíveis.

A implementação deve priorizar:

- privacidade por padrão
- menor privilégio
- não exposição de secrets
- não exposição de dados sensíveis em logs
- respostas de erro seguras
- controle do usuário sobre seus dados
- separação clara entre dados públicos, privados e sensíveis

---

# 8) IA e limites de escopo

Toda funcionalidade de IA deve respeitar os limites do produto:

- não diagnosticar
- não prescrever
- não ajustar medicação
- não prometer cura
- não substituir profissionais
- não fazer interpretações clínicas profundas
- não incentivar dependência emocional do agente
- responder apenas no escopo do BBrain

A IA atua como apoio reflexivo, organização emocional e autoconhecimento.

---

# 9) Planos, pagamentos e recursos premium

Podem existir recursos gratuitos e pagos.

Integrações e cobrança devem ficar no backend.

Recursos premium podem incluir, por exemplo:

- análises mais profundas
- ferramentas avançadas de diário
- plano de apoio
- relatórios exportáveis
- acompanhamento contínuo

Evitar amarrar regras conceituais a provedores específicos no AGENTS global.

---

# 10) Fontes de verdade e hierarquia

1. `bbrain-core/AGENTS.md` define regras globais.
2. `bbrain/AGENTS.md` define regras específicas de frontend.
3. `bbrain-api/AGENTS.md` define regras específicas de backend.
4. `DESIGN.md` define regras de design visual por projeto.

Em caso de conflito:

- preservar segurança, privacidade, clareza arquitetural e coerência de produto.

---

# 11) Privacidade e segurança por desenho

Todas as funcionalidades devem considerar privacidade desde o início do desenho, e qualquer implementação deve reforçar transparência, segurança e confiança do usuário.

---

# 12) IA, Humor, Sono e Insights

Para fluxos de IA e dados de bem-estar, consulte também `docs/ai-architecture/` e mantenha a documentação atualizada quando contratos, prompts, schemas, eventos, privacidade ou retenção mudarem.

- outputs de IA são sugestões não confiáveis até parser, policy e domínio aprovarem;
- somente backend pode chamar providers ou persistir derivados;
- fatos de Humor/Sono devem preservar fonte, precisão temporal, revisão e controle do usuário;
- ausência de dado não é neutralidade, score ou noite de sono;
- dados de terceiros, hipóteses, desejos, ficção e negação não devem criar fatos pessoais;
- resumo diário de Humor é derivado e não substitui evento primário;
- edição, correção e exclusão precisam invalidar derivados relevantes;
- histórico básico não é premium; Insights premium exige autorização no backend;
- Recursos/RAG nunca atualizam perfil ou memória pessoal por conta própria.

O fluxo atual possui modo shadow e persistência separados. Não habilite `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED` sem gates de eval, privacidade, operação e rollback documentados.

---

# 13) Fontes normativas e estado da implementação

`BUSINESS_RULES.md` é a fonte normativa de produto. O código mostra o estado implementado; `README.md` explica operação; documentos em `docs/ai-architecture/` registram decisões e limitações.

Quando houver conflito, preserve limites clínicos, segurança, privacidade e autorização. Não declare como implementado aquilo que esteja somente planejado ou documentado.

---

# 14) Estado de conversa e retenção

O fluxo ativo de conversa não deve ler nem escrever transcrições literais. Para continuidade, use somente `ConversationState` estruturado, minimizado, consentido e com TTL.

- não persistir mensagem do usuário nem resposta do BBrain;
- não usar `current_context_summary` como substituto de transcrição;
- não guardar rótulo clínico, diagnóstico, padrão ou Insight no estado efêmero;
- validar e rejeitar trechos copiados antes do repository;
- usar HMAC separado por finalidade para idempotência/evidência, com secret fora do código;
- revogação de memória ou storage sensível deve impedir leitura/escrita e apagar o estado ativo;
- schema/repository de `conversation_messages` não existem mais e não podem ser reintroduzidos no caminho de chat;
- uma única conversa nunca autoriza criar padrão longitudinal.

Autorrotulação clínica e exclusividade emocional exigem defesa fora do prompt: o modelo não pode confirmar o rótulo, inventar sintomas nem dizer que ele e o usuário estão “sozinhos nessa”. Impulsividade difícil de controlar somada à falta de apoio humano exige verificação direta de segurança imediata.
