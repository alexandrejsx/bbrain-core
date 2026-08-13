# Arquitetura de IA

O BBrain usa IA como apoio reflexivo, sem diagnóstico, prescrição ou substituição de profissionais.

O fluxo de conversa mantém apenas `ConversationState` estruturado, consentido e temporário. Não há
transcrições literais no caminho ativo. Depois da resposta, a mensagem atual pode gerar observações
de Humor ou Sono: o provider propõe candidatos estruturados, parser e policies de domínio validam
o resultado e somente observações válidas, autorizadas e consentidas são persistidas.

Documentos principais:

- [Dados de Humor e Sono](./mood-sleep-data.md)
- [Estratégia de evals](./eval-strategy.md)
- [Segurança e observabilidade](./security-observability.md)
- [Arquitetura alvo](./target-architecture.md)
- [ADRs](./adr/README.md)
