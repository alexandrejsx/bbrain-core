# Registros de decisão arquitetural

Os ADRs deste diretório registram decisões da arquitetura-alvo. `Proposta` significa que a direção foi documentada, mas ainda depende de implementação e dos gates descritos no próprio registro. A adoção de um ADR não autoriza tratar funcionalidades futuras como disponíveis.

## Índice

- [ADR-0001 — Workflow determinístico pós-resposta](./0001-deterministic-post-response-workflow.md)
- [ADR-0002 — Gateway de IA orientado a capacidades](./0002-capability-based-ai-gateway.md)
- [ADR-0003 — Proveniência, revisões e invalidação de derivados](./0003-provenance-revisions-and-derived-invalidation.md)
- [ADR-0004 — Trust domains separados para Recursos e RAG](./0004-trust-separated-resources-rag.md)
- [ADR-0005 — Contexto privacy-first e telemetria sanitizada](./0005-privacy-first-context-and-sanitized-telemetry.md)
- [ADR-0006 — Aggregate unificado e projeção diária estável](./0006-unified-wellbeing-observations-and-stable-projection.md)
- [ADR-0007 — Conversa sem transcrição e estado efêmero](./0007-transcript-free-conversation-state.md)

## Convenção

Cada ADR contém contexto, decisão, alternativas, consequências, gates e rollback. Uma mudança material em contratos, privacidade, persistência, política de modelos ou fronteiras de confiança exige novo ADR que substitua explicitamente o anterior; não se reescreve silenciosamente uma decisão já aplicada.
