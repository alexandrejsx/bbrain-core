# ADR-0004 — Trust domains separados para Recursos e RAG

- Status: Proposta
- Data: 2026-07-20
- Escopo: Recursos, documentos, material editorial e síntese fundamentada

## Contexto

Recursos responde perguntas factuais e educacionais com corpus curado; memória e histórico pessoal descrevem o usuário. Misturá-los no mesmo índice amplia vazamento entre usuários, contamina autorização e permite que conteúdo recuperado seja confundido com instrução. Documentos oficiais e material editorial derivado de livros também têm autoridade, atualização e direitos diferentes.

## Decisão

Manter trust domains e índices fisicamente distintos:

- `official_resources_index`: fontes oficiais, com jurisdição, versão, vigência e seção;
- `editorial_book_derivatives_index`: notas/resumos autorizados, com direitos e revisão editorial; não contém reprodução extensa;
- `internal_product_content_index`: conteúdo próprio do BBrain;
- dados pessoais: fora do corpus de Recursos e sempre sujeitos a autorização por usuário.

Ingestão valida origem, licença/permissão, hash, versão, idioma, data e estado editorial antes de publicar chunks. Material recuperado entra como dados delimitados e não executáveis. Retrieval usa filtros de trust domain e metadados; busca semântica, híbrida, query rewriting ou rerank só são adotados se superarem baseline lexical no conjunto de avaliação.

Síntese fundamentada recebe evidence pack mínimo e deve ligar cada afirmação factual verificável a uma citação de trecho. Uma etapa determinística, complementada por verificador avaliado quando necessário, bloqueia citação inexistente, fonte vencida ou claim sem suporte. Pergunta de Recursos não cria memória pessoal nem atualiza perfil.

## Alternativas consideradas

- Um vector store único: simplifica ingestão, mas viola isolamento, retenção e autoridade.
- Enviar documentos completos ao modelo: reduz implementação e aumenta custo, exposição, prompt injection e ruído.
- GraphRAG/knowledge graph desde o início: sem evidência de benefício proporcional à complexidade.
- Permitir resposta sem fonte quando retrieval falha: gera confiança indevida; o sistema deve declarar limitação.

## Consequências

- Autorização, deleção e atualização ficam compreensíveis por corpus.
- Fontes oficiais podem ter política mais estrita que conteúdo editorial.
- Há custo operacional de múltiplos índices e de uma pipeline editorial.
- Qualidade depende de metadados e revisão, não apenas de embeddings.

## Gates e rollback

Publicação requer rights review, checks de malware/formato, retrieval eval, groundedness e verificação de citações. Corpus ou versão problemática é despublicado por manifesto sem reindexar os demais. Feature flags desligam query rewriting, rerank e síntese por modelo independentemente, preservando uma resposta segura sem fonte ou um fallback editorial aprovado.

## Evidência relacionada

Ver [Insights, Recursos e RAG](../insights-resources-rag.md), [segurança e observabilidade](../security-observability.md) e [evals](../eval-strategy.md).
