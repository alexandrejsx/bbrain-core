# BBrain Core

Repositório raiz responsável por centralizar e orquestrar os projetos do ecossistema BBrain.

## Estrutura

```txt
bbrain-core/
├── bbrain/       # Frontend
├── bbrain-api/   # Backend/API
└── ...
```

## Clonando os projetos

Após clonar este repositório, clone também os projetos dependentes dentro da raiz:

```bash
git clone <url-do-bbrain> bbrain
git clone <url-do-bbrain-api> bbrain-api
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

* Alterações realizadas no frontend devem ser commitadas no repositório `bbrain`.
* Alterações realizadas na API devem ser commitadas no repositório `bbrain-api`.
* O repositório `bbrain-core` não versiona o código desses projetos.
* O objetivo do `bbrain-core` é apenas concentrar configurações compartilhadas, documentação e scripts de desenvolvimento.

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
