# Wikan — Contrato da API REST

**Status:** normativo para o MVP  
**Versão:** `/api/v1`  
**Última revisão:** 18 de agosto de 2026

## 1. Princípios

A API é a fronteira obrigatória entre o frontend e o backend. Todas as mutations passam por autenticação, autorização, validação e transação apropriadas. A API deve ser previsível, versionada, observável e segura para chamadas repetidas quando a operação for idempotente.

O contrato completo deve ser publicado em OpenAPI e validado no CI contra os DTOs implementados. Este documento define o comportamento mínimo; schemas OpenAPI e exemplos podem detalhar tipos sem alterar as regras abaixo.

## 2. Convenções HTTP

| Convenção | Regra |
|---|---|
| Base URL | `/api/v1`. |
| Conteúdo | JSON UTF-8, salvo upload v1.1. |
| Autenticação | Sessão server-side por cookie seguro; API não aceita role no body. |
| Datas | ISO 8601 UTC. |
| IDs | UUID/ULID opaco. |
| Erros | Objeto padronizado com `code`, `message`, `details` opcional e `requestId`. |
| Idempotência | Header `Idempotency-Key` em comandos administrativos e operações que possam ser repetidas. |
| Conflito | `409` para revisão, slug ou estado concorrente. |
| Rate limit | `429` com `Retry-After`. |
| Cache | GET público pode usar `ETag`; mutations invalidam derivados por evento. |

## 3. Formato de erro

```json
{
  "code": "PAGE_REVISION_CONFLICT",
  "message": "A página foi alterada por outro editor.",
  "details": {
    "currentRevisionId": "2fe2...",
    "expectedRevisionId": "1ac4..."
  },
  "requestId": "req_01J..."
}
```

`message` deve ser seguro para o usuário. `details` não pode conter stack trace, query, token, e-mail completo ou segredo. Códigos de erro são estáveis e documentados; o texto pode ser traduzido no frontend.

## 4. Autenticação e sessão

| Método | Rota | Acesso | Comportamento |
|---|---|---|---|
| `GET` | `/auth/google/start` | Público | Inicia fluxo OIDC com state, nonce e PKCE. |
| `GET` | `/auth/google/callback` | Público, state obrigatório | Valida retorno, encontra/cria identity e abre sessão. |
| `POST` | `/auth/logout` | Autenticado | Invalida sessão server-side e limpa cookie. |
| `GET` | `/me` | Autenticado | Retorna usuário público, memberships e flags próprias. |
| `GET` | `/me/activity` | Autenticado | Retorna atividade paginada do próprio usuário. |

O callback não deve confiar em `email` para reassociar conta. Falhas OIDC recebem resposta genérica e log de segurança com request ID, sem revelar qual identidade falhou.

## 5. Páginas e revisões

| Método | Rota | Acesso | Entrada principal |
|---|---|---|---|
| `GET` | `/pages/:slug` | Público | `wiki`, opcional; retorna página publicada ou redirect. |
| `GET` | `/pages/:id` | Conforme status | Admin pode consultar excluída; público só publicada. |
| `GET` | `/pages/:id/revisions` | Público conforme política | `cursor`, `limit`; histórico paginado. |
| `GET` | `/pages/:id/revisions/:revisionId` | Público conforme política | Conteúdo e metadata da revisão. |
| `POST` | `/pages` | Editor/Admin | `title`, `content`, `summary`, `slug` opcional. |
| `PUT` | `/pages/:id` | Editor/Admin | `title`, `content`, `summary`, `requestedSlug`, `expectedRevisionId`. |
| `POST` | `/pages/:id/revert` | Editor permitido/Admin | `revisionId`, `reason`, `idempotencyKey`. |
| `DELETE` | `/pages/:id` | Admin | `reason`, confirmação explícita. |
| `POST` | `/pages/:id/restore` | Admin | `reason`, confirmação explícita. |
| `GET` | `/pages/:id/diff` | Público conforme política | `fromRevisionId`, `toRevisionId`. |

`POST /pages` retorna `201` com page e revision. `PUT` retorna `200` quando cria revisão. `DELETE` e `restore` retornam o recurso atualizado. Operações repetidas com a mesma `Idempotency-Key` devem retornar o resultado original sem duplicar revisão ou auditoria.

## 6. Solicitações de editor

