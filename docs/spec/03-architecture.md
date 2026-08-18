# Wikan — Arquitetura do Sistema

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Visão arquitetural

A Wikan será um **monólito modular** com frontend Next.js, API NestJS e PostgreSQL. O monólito é uma escolha de simplicidade operacional, não uma autorização para misturar responsabilidades. Cada módulo deve possuir limites claros, contratos internos explícitos e testes próprios.

```text
┌─────────────────────────────────────────────────────────┐
│                      Browser                            │
│  páginas públicas · editor · administração · sessão     │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTPS
┌───────────────────────▼─────────────────────────────────┐
│                 Next.js Web                             │
│ SSR público · UI · acessibilidade · chamadas à API      │
└───────────────────────┬─────────────────────────────────┘
                        │ REST /api/v1
┌───────────────────────▼─────────────────────────────────┐
│                 NestJS API                              │
│ auth · users · policies · wiki · pages · revisions      │
│ admin · audit · outbox · health                         │
└──────────────┬───────────────────────┬──────────────────┘
               │                       │
       ┌───────▼────────┐      ┌───────▼─────────┐
       │ PostgreSQL     │      │ Redis           │
       │ domínio · FKs  │      │ sessão · limit  │
       │ outbox · audit │      │ locks curtos    │
       └────────────────┘      └─────────────────┘
```

## 2. Limites de módulos

| Módulo | Responsabilidade | Não deve fazer |
|---|---|---|
| `auth` | OIDC, callback, sessão, logout, revogação | Conceder role diretamente pelo payload do usuário. |
| `users` | Perfil, identidade interna, bloqueio, anonimização | Alterar membership sem policy administrativa. |
| `wiki` | Configuração e escopo da wiki | Duplicar autorização em cada tela. |
| `memberships` | Roles e estados por wiki | Tratar role como campo livre enviado pelo cliente. |
| `editor-requests` | Solicitação, aprovação, rejeição e revogação derivada | Enviar e-mail ou atualizar cache dentro da transação central. |
| `pages` | Criar, editar, publicar, excluir e restaurar páginas | Renderizar HTML arbitrário ou chamar diretamente um worker externo. |
| `revisions` | Versionamento, diff, reversão e conflito | Apagar histórico no fluxo editorial. |
| `content` | Parse, validação, sanitização e renderização segura | Confiar no HTML enviado pelo browser. |
| `redirects` | Slugs históricos e destino canônico | Criar cadeias ou ciclos de redirect. |
| `search` | Contrato `SearchProvider`, indexação e consulta | Tornar o domínio dependente de um motor externo. |
| `media` | Upload, processamento e serving de arquivos; v1.1 | Armazenar arquivo arbitrário na webroot. |
| `admin` | Comandos de governança e operações protegidas | Ser a única camada de autorização. |
| `audit` | Eventos de auditoria append-only e consulta controlada | Registrar secrets ou conteúdo sensível sem necessidade. |
| `outbox` | Persistir e despachar side effects idempotentes | Ser usado para decidir se a publicação foi aceita. |
| `health` | Liveness, readiness e dependências | Expor credenciais ou queries caras no health check. |

## 3. Camadas internas

Cada módulo deve separar quatro responsabilidades:

1. **Controller/transport:** traduz HTTP para comandos, valida DTOs e traduz erros conhecidos.
2. **Application/use case:** coordena autorização, transação e chamadas de domínio.
3. **Domain:** invariantes, entidades, políticas e eventos de negócio sem dependência de HTTP.
4. **Infrastructure:** Prisma, Redis, OIDC, storage, mensageria e integrações externas.

Um controller não deve decidir se um usuário pode editar; ele chama a policy. Um repository não deve decidir se uma revisão é válida; ele persiste uma operação já validada e também depende das constraints do banco.

## 4. Fluxo de uma publicação

A publicação de uma página nova deve seguir:

```text
HTTP request
  → autenticação da sessão
  → policy page.create
  → validação do DTO
  → parse e sanitização do conteúdo
  → transação PostgreSQL
      → criar page
      → criar page_revision v1
      → atualizar current_revision_id
      → criar audit_log
      → criar outbox_event PagePublished
  → commit
  → resposta 201
  → worker processa index/cache/notification
```

