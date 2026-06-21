# BBrain Core

Repositório central do ecossistema BBrain.

Este documento define as diretrizes globais válidas para todos os projetos do ecossistema.

---

# 1. Visão do produto

O BBrain é uma plataforma de acompanhamento emocional, autoconhecimento e desenvolvimento pessoal assistida por inteligência artificial.

O objetivo do produto é ajudar pessoas a:

- Entender padrões emocionais.
- Desenvolver autoconsciência.
- Criar hábitos saudáveis.
- Refletir sobre comportamentos e rotinas.
- Receber insights personalizados.
- Manter acompanhamento contínuo da própria evolução.

A experiência deve transmitir:

- Clareza.
- Simplicidade.
- Acolhimento.
- Segurança.
- Confiança.
- Privacidade.

---

# 2. Estrutura do ecossistema

Este repositório contém múltiplos projetos.

## bbrain

Aplicação web principal do produto.

Responsável por:

- Interface do usuário.
- Experiência de uso.
- Fluxos de autenticação.
- Dashboards.
- Chat.
- Diário.
- Sono.
- Humor.
- Rotina.
- Insights.
- Recursos.

Diretrizes específicas devem ser definidas em:

```txt
bbrain/AGENTS.md
````

---

## bbrain-api

Backend principal do produto.

Responsável por:

* APIs.
* Regras de negócio.
* Persistência de dados.
* Integrações externas.
* Inteligência de aplicação.
* Processamento de IA.

Diretrizes específicas devem ser definidas em:

```txt
bbrain-api/AGENTS.md
```

# 3. Princípios globais

Toda implementação deve priorizar:

1. Simplicidade.
2. Legibilidade.
3. Manutenibilidade.
4. Segurança.
5. Consistência.

Evitar:

* Complexidade desnecessária.
* Overengineering.
* Dependências sem justificativa.

As diretrizes de design visual e produto detalhadas estão no `DESIGN.md` de cada projeto.

---

# 4. Arquitetura

Cada projeto deve possuir autonomia técnica.

Frontend e backend podem evoluir independentemente.

No backend, domínio, classes, DTOs e casos de uso devem usar `camelCase`. Schemas, documentos, filtros e objetos persistidos no banco devem usar `snake_case`. Toda conversão entre esses padrões deve ficar centralizada em mappers da camada de infraestrutura.

As regras específicas de cada projeto devem permanecer em seus respectivos AGENTS.md.

Este documento deve conter apenas diretrizes globais do ecossistema BBrain.

---

# 5. Privacidade e segurança

Dados do usuário devem ser tratados como informações sensíveis.

Prioridades:

* Transparência.
* Segurança.
* Controle pelo usuário.

Toda funcionalidade deve considerar privacidade desde a concepção.

---

# 6. Fonte de verdade

Em caso de conflito:

1. Este AGENTS.md define as diretrizes globais.
2. O AGENTS.md do projeto define as regras específicas.
3. Implementações devem respeitar ambos.

Hierarquia:

bbrain-core (global)
    - bbrain
        - AGENTS.md
    - bbrain-api
        - AGENTS.md
