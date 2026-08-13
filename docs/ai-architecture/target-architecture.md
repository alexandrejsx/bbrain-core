# Arquitetura atual

```text
Frontend → Controller → Use case → Provider → Parser → Policies de domínio → MongoDB
                              ↓
                    captura pós-resposta de Humor e Sono
```

A conversa produz a resposta normalmente. A captura posterior é best-effort: falhas não alteram a
resposta do chat e não criam registros parciais. Quando há consentimento válido, uma observação
estruturada e validada é persistida com proveniência, deduplicação e revisão otimista; projeções de
Humor são atualizadas em seguida.

O backend é a fonte de verdade para autenticação, ownership, consentimento e persistência. Providers
e adapters permanecem isolados da camada de domínio. Logs usam somente metadados sanitizados.