Se o commit falhar, nenhum objeto editorial central pode ser considerado publicado. Se o worker falhar, a página continua legível e o evento é reprocessado. A resposta de sucesso não deve depender de busca externa ou cache.

## 5. Fluxo de uma edição concorrente

A leitura da página retorna a revisão atual. O editor submete `expectedRevisionId`. O use case abre uma transação, bloqueia a linha da página, compara a revisão, cria a próxima versão e atualiza o ponteiro. Um segundo request com a revisão antiga recebe `409` e a resposta inclui a revisão atual suficiente para a UI montar o conflito.

O lock deve durar apenas a transação de gravação. Nunca manter lock enquanto o usuário digita, faz upload ou revisa preview.

## 6. Outbox e side effects

A tabela `outbox_events` faz parte da transação que altera o domínio. Cada evento contém tipo, agregado, payload mínimo, chave de idempotência, tentativas e timestamps. Um worker lê eventos não processados e chama handlers:

| Evento | Handlers possíveis |
|---|---|
| `PagePublished` | Indexação, invalidação de cache e notificações futuras. |
| `PageReverted` | Indexação, auditoria derivada e notificação futura. |
| `EditorRequestApproved` | Notificação in-app futura e métricas. |
| `UserBlocked` | Revogação de sessão, cache de policy e alertas. |
| `PageSlugChanged` | Atualização de sitemap, cache e redirect. |

Cada handler deve ser idempotente por `event_id` ou chave equivalente. A falha deve aumentar `attempts`, registrar o motivo sem dados sensíveis e usar backoff. Após o limite de tentativas, o evento deve ficar visível para operação sem ser descartado.

## 7. Transações obrigatórias

| Caso de uso | Operações que devem ser atômicas |
|---|---|
| Bootstrap | Criar/promover membership e registrar auditoria. |
| Aprovação | Alterar request, criar membership, registrar auditoria. |
| Revogação | Alterar membership, invalidar estado lógico e registrar auditoria. |
| Publicação | Página, revisão, ponteiro atual, auditoria e outbox. |
| Edição | Verificação da revisão, nova revisão, ponteiro atual, auditoria e outbox. |
| Reversão | Leitura da revisão base, nova revisão, ponteiro atual, auditoria e outbox. |
| Soft delete | Status da página, auditoria e outbox de indexação. |
| Mudança de slug | Novo slug, redirect antigo, auditoria e outbox. |

Redis, busca externa e CDN não participam da transação PostgreSQL. Eles são atualizados por eventos reprocessáveis.

## 8. Isolamento da futura multi-wiki

A entidade `Wiki` existe no MVP mesmo com uma única instância pública. Toda página, revisão, categoria futura, membership, request e auditoria deve ser alcançável pelo `wiki_id` ou por uma relação que permita inferi-lo sem ambiguidade. Policies devem receber o escopo da wiki; não confiar somente no `user_id` global.

Não construir uma UI de seleção de wiki agora. O objetivo é evitar migração estrutural futura sem criar o custo de uma plataforma multi-tenant completa antes da hora.

## 9. Deploy e fronteiras operacionais

O frontend e a API podem ser publicados em uma imagem conjunta no início, desde que os processos tenham health checks e logs separados. A separação física pode ocorrer depois sem alterar os contratos. PostgreSQL e Redis devem possuir redes privadas e credenciais próprias.

A API deve possuir:

- `/health/live`: verifica apenas se o processo está respondendo;
- `/health/ready`: verifica dependências necessárias para aceitar tráfego;
- `/metrics`: protegido ou exposto somente à rede de observabilidade;
- logs de startup sem secrets;
- shutdown gracioso para terminar requests e drenar workers.

## 10. Decisões que exigem ADR

Devem ser registradas em `docs/spec/adr/` as mudanças de: monólito para serviços separados; PostgreSQL para outro banco; sessão server-side para outro modelo; snapshot de revisão para delta; busca PostgreSQL para motor externo; ou mudança do escopo de `Wiki` e memberships.

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
