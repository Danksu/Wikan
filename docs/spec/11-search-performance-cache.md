# Wikan — Busca, Performance e Cache

**Status:** normativo para o MVP e v1.1  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Princípio

Performance deve ser medida em fluxos representativos, não presumida por uma lista de tecnologias. O MVP deve começar com PostgreSQL, índices corretos e paginação. Cache, CDN e motor externo somente entram quando houver benefício demonstrado e invalidação definida.

## 2. Busca por etapa

A busca completa pertence à v1.1. O contrato deve existir desde o MVP para que telas e casos de uso não dependam de SQL específico.

```text
interface SearchProvider {
  search(input: SearchInput): Promise<SearchPage>
  index(document: SearchDocument): Promise<void>
  remove(documentId: string): Promise<void>
}
```

### 2.1 PostgreSQL FTS

Na v1.1, indexar título, conteúdo, slug, aliases futuros e metadata permitida em estrutura compatível com `tsvector`/GIN. A busca deve:

- respeitar `wiki_id`;
- incluir somente páginas publicadas;
- retornar título, slug, trecho seguro e relevância;
- ser paginada;
- limitar tamanho e complexidade da consulta;
- possuir rate limit;
- não aceitar SQL, regex arbitrária ou filtros livres.

A indexação pode ocorrer por outbox. Enquanto o documento não estiver indexado, a página publicada continua acessível por slug; a UI deve tratar resultado de busca eventualmente atrasado como estado operacional, não como perda de conteúdo.

### 2.2 Gatilho para motor externo

Avaliar Meilisearch ou equivalente quando uma destas condições ocorrer por sete dias consecutivos:

| Gatilho | Valor |
|---|---:|
| Páginas publicadas | Mais de 50.000 |
| p95 da busca em staging/prod representativo | Acima de 1.000 ms |
| CPU do PostgreSQL atribuída à busca | Acima de 70% de forma sustentada |
| Janela de reindexação | Excede a janela operacional acordada |

A troca exige benchmark, análise de custo, reindexação completa, plano de fallback e ADR. A aplicação continua dependendo de `SearchProvider`, nunca de APIs do motor diretamente.

## 3. Paginação

Histórico, contribuições, requests, usuários, auditoria e resultados de busca devem possuir paginação. Cursor opaco é preferido para coleções que crescem ou sofrem alterações durante navegação. Ordenação deve ser estável e incluir tie-breaker.

Limites iniciais:

| Recurso | Padrão | Máximo |
|---|---:|---:|
| Histórico | 25 | 100 |
| Busca | 20 | 50 |
| Requests | 25 | 100 |
| Auditoria | 50 | 200 |
| Contribuições | 25 | 100 |

## 4. Cache

No MVP, cache Redis é obrigatório para sessões e rate limit, mas cache de página pública é opcional. Se ativado, o cache deve ser versionado:

```text
page:{wikiId}:{slug}:revision:{revisionId}
```

Nunca usar slug sozinho como chave de conteúdo. Eventos `PagePublished`, `PageReverted`, `PageDeleted`, `PageRestored` e `PageSlugChanged` devem invalidar o cache correspondente e derivados. Cache de busca deve possuir TTL curto e ser invalidado ou reindexado por evento.

Conteúdo privado, administrativo, requests e auditoria não devem ser armazenados em cache público. Headers devem impedir CDN de compartilhar resposta de sessão.

## 5. Performance de leitura

Páginas públicas devem ser renderizadas server-side com HTML seguro e metadata. O backend deve evitar N+1 queries, carregar apenas campos necessários e usar índices. O frontend deve fazer lazy loading de elementos abaixo da dobra sem atrasar título, corpo inicial e navegação.

Imagens entram na v1.1 e devem ter dimensões, formatos e thumbnails adequados. Não enviar asset original quando thumbnail satisfaz o contexto.

## 6. Metas de referência

Em ambiente de staging representativo, medir:

| Fluxo | Meta inicial p95 |
|---|---:|
| GET página publicada sem dependência externa | < 800 ms |
| GET histórico paginado | < 1.000 ms |
| Busca FTS até 50.000 páginas | < 1.000 ms |
| Publicação editorial | < 1.500 ms no caminho síncrono, sem esperar indexação |
| API de leitura com cache quente | < 300 ms quando o cache for ativado |

As metas são gatilhos de investigação, não autorização para adicionar serviços sem análise. Registrar p50, p95, p99, taxa de erro e volume.

## 7. Testes de performance

O CI deve executar smoke tests sem carga pesada. Antes do lançamento, executar benchmark controlado com dataset sintético que represente páginas, revisões e usuários esperados. O teste deve cobrir leitura fria, leitura quente, publicação, busca, histórico e concorrência.

Queries lentas devem ser analisadas com `EXPLAIN ANALYZE`. O código não deve aceitar consulta livre de usuário sem limite de tempo, quantidade ou tamanho.

## 8. Cache e consistência

Cache é derivado e pode ficar indisponível sem corromper o domínio. Ao falhar, a aplicação consulta fonte de verdade ou responde erro temporário conforme criticidade. Nunca confirmar publicação porque cache foi atualizado; confirmar somente depois do commit PostgreSQL.

## 9. Critérios de aceite

| ID | Critério |
|---|---|
| PERF-01 | Histórico, busca e listagens usam paginação com máximo no servidor. |
| PERF-02 | Nenhuma query pública faz full scan por falta de índice em slug/status. |
| PERF-03 | Cache de página, quando ativado, inclui revisão e invalida em eventos editoriais. |
| PERF-04 | Busca não retorna draft, excluída ou conteúdo de outra wiki. |
| PERF-05 | Metas p95 são medidas em staging antes do lançamento. |
| PERF-06 | Migração para motor externo exige gatilho e ADR, não preferência subjetiva. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
