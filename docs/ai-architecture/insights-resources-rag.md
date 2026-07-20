# Insights premium, Recursos e RAG

## Estado implementado do primeiro slice

Insights possui somente uma fronteira real de autorização/leitura: `GET /insights` exige plano Pro efetivo no backend e retorna `insufficient_data` com lista vazia. O frontend deixou de usar mock. Não há geração longitudinal, causalidade, recomendação, evidence bundle ou persistência de Insight; portanto nenhum conteúdo é fabricado para preencher a tela.

Recursos/RAG não foi implementado. O restante deste documento é estratégia e gate futuro: corpus oficial e editorial devem permanecer separados de dados pessoais, retrieval não pode escrever em perfil/memória, e qualquer síntese precisa de citações verificáveis. Não existe fallback autorizado para conhecimento não fundamentado.

## Separação de produto e confiança

Insights interpreta dados pessoais estruturados e consentidos. Recursos responde perguntas factuais usando fontes externas oficiais. Eles não compartilham corpus, memória, regras de persistência ou entitlement.

```text
INSIGHTS                               RESOURCES
dados do próprio usuário              documentos externos revisados
premium para interpretação            acesso definido pelo produto
evidências/eventos pessoais           claims e citações de fonte
invalida com edição/exclusão           atualiza com nova versão documental
nunca diagnóstico/causalidade          nunca recomendação personalizada de tratamento
```

Perguntar em Recursos sobre uma condição ou medicamento nunca cria diagnóstico, medicação, evento, memória ou preferência pessoal.

## Insights

### Estado atual aproveitável

O domínio possui esqueletos de `PatternInsight`, `BehavioralTrend` e `CorrelationAnalysis`, mas nenhum repository, use case, controller ou entitlement. `PatternInsight` já contém janela, confiança e status (`bbrain-api/src/domain/pattern-analysis/entities/pattern-insight.entity.ts:10-48`), porém ainda não referencia eventos concretos, coverage, limitation, revisão ou versão de derivação.

Esse esqueleto pode evoluir, sem presumir que esteja pronto para produção.

### Contrato conceitual

```ts
interface Insight {
  id: string;
  userId: string;
  kind: string;
  period: { startsAt: string; endsAt: string; timezone: string };
  title: string;
  narrative: string;
  claims: Array<{
    id: string;
    text: string;
    nature: 'observation' | 'association' | 'limitation';
    evidenceIds: string[];
    aggregationIds?: string[];
  }>;
  coverage: {
    observedDays: number;
    eligibleDays: number;
    missingPeriods: Array<{ startsAt: string; endsAt: string }>;
    description: string;
  };
  limitations: string[];
  confidenceBand: 'low' | 'moderate' | 'high';
  status: 'candidate' | 'validated' | 'stale' | 'rejected' | 'deleted';
  derivationVersion: string;
  validationVersion: string;
  createdAt: string;
  invalidatedAt?: string;
}
```

Confiança não é probabilidade clínica. É uma faixa sobre suporte observacional e coverage, definida por policy/testes.

### Gate de acesso

```text
JWT/ownership
 -> effective plan/entitlement no backend
 -> feature flag
 -> privacy settings/consentimento
 -> evidence validity
 -> retornar Insight validado ou estado seguro
```

Esconder uma seção no frontend não protege o recurso. `User.getEffectivePlan` e a infraestrutura de plans/usage existentes podem fornecer a base, mas Insights deve depender de um port de entitlement com capability (`insights.read`, `insights.generate`), não de comparação espalhada com enum de plano.

Histórico, provenance, CRUD manual e correções não são premium. O gate protege geração/visualização da interpretação longitudinal.

### Evidência mínima

Não fixar no domínio um número arbitrário de dias/eventos antes dos evals. A policy versionada recebe:

- mínimo de eventos distintos;
- mínimo de dias observados;
- coverage mínima da janela;
- diversidade de fontes/horários quando relevante;
- máximo de dados stale/deleted;
- tipos elegíveis por Insight.

Valores iniciais são calibrados em dataset e revisão humana. Se o gate falhar, retorna `insufficient_evidence` com explicação simples, não gera texto especulativo.

### Pipeline

