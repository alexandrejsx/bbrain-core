# BBrain Core

Repositório central do ecossistema BBrain. Ele concentra scripts de desenvolvimento, diretrizes de produto e a documentação transversal; `bbrain` e `bbrain-api` permanecem repositórios independentes.

## Estrutura

```txt
bbrain-core/
├── bbrain/       # Frontend
├── bbrain-api/   # Backend/API
├── docs/          # Arquitetura e decisões transversais
└── ...
```

## Arquitetura e documentação

- [AGENTS.md](./AGENTS.md): diretrizes globais de engenharia, privacidade e fronteiras.
- [BUSINESS_RULES.md](./BUSINESS_RULES.md): regras normativas de produto.
- [Arquitetura de IA](./docs/ai-architecture/README.md): estado auditado, arquitetura-alvo, ADRs, migração e implementação incremental.
- [Relatório da implementação](./docs/ai-architecture/implementation-report.md): slice de conversa, Humor/Sono, Insights e seus limites.

O slice atual mantém o chat síncrono sem criar arquivo de transcrições. A API usa somente um estado conversacional estruturado, consentido e temporário (`conversation_states`) para continuidade; o estado expira por TTL e não aceita diagnóstico, padrão ou cópia literal. Idempotência usa um ledger técnico com HMAC (`conversation_exchange_ledgers`), sem pergunta ou resposta. Replays concluídos retornam conflito estável porque a resposta não é guardada.

A captura pós-resposta de Humor/Sono valida a citação apenas em memória e persiste uma impressão HMAC, nunca o trecho literal. Execução e persistência são separadas: `AI_OBSERVATION_EXTRACTION_ENABLED` habilita shadow; writes exigem também `AI_OBSERVATION_EXTRACTION_PERSIST_ENABLED=true`. Ambas começam em `false`.

Insights possui autorização Pro no backend e resposta honesta de dados insuficientes; Recursos/RAG e geração longitudinal de Insights permanecem etapas futuras.

## Clonando os projetos

Após clonar este repositório, clone também os projetos dependentes dentro da raiz:

```bash
git clone git@github.com:alexandrejsx/bbrain.git
git clone git@github.com:alexandrejsx/bbrain-api.git
```

Estrutura esperada:

```txt
bbrain-core/
├── bbrain/
└── bbrain-api/
```

## Importante

Os diretórios `bbrain/` e `bbrain-api/` são ignorados pelo Git do `bbrain-core`.

Isso significa que:

- Alterações realizadas no frontend devem ser commitadas no repositório `bbrain`.
- Alterações realizadas na API devem ser commitadas no repositório `bbrain-api`.
- O repositório `bbrain-core` não versiona o código desses projetos.
- O objetivo do `bbrain-core` é concentrar configurações compartilhadas, documentação e scripts de desenvolvimento.

## Instalação

Na raiz do projeto:

```bash
pnpm install
```

Instale também as dependências de cada projeto conforme suas documentações.

## Desenvolvimento

### Subir frontend e API

```bash
pnpm dev
```

Este comando executa:

* Frontend (`bbrain`)
* Docker da API
* Backend (`bbrain-api`)

### Executar apenas o frontend

```bash
pnpm dev:web
```

### Executar apenas a API

```bash
pnpm dev:api
```

### Subir containers da API

```bash
pnpm docker:up
```

### Derrubar containers da API

```bash
pnpm docker:down
```

## Commits

### Frontend

```bash
cd bbrain

git add .
git commit -m "sua mensagem"
git push
```

### Backend

```bash
cd bbrain-api

git add .
git commit -m "sua mensagem"
git push
```

### Core

```bash
git add .
git commit -m "sua mensagem"
git push
```

Utilize o repositório correto para cada alteração.

## Validação mínima

Execute as validações no repositório afetado. Para a API, a referência atual é:

```bash
cd bbrain-api
pnpm test --runInBand
pnpm run lint
pnpm build
```

Não habilite persistência automática de dados sensíveis sem evals de provider, integração Mongo, revisão de privacidade e plano de rollback.
