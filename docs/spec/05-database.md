# Wikan — Banco de Dados e Integridade

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Objetivo

O banco PostgreSQL é a fonte de verdade do domínio editorial. O schema deve tornar estados inválidos difíceis de representar, proteger contra corridas e preservar histórico. A aplicação não pode depender apenas de validações em memória para unicidade, integridade referencial ou concorrência.

## 2. Convenções

| Convenção | Regra |
|---|---|
| IDs | UUID ou ULID; não usar ID sequencial em URLs públicas. |
| Timestamps | `timestamptz` em UTC, preenchido pelo backend ou banco de forma consistente. |
| Nomes | Tabelas e colunas em `snake_case`; enums com valores documentados. |
| Soft delete | Entidades editoriais usam `status` e `deleted_at` quando houver necessidade de restauração. |
| Auditoria | `audit_logs` é append-only no fluxo normal. |
| Migrations | Versionadas, revisadas, executadas em staging e nunca editadas após aplicação. |
| Conteúdo | `content_markdown` é texto fonte; HTML renderizado não é fonte de verdade. |
| Hash | `content_hash` usa algoritmo criptográfico estável para deduplicação e verificação. |

## 3. Entidades do MVP

### 3.1 `wikis`

```text
id                  uuid primary key
slug                varchar(80) not null unique
name                varchar(160) not null
status              varchar(20) not null default 'active'
created_at          timestamptz not null
updated_at          timestamptz not null
```

A coluna escrita corretamente na implementação deve ser `updated_at`; o trecho acima apresenta a intenção lógica e não substitui a migration. O status mínimo é `active` e `suspended`. O MVP cria uma wiki padrão, mas não deve remover `wiki_id` das entidades filhas.

### 3.2 `users`

```text
id                  uuid primary key
status              varchar(24) not null default 'active'
display_name        varchar(120) not null
avatar_url          text null
created_at          timestamptz not null
updated_at          timestamptz not null
last_login_at       timestamptz null
anonymized_at       timestamptz null
```

Não há `email` nem `google_id` como chave primária. E-mail e provider subject ficam em `user_identities`. `display_name` deve ser validado contra nomes reservados e caracteres de controle.

### 3.3 `user_identities`

```text
id                  uuid primary key
user_id             uuid not null references users(id) on delete restrict
provider            varchar(40) not null
provider_sub        varchar(255) not null
email_snapshot      varchar(320) null
email_verified      boolean not null default false
created_at          timestamptz not null
updated_at          timestamptz not null
unique(provider, provider_sub)
```

Uma identidade pertence a exatamente um usuário. A aplicação não deve reassociar `provider_sub` automaticamente. Remover uma identidade exige que exista outro caminho de recuperação ou operação administrativa documentada.

### 3.4 `wiki_memberships`

```text
id                  uuid primary key
wiki_id             uuid not null references wikis(id) on delete restrict
user_id             uuid not null references users(id) on delete restrict
role                varchar(24) not null
status              varchar(24) not null default 'active'
granted_by          uuid null references users(id) on delete set null
granted_at          timestamptz not null
revoked_at          timestamptz null
created_at          timestamptz not null
updated_at          timestamptz not null
unique(wiki_id, user_id)
check(role in ('user','editor','admin'))
```

Membership revogada não deve ser apagada. O sistema pode impedir mais de um registro por usuário e wiki usando a linha existente e alterando o status.

### 3.5 `editor_requests`

```text
id                  uuid primary key
wiki_id             uuid not null references wikis(id) on delete restrict
user_id             uuid not null references users(id) on delete restrict
status              varchar(24) not null default 'pending'
motivation          varchar(4000) not null
experience          varchar(4000) null
intended_contribution varchar(4000) null
reviewed_by         uuid null references users(id) on delete set null
reviewed_at         timestamptz null
rejection_reason    varchar(2000) null
created_at          timestamptz not null
updated_at          timestamptz not null
```

A aplicação deve garantir um único request pendente por usuário e wiki. PostgreSQL deve reforçar essa regra com índice parcial:

```sql
create unique index editor_requests_one_pending
  on editor_requests (wiki_id, user_id)
  where status = 'pending';
```

### 3.6 `pages`

```text
id                  uuid primary key
wiki_id             uuid not null references wikis(id) on delete restrict
slug                varchar(220) not null
title               varchar(240) not null
status              varchar(24) not null default 'published'
current_revision_id uuid null
created_by          uuid not null references users(id) on delete restrict
updated_by          uuid null references users(id) on delete set null
created_at          timestamptz not null
updated_at          timestamptz not null
deleted_at          timestamptz null
unique(wiki_id, slug)
```

`current_revision_id` referencia `page_revisions(id)` e deve ser adicionada depois da criação inicial das duas tabelas ou por migration em duas etapas. Uma página publicada deve ter revisão atual não nula; a constraint pode ser reforçada após a migration de bootstrap.

### 3.7 `page_revisions`

```text
id                  uuid primary key
page_id             uuid not null references pages(id) on delete restrict
version             integer not null
content_markdown    text not null
content_hash        char(64) not null
summary             varchar(1000) not null
author_id           uuid not null references users(id) on delete restrict
based_on_revision_id uuid null references page_revisions(id) on delete restrict
created_at          timestamptz not null
unique(page_id, version)
unique(page_id, content_hash, version)
```

O histórico é snapshot integral no MVP. `based_on_revision_id` registra a origem de uma edição ou reversão, mas não substitui o conteúdo integral. Não aplicar `on delete cascade` em revisões; isso violaria a preservação histórica.

### 3.8 `page_redirects`