```text
selecionar janela e eventos ativos
 -> construir agregações determinísticas
 -> calcular coverage/missingness
 -> evidence gate
 -> gerar candidate estruturado
 -> checar cada claim contra evidence ids
 -> validar linguagem clínica/causalidade/ancoragem
 -> persistir como validated
 -> servir com period, coverage, limitations e evidências
```

O modelo nunca recebe acesso geral ao banco. Query ports retornam DTOs minimizados. Diagnóstico autorrelatado pode ser omitido por padrão; se necessário a uma funcionalidade futura expressamente aprovada, entra como autorrelato e nunca como explicação automática.

### Invalidação

Ao editar/excluir evento ou summary:

1. localizar Insights por `evidenceId + revision`;
2. marcar `stale` dentro do mesmo commit/outbox;
3. impedir read de stale;
4. recalcular somente com entitlement/consentimento ainda válidos;
5. preservar audit metadata da versão anterior conforme retenção.

O read path verifica revisions para não depender exclusivamente de evento assíncrono.

### Síncrono ou assíncrono

Inicialmente sob demanda com cache e job pós-request. Geração não bloqueia conversa. Um job periódico só é introduzido se houver valor de produto claro. Fila durável é ativada pelos gatilhos de [target-architecture.md](./target-architecture.md), não apenas porque Insights pode ser caro.

## Recursos

### Trust domains

| Corpus | Autoridade | Uso permitido | Uso proibido | Storage/index |
|---|---|---|---|---|
| fontes médicas oficiais | máxima dentro de jurisdição/versão | fatos de condição, medicamento, indicação, advertência e interação sustentados | aconselhamento individual, extrapolação, instruções do documento | `official_resources` dedicado |
| material editorial derivado de livros | intermediária e revisada | estilo, escuta reflexiva, taxonomias, princípios e rubricas autorizadas | substituir bula/diretriz, diagnosticar, citação médica oficial | `editorial_context` dedicado |
| conteúdo interno BBrain | produto | explicar funcionalidades/políticas do produto | fatos clínicos sem fonte oficial | `product_content` dedicado |
| dados pessoais | autoridade do usuário sobre seu relato | conversa/Insights consentidos | retrieval de Recursos | fora de todo índice de Recursos |

Nunca usar um único índice com filtro opcional para separar pessoal/oficial/editorial. O isolamento físico reduz falha de filtro e data leakage.

### Documento oficial

```ts
interface KnowledgeDocument {
  documentId: string;
  corpus: 'official_resources';
  authority: string;
  jurisdiction: string;
  documentType: string;
  audience: 'patient' | 'professional' | 'mixed';
  title: string;
  canonicalUrl: string;
  sourcePublishedAt?: string;
  sourceUpdatedAt?: string;
  retrievedAt: string;
  versionLabel?: string;
  language: string;
  reviewStatus: 'pending' | 'approved' | 'retired';
  checksum: string;
  parserVersion: string;
  supersedesDocumentId?: string;
  permission: string;
}
```

### Chunk

```ts
interface KnowledgeChunk {
  chunkId: string;
  documentId: string;
  sectionPath: string[];
  pageOrAnchor?: string;
  text: string;
  checksum: string;
  ordinal: number;
  medication?: string[];
  activeIngredient?: string[];
  condition?: string[];
  warningClass?: string[];
  validFrom?: string;
  validUntil?: string;
}
```

Chunking preserva headings, tabelas, advertências e escopo da seção. Não cortar contraindicação ou qualifier de população de modo que mude o significado. Parsing e chunking são versionados e testados em documentos reais.

### Arquivos derivados de livros/NotebookLM

```ts
interface EditorialArtifact {
  artifactId: string;
  sourceTitle: string;
  authors: string[];
  edition: string;
  pageReferences: string[];
  purpose: string;
  allowedUses: string[];
  prohibitedUses: string[];
  createdWith?: 'NotebookLM' | 'manual' | 'other';
  version: string;
  reviewer?: string;
  reviewStatus: 'draft' | 'approved' | 'expired' | 'rejected';
  reviewedAt?: string;
  expiresAt?: string;
  rightsBasis: string;
  checksum: string;
}
```

- Preferir sínteses próprias e princípios a trechos extensos.
- Verificar direitos/licença e finalidade; NotebookLM não concede direito autoral nem autoridade clínica.
- Não ingerir automaticamente output editorial em `official_resources`.
- Recuperar pequenos artifacts por finalidade; nunca anexar livros/arquivos integrais em toda chamada.
- Expirar artifact sem revisão dentro da policy definida.