| Método | Rota | Acesso | Regras |
|---|---|---|---|
| `POST` | `/editor-requests` | Usuário | Uma solicitação pendente por wiki; rate limit. |
| `GET` | `/editor-requests/me` | Usuário | Próprios requests, paginados. |
| `GET` | `/admin/editor-requests` | Admin | Filtros por status, cursor e limite. |
| `POST` | `/admin/editor-requests/:id/approve` | Admin | Transação idempotente. |
| `POST` | `/admin/editor-requests/:id/reject` | Admin | Exige motivo. |
| `POST` | `/admin/editor-requests/:id/cancel` | Usuário/Admin | Somente quando estado permitir. |

A API não deve expor a existência de requests de outro usuário para não-admin. Responses administrativas incluem contexto necessário, mas não devem vazar secrets ou PII fora da finalidade.

## 7. Memberships e administração

| Método | Rota | Acesso | Efeito |
|---|---|---|---|
| `GET` | `/admin/users` | Admin | Lista paginada e filtrável. |
| `GET` | `/admin/users/:id` | Admin | Perfil operacional mínimo e memberships. |
| `POST` | `/admin/memberships/:id/revoke` | Admin | Revoga membership e audita. |
| `POST` | `/admin/users/:id/block` | Admin | Bloqueia e invalida sessões. |
| `POST` | `/admin/users/:id/unblock` | Admin | Desbloqueia e audita. |
| `GET` | `/admin/audit-logs` | Admin | Consulta paginada com filtros e redaction. |

Não existe rota pública para promover admin. Bootstrap ocorre pelo comando operacional descrito no spec de domínio e operações.

## 8. Paginação e filtros

Listagens devem aceitar `limit` limitado pelo servidor e cursor opaco. A resposta deve conter:

```json
{
  "items": [],
  "pageInfo": {
    "nextCursor": "eyJjcmVhdGVkQXQiOi...",
    "hasNextPage": true
  },
  "requestId": "req_01J..."
}
```

O cursor deve incluir a ordenação necessária e não revelar query interna. `limit` padrão e máximo devem ser definidos por endpoint. Ordenação precisa ser estável, com tie-breaker por UUID ou timestamp.

## 9. Códigos de erro mínimos

| Código | HTTP | Uso |
|---|---:|---|
| `AUTH_REQUIRED` | 401 | Sessão ausente ou inválida. |
| `ACCESS_DENIED` | 403 | Sessão válida sem policy. |
| `RESOURCE_NOT_FOUND` | 404 | Recurso não existe ou não é visível ao ator. |
| `VALIDATION_ERROR` | 422 | Entrada inválida. |
| `PAGE_SLUG_CONFLICT` | 409 | Slug usado ou reservado. |
| `PAGE_REVISION_CONFLICT` | 409 | `expectedRevisionId` desatualizado. |
| `STATE_TRANSITION_INVALID` | 409 | Request ou membership em estado incompatível. |
| `IDEMPOTENCY_REPLAY` | 200/409 | Repetição coerente ou chave reutilizada com payload divergente. |
| `RATE_LIMITED` | 429 | Limite excedido. |
| `DEPENDENCY_UNAVAILABLE` | 503 | Dependência necessária indisponível. |

## 10. Segurança do contrato

Cada rota deve declarar publicamente seu requisito de autenticação e policy. O backend deve rejeitar campos desconhecidos em mutations sensíveis. IDs na URL nunca substituem checagem de escopo.

Métodos GET não devem produzir side effects. Mutations com cookie exigem CSRF. Payloads têm limite de bytes, profundidade e número de itens. O servidor deve aplicar timeout e não manter conexão aberta indefinidamente para requests comuns.

## 11. Versionamento

Mudança incompatível exige `/api/v2` ou estratégia de compatibilidade documentada. Adicionar campo opcional é compatível; remover campo, alterar semântica, trocar código de erro ou mudar autorização é breaking change e exige ADR, changelog e período de migração.

## 12. Critérios de aceite

| ID | Critério |
|---|---|
| API-01 | A especificação OpenAPI representa todas as rotas e passa validação no CI. |
| API-02 | Rotas protegidas retornam `401` sem sessão e `403` sem policy. |
| API-03 | Campos protegidos não são aceitos por DTOs públicos. |
| API-04 | Listagens usam paginação e não aceitam limite arbitrário. |
| API-05 | Repetir comando idempotente não duplica revisão, membership ou auditoria. |
| API-06 | Conflitos de edição retornam `409` com dados seguros para recuperação da UI. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