```text
id                  uuid primary key
wiki_id             uuid not null references wikis(id) on delete restrict
from_slug           varchar(220) not null
to_page_id          uuid not null references pages(id) on delete restrict
http_status         smallint not null default 301
created_by          uuid not null references users(id) on delete restrict
created_at          timestamptz not null
unique(wiki_id, from_slug)
check(http_status = 301)
```

A aplicação deve rejeitar `from_slug` igual ao slug atual e impedir que `to_page_id` pertença a outra wiki.

### 3.9 `audit_logs`

```text
id                  uuid primary key
wiki_id             uuid null references wikis(id) on delete set null
actor_id            uuid null references users(id) on delete set null
action              varchar(80) not null
target_type         varchar(40) not null
target_id           uuid null
result              varchar(24) not null
metadata            jsonb null
request_id          varchar(80) null
ip_hash             char(64) null
created_at          timestamptz not null
```

`metadata` deve conter somente valores necessários à investigação. Não gravar cookies, tokens, conteúdo integral de página ou e-mails completos quando um identificador pseudonimizado for suficiente.

### 3.10 `outbox_events`

```text
event_id            uuid primary key
event_type          varchar(100) not null
aggregate_type      varchar(60) not null
aggregate_id        uuid not null
idempotency_key     varchar(180) not null unique
payload             jsonb not null
attempts            integer not null default 0
available_at        timestamptz not null
processed_at        timestamptz null
last_error          text null
created_at          timestamptz not null
```

Criar índices em `(processed_at, available_at)` e `(aggregate_type, aggregate_id)`. `last_error` deve ser redigido para não conter secrets.

## 4. Foreign keys e comportamento de exclusão

| Relação | Comportamento |
|---|---|
| Página → wiki | `restrict`; uma wiki não pode ser apagada com conteúdo ligado. |
| Revisão → página | `restrict`; soft delete não remove histórico. |
| Revisão → autor | `restrict` até anonimização; depois apontar para usuário anonimizável ou `set null` com snapshot de autoria conforme decisão legal. |
| Identity → user | `restrict`; remoção exige procedimento de conta. |
| Membership → user/wiki | `restrict`; revogar altera estado. |
| Auditoria → ator | `set null`; preservar evento mesmo após anonimização. |
| Redirect → page | `restrict`; remover destino exige migração explícita de redirects. |
| Outbox → agregado | Referência lógica; o evento deve sobreviver ao soft delete. |

## 5. Índices mínimos

| Tabela | Índice |
|---|---|
| `user_identities` | `(provider, provider_sub)` unique; `(user_id)`. |
| `wiki_memberships` | `(wiki_id, role, status)`; `(user_id, status)`. |
| `editor_requests` | `(wiki_id, status, created_at)`; parcial de pendente. |
| `pages` | unique `(wiki_id, slug)`; `(wiki_id, status, updated_at)`; `(wiki_id, title)`. |
| `page_revisions` | `(page_id, version desc)`; `(page_id, created_at desc)`; `(author_id, created_at desc)`. |
| `page_redirects` | unique `(wiki_id, from_slug)`; `(to_page_id)`. |
| `audit_logs` | `(wiki_id, created_at desc)`; `(target_type, target_id, created_at desc)`; `(action, created_at desc)`. |
| `outbox_events` | `(processed_at, available_at)`; `(event_type, created_at)`. |

Índices devem ser confirmados por `EXPLAIN ANALYZE` em staging. Não criar índices duplicados por hábito; cada índice precisa ter query que o utiliza ou justificar uma constraint.

## 6. Transações e concorrência

Publicação, edição, reversão, aprovação, revogação, soft delete, restauração e mudança de slug são transações de domínio. O serviço deve usar isolamento compatível com o caso, bloquear a linha da página durante compare-and-swap e deixar o banco rejeitar unique conflicts.

Uma edição deve calcular a próxima versão dentro da transação. Não confiar em `max(version) + 1` fora de lock. Uma aprovação deve verificar o estado atual do request dentro da transação. Uma mudança de slug deve criar a nova página lógica e redirect de forma atômica.

## 7. Retenção e crescimento

O histórico não é apagado pelo fluxo normal. A tabela de revisões deve ser monitorada. Avaliar arquivamento quando exceder 10 milhões de linhas, quando backup ultrapassar o RPO operacional ou quando consultas paginadas excederem as metas de performance. Qualquer compactação deve preservar restauração determinística e receber ADR.

Logs técnicos e auditoria seguem as retenções do spec de privacidade e SLA. Particionamento de `audit_logs` por mês só deve ser ativado após benchmark ou crescimento demonstrado.

## 8. Migrations

Toda migration deve:

1. ter nome descritivo e ordem determinística;
2. ser executada em banco vazio no CI;
3. ser testada contra uma cópia representativa em staging;
4. declarar impacto de lock e duração esperada;
5. possuir rollback operacional quando possível;
6. separar mudanças incompatíveis em etapas compatíveis;
7. atualizar este documento quando alterar entidade, índice ou invariant.

## 9. Critérios de aceite

| ID | Critério |
|---|---|
| DB-01 | Dois pages com o mesmo slug na mesma wiki não podem ser persistidos. |
| DB-02 | Requests pendentes duplicados são rejeitados pela aplicação e pelo índice parcial. |
| DB-03 | Duas edições concorrentes não criam a mesma versão nem apagam uma revisão. |
| DB-04 | Soft delete mantém revisões e permite restauração. |
| DB-05 | Uma identidade OIDC pode atualizar e-mail sem duplicar usuário. |
| DB-06 | Uma migration em banco vazio cria schema e índices sem intervenção manual. |
| DB-07 | Auditoria e outbox da publicação são persistidos na mesma transação do domínio. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