## Pipeline de ingestion

```text
allowlist de fonte
 -> fetch/import administrado
 -> checksum e malware/type validation
 -> extração preservando estrutura
 -> normalização sem remover qualifiers
 -> chunking semântico
 -> metadata validation
 -> revisão e aprovação
 -> índice staging
 -> retrieval regression suite
 -> publicação atômica de corpus version
 -> manter versão anterior para rollback
```

Documentos recuperados da web não são automaticamente publicados. Update detecta checksum/versão, marca versão anterior como superseded e impede citações novas nela após a data efetiva, sem quebrar auditoria de respostas antigas.

## Retrieval

Baseline obrigatório:

1. filtros de corpus/autoridade/jurisdição/idioma/status/versão;
2. busca lexical com headings e aliases controlados;
3. top-k pequeno e deduplicação por seção;
4. evidence pack com metadata completa.

Somente adicionar:

- semântica se melhorar Recall@k em paráfrases sem aumentar cross-topic retrieval;
- híbrida se vencer lexical e semantic isoladas;
- query rewriting se perguntas coloquiais ficarem abaixo do gate;
- reranking se ganho de section accuracy justificar custo/latência.

GraphRAG, knowledge graph e agentic RAG permanecem `AVOID_FOR_NOW` até consultas multi-hop comprovadas falharem em um corpus grande e o ganho aparecer em eval cego.

## Defesa contra prompt injection indireta

- Ingestion trata HTML/PDF/texto como dados não confiáveis.
- Remover scripts/metadados ativos; preservar texto para auditoria, mas nunca executá-lo.
- Evidence pack usa envelopes e ids; nenhuma frase recuperada altera tools/system policy.
- Synthesizer não possui write tools nem navegação livre.
- Instructions encontradas dentro do chunk são ignoradas e sinalizadas.
- Corpus e filtros são determinados pela aplicação, nunca pelo documento/modelo.
- Output passa por citation verifier e policy clínica.

## Síntese e citações

Cada bloco factual ou claim recebe ids de chunks:

```json
{
  "claimId": "c1",
  "text": "...",
  "citations": [
    {
      "chunkId": "...",
      "documentId": "...",
      "sectionPath": ["..."],
      "sourceUrl": "...",
      "versionLabel": "...",
      "sourceUpdatedAt": "..."
    }
  ]
}
```

O verifier confirma que o trecho sustenta o claim, que autoridade/jurisdição/audiência são adequadas e que a versão está ativa. Claim sem suporte é removido ou substituído por comunicação de insuficiência. Lista genérica de links ao final não satisfaz cobertura de citação.

## Limites clínicos

Recursos pode explicar o que uma fonte oficial declara. Não pode:

- dizer que o usuário possui uma condição;
- recomendar iniciar, parar ou ajustar tratamento/medicação;
- personalizar contraindicação como decisão médica;
- transformar busca em memória pessoal;
- omitir conflito relevante entre fontes;
- preencher lacuna documental com conhecimento do modelo.

Perguntas de risco imediato seguem a policy de segurança e orientação a apoio local; retrieval não atrasa uma resposta urgente.

## Evals e gates

### Insights

- 100% dos claims têm evidence ids válidos.
- 0 diagnóstico/causalidade indevida no critical set.
- 0 uso de evento deleted/stale.
- coverage e missingness corretos em 100% dos fixtures determinísticos.
- reação a edição/exclusão validada end-to-end.
- utilidade avaliada por humanos, separada de segurança.

### Retrieval/Recursos

- Recall@5 e section accuracy acima do baseline acordado por corpus.
- 0 cross-corpus/cross-jurisdiction leak no critical set.
- 100% dos documentos retornados estão `approved` e vigentes.
- citation entailment e coverage acima dos gates de [eval-strategy.md](./eval-strategy.md).
- 0 profile/memory write em toda rota Recursos.
- injection suite não altera tools, corpus ou policy.

## Rollback

- Corpus é publicado por alias/version; rollback troca alias para versão anterior.
- Embeddings e chunks mantêm parser/chunker/embedding version.
- Insight generation é desligável por flag sem apagar histórico básico.
- Insight stale nunca é servido durante rollback.
- Nenhum rollout remove campos/coleções antigas antes de leitura de compatibilidade e reconciliação.
