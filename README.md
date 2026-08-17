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
- [Relatório operacional de 22/07/2026](./docs/ai-architecture/operational-validation-report.md): baseline, rastreabilidade, Mongo real, providers, evals, validações e bloqueadores atuais.
- [Sistema de IA explicado](./docs/ai-architecture/project-ai-system-explained.md): visão técnica e de produto do fluxo completo, harness, memória, segurança e limites.
- [Relatório da implementação de 20/07/2026](./docs/ai-architecture/implementation-report.md): snapshot histórico do slice de conversa, Humor/Sono e Insights.

O slice atual mantém o chat síncrono sem criar arquivo permanente de transcrições. A API usa uma janela recente pequena e temporária para continuidade e um ledger técnico com HMAC para idempotência, sem transformar conversa em memória permanente.

Humor e Sono estruturados são criados somente por registro manual ou pelo Daily Check-in guiado. O check-in usa um `DailyCheckInAgent` independente no papel `FAST`, possui draft estruturado por usuário/data, trial de sete dias e não consome mensagens do chat. O pós-processamento da conversa comum continua limitado a Current Context, Memory e Pattern.

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

- Frontend (`bbrain`)
- Docker da API
- Backend (`bbrain-api`)

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

Não habilite persistência automática de dados sensíveis enquanto os bloqueadores do relatório operacional estiverem abertos. Nenhuma flag de produção ou deploy foi alterado nesta validação.
