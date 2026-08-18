# Wikan — Overrides Normativos e Correções de Arquitetura

**Status:** normativo e prioritário para implementação
**Versão:** 1.1
**Última revisão:** 18 de agosto de 2026

> Este arquivo tem precedência sobre qualquer regra conflitante em `docs/spec/01-*` até `docs/spec/15-*`.
> Ele existe para fechar ambiguidades encontradas na auditoria arquitetural sem apagar o histórico das decisões anteriores.

## 1. Precedência e regra de conflito

Quando dois documentos normativos divergirem:

1. este arquivo vence;
2. depois `03-architecture.md`, `05-database.md`, `04-domain-and-authorization.md` e `07-api-contract.md` na ordem indicada;
3. ADRs posteriores vencem decisões antigas quando explicitamente aprovados.

Nenhuma implementação pode escolher silenciosamente a interpretação mais permissiva de uma regra ambígua.

---

## 2. Stack fechada

A stack do MVP fica determinística:

| Componente | Decisão |
|---|---|
| Node.js | **Node 24 LTS** no desenvolvimento e produção. Atualizações dentro da major por patch/minor; troca de major exige ADR. |
| Next.js | **Next.js 16.x**. Não usar `15 ou superior` como faixa aberta. |
| React | Versão compatível e fixada pelo lockfile da versão adotada pelo Next.js. |
| NestJS | **NestJS 11.x**. |
| TypeScript | `strict: true`, `noUncheckedIndexedAccess`, `noImplicitOverride`; `exactOptionalPropertyTypes` quando o bootstrap estiver compatível. |
| PostgreSQL | PostgreSQL 16+. |
| Redis | Redis 7+. |
| Testes backend | **Vitest**. Não manter Vitest/Jest como escolha aberta. |
| E2E | Playwright. |

O `package.json` raiz deve declarar `engines.node` e o gerenciador de pacotes deve ser fixado por `packageManager`. O Dockerfile deve usar a mesma major de Node.

A política de upgrade é: patch/minor dentro da janela normal; major por ADR com testes, benchmark quando relevante, revisão de security impact e plano de rollback.

---

## 3. Topologia canônica de produção

Para reduzir ambiguidades de cookie, CSRF e CORS, o MVP usa **same-origin público** atrás de reverse proxy:

```text
https://wikan.example/
        │
        ├── /            → Next.js
        └── /api/v1/*    → NestJS
```

A API não fica exposta como uma origem pública independente no fluxo normal do browser.

Consequências normativas:

- cookie de sessão é host-only por padrão;
- CORS para o browser não é necessário no fluxo principal;
- CSRF continua obrigatório porque a sessão usa cookie;
- `Origin`/`Referer` são validados junto ao token CSRF conforme a política de segurança;
- integrações server-to-server podem usar outra origem interna sem herdar automaticamente permissões do browser.

---

## 4. Redis: o que é dependência crítica e o que é degradável

Redis é utilizado para:

- sessões revogáveis;
- rate limiting;
- locks operacionais muito curtos quando explicitamente necessários.

**Redis não é usado para concorrência editorial de páginas.** A fonte de verdade para conflito de revisão e locking editorial é PostgreSQL.

### Falha do Redis

Se Redis estiver indisponível:

- não criar novas sessões;
- revogar/logout de sessões existentes continua sendo tentado quando possível e deve falhar fechado para ações sensíveis;
- bloquear operações administrativas e editoriais que dependam de sessão válida se o store de sessão não puder confirmar o ator;
- leitura pública por slug pode continuar disponível se não depender de Redis;
- rate limiting pode entrar em modo conservador local apenas em leitura pública, nunca usar fallback local para autorização;
- `/health/live` continua `200` se o processo estiver vivo;
- `/health/ready` só falha quando uma dependência necessária para o tipo de tráfego que será recebido estiver indisponível.

O readiness deve expor checks separados, por exemplo:

```text
live
ready_public_read
ready_authenticated_write
```

O balanceador só envia tráfego de escrita autenticada quando `ready_authenticated_write` estiver saudável.

---

## 5. Sessão e OIDC

O fluxo OIDC é explicitamente:

```text
start
  → gerar state + nonce + PKCE
  → persistir challenge/transação temporária server-side
  → redirect Google
callback
  → validar state
  → validar nonce
  → trocar code com PKCE
  → validar issuer/audience/assinatura/expiração
  → obter provider_sub
  → buscar UserIdentity(provider, provider_sub)
  → criar ou reutilizar User
  → rotacionar sessão
```

E-mail não reassocia automaticamente identidade.

A store temporária do fluxo OIDC deve ter TTL curto e ser vinculada ao contexto necessário para impedir replay.

O login deve criar uma sessão nova e invalidar a sessão pré-login para evitar session fixation.

---

## 6. Primeiro administrador e proteção do último administrador

Cada `Wiki` com `status = active` deve possuir **pelo menos um admin ativo**.

As seguintes operações são proibidas quando deixariam zero admins ativos:

- revogar membership do último admin;
- rebaixar o último admin;
- bloquear o último admin;
- anonimizar o último admin sem transferência prévia;
- suspender o último admin;
- excluir a única membership administrativa.

A verificação deve ocorrer **dentro da mesma transação PostgreSQL** que modifica a membership/estado.

### Autoações administrativas

Por padrão, um admin não pode:

- revogar a própria membership administrativa;
- bloquear a própria conta;
- rebaixar a própria role quando isso eliminar o último admin.

Uma operação de emergência só pode existir por comando operacional documentado e nunca como endpoint HTTP público.

---

## 7. Precedência de estado

A autorização é avaliada nesta ordem:

```text
1. identidade/sessão válida
2. user.status
3. wiki.status
4. membership.status
5. membership.role
6. resource status
7. policy específica da ação
```

Estados bloqueados/anônimos não recuperam permissão por possuírem role anterior.

Exemplo:

```text
user.status = blocked
membership.role = admin
→ efetivamente sem autorização para operações normais
```

A implementação deve centralizar essa precedência para evitar divergências entre policies.

---

## 8. Anonimização e autoria histórica

A estratégia oficial é utilizar um **tombstone user** por wiki ou global de sistema, denominado internamente algo equivalente a `deleted-user`, para preservar referências históricas sem PII.

Regras:

- `page_revisions.author_id` continua `NOT NULL`;
- antes de concluir anonimização, todas as referências editoriais do usuário são transferidas para o tombstone correspondente;
- o tombstone não pode autenticar;
- o tombstone não possui identidade OIDC;
- o tombstone não pode receber permissões editoriais;
- a UI mostra `Usuário removido`;
- auditoria sensível mantém apenas o mínimo necessário e deve seguir retenção própria.

Não usar `SET NULL` em `page_revisions.author_id`.

A operação de anonimização é transacional por domínio, invalidando sessões antes de remover identidades.

---

## 9. Histórico e visibilidade

Histórico possui níveis de visibilidade:

| Estado | Visitante | User/Editor | Admin |
|---|---|---|---|
| Página publicada | Sim | Sim | Sim |
| Revisões públicas de página publicada | Sim | Sim | Sim |
| Página soft-deleted | Não | Não | Sim |
| Revisões de página soft-deleted | Não | Não | Sim |
| Draft | Nunca | Apenas autor autorizado | Admin se a policy permitir |
| Conteúdo redigido por privacidade | Apenas versão redigida, se publicada | Conforme policy | Admin somente quando legalmente necessário |

Uma exclusão ou anonimização não transforma automaticamente todo o histórico em público.

Se uma revisão contiver PII que precise ser removida por obrigação válida de privacidade, a operação é uma **redação de revisão**, não uma edição silenciosa do passado. A existência da redação e seu motivo ficam auditados; o conteúdo substituído não aparece para usuários comuns.

---

## 10. Namespace de slugs e redirects

Slug atual e redirect pertencem ao mesmo namespace lógico da wiki.

O estado permitido é:

```text
slug livre
slug ocupado por página
slug ocupado por redirect
```

Nunca podem existir simultaneamente:

```text
page(wiki_id, slug = X)
redirect(wiki_id, from_slug = X)
```

A implementação preferida é uma tabela de aliases/route bindings compartilhando unicidade por `(wiki_id, slug)`, com referência opcional para página atual. Se a implementação mantiver tabelas separadas, deve existir mecanismo transacional de reserva do namespace e teste de concorrência que impeça a colisão.

Redirects antigos devem apontar diretamente para a página canônica atual.

---

## 11. Integridade das revisões no banco

Além das constraints já previstas, a implementação deve garantir que:

- `pages.current_revision_id` pertence à própria página;
- `page_revisions.based_on_revision_id` pertence à mesma página;
- `page_revisions.version` é monotonicamente crescente por página;
- uma página publicada não pode ficar sem revisão atual;
- uma página deletada continua com revisão atual válida para restauração.

Quando possível, usar FKs compostas ou chaves únicas auxiliares para fazer o PostgreSQL também verificar essas relações, em vez de depender somente da aplicação.

A constraint `unique(page_id, content_hash, version)` não é necessária e não deve ser usada como mecanismo de deduplicação. `content_hash` é metadado de integridade/deduplicação informativa; reversões idênticas ao conteúdo atual continuam sendo revisões válidas.

---

## 12. Estados do banco

Estados críticos devem ser fechados por `CHECK` constraints ou enums.

### Wiki

```text
active
suspended
```

### User

```text
active
blocked
pending_deletion
anonymized
```

### Membership

```text
active
revoked
```

### Editor request

```text
pending
approved
rejected
cancelled
expired
```

### Page

```text
published
deleted
```

O banco é a última barreira contra valores inválidos.

---

## 13. Idempotência da API

`Idempotency-Key` é **somente header**. Não aceitar `idempotencyKey` paralelo no JSON.

Para mutations idempotentes, persistir uma entrada com pelo menos:

```text
idempotency_key
actor_id
wiki_id
method
route
request_hash
status
response_status
response_body_redacted
created_at
expires_at
```

Regras:

1. mesma chave + mesmo ator + mesmo contrato + mesmo hash → repetir resposta original;
2. mesma chave com payload diferente → `409 IDEMPOTENCY_KEY_REUSE`;
3. chave de outro ator não pode ser reutilizada;
4. resposta original precisa ser persistida de forma segura para replay;
5. expiração deve ser definida por operação;
6. operações editoriais dentro da transação devem registrar a chave antes de produzir o commit definitivo, ou usar locking equivalente para impedir dupla execução.

Não usar apenas Redis para a garantia de idempotência de publicação/aprovação; a proteção precisa sobreviver a restart e depender da fonte de verdade durável.

---

## 14. Outbox: claiming, lease e dead letter

A tabela de outbox usa workers concorrentes com claiming PostgreSQL.

Padrão obrigatório ou equivalente:

```sql
SELECT ...
FROM outbox_events
WHERE processed_at IS NULL
  AND available_at <= now()
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT :batch;
```

Cada evento reivindicado deve possuir uma lease/visibility timeout. Se o worker morrer, o evento volta a ser elegível depois da lease.

Após o limite de tentativas:

```text
pending → retrying → dead
```

Eventos `dead` não são apagados. Devem aparecer em dashboard operacional e possuir comando seguro para replay depois da correção.

O handler deve ser idempotente por `event_id`.

Shutdown gracioso deve:

1. parar de reivindicar novos eventos;
2. concluir um lote ativo dentro do timeout;
3. devolver eventos não concluídos à fila por expiração de lease;
4. sair sem marcar eventos como processados indevidamente.

---

## 15. Worker separado logicamente

Mesmo que API e worker usem a mesma imagem Docker, o processo de worker deve ser separado por comando operacional:

```text
web
api
worker
```

Isso permite escalar requests e side effects independentemente sem alterar os módulos de domínio.

---

## 16. Multi-wiki e contexto da API

A wiki deixa de ser implicitamente opcional em rotas cujo recurso pertence a uma wiki.

Para o MVP, o contexto da wiki pode ser fixado pela configuração da instância pública, mas o contrato interno deve receber `wiki_id` explicitamente.

Quando houver mais de uma wiki pública, URLs e rotas devem usar contexto explícito, por exemplo:

```text
/w/:wikiSlug/wiki/:pageSlug
/api/v1/wikis/:wikiId/pages/:pageId
```

Não permitir fallback silencioso para "primeira wiki" ou "wiki padrão" em queries de autorização.

---

## 17. Histórico e redirects após mudança de slug

Links internos armazenam o alvo lógico da página. Redirect é compatibilidade para URLs antigas e links externos.

Portanto:

- renomear slug não quebra links internos lógicos;
- URLs antigas continuam funcionando por 301;
- o render não deve depender de seguir um redirect HTTP para resolver link interno;
- a resolução de links internos deve validar `wiki_id` e página atual.

---

## 18. Upload e SSRF

A v1.1 mantém upload fora do MVP, mas a regra fica fechada:

- nenhum fetch remoto automático de imagens no servidor;
- URLs remotas de mídia são desabilitadas por padrão;
- se futuramente existir importação remota, usar allowlist de destino, bloqueio de IP privado/link-local, resolução DNS segura e limites de tamanho/tempo.

---

## 19. CSP

A CSP deve ser definida pelo ambiente real do Next.js, e não apenas por uma string genérica do threat model.

Antes do release, testar:

- OIDC;
- assets do Next.js;
- fontes;
- analytics, se existirem;
- imagens permitidas;
- preview de conteúdo;
- páginas públicas;
- área administrativa.

Preferir CSP com nonce/hash quando necessário para scripts legítimos. Conteúdo editorial nunca recebe permissão para executar script.

---

## 20. Métricas e privacidade

Métricas de produto e segurança devem ser minimizadas e agregadas.

Regras:

- não expor ranking social no MVP;
- usar agregação mínima de grupos quando a comunidade for pequena;
- separar métricas operacionais de perfis públicos;
- limitar quem pode ver métricas administrativas;
- não usar métricas para inferir ou expor características pessoais desnecessárias.

---

## 21. Backup, restore e outbox

Um restore de PostgreSQL deve ser acompanhado por uma estratégia explícita para outbox:

- restaurar banco e outbox juntos;
- identificar eventos cujo side effect externo já ocorreu antes do desastre;
- handlers devem ser idempotentes para replay;
- não apagar outbox por conveniência após restore;
- executar reconciliação de index/cache/notifications após recuperar o domínio.

O backup de Redis não é requisito de preservação editorial: sessões e rate limits podem ser recriados. O banco PostgreSQL é a fonte de verdade do domínio.

---

## 22. Critérios de bloqueio de release

O release deve ser bloqueado se qualquer uma destas condições existir:

- não houver proteção do último admin;
- anonimização quebrar FKs ou autoria histórica;
- idempotência de mutations críticas estiver somente em memória/Redis;
- slug e redirect puderem colidir;
- worker puder processar evento indefinidamente sem observabilidade;
- uma rota protegida puder operar por ID sem checar `wiki_id`;
- restore não puder reconstruir páginas e revisões de maneira determinística;
- Redis indisponível fizer o sistema aceitar mutation autenticada sem confirmar sessão;
- migration incompatível tornar rollback da aplicação inseguro sem plano expand/contract;
- testes negativos de autorização falharem.

---

## 23. Checklist final de implementação

Antes do primeiro release do núcleo editorial, verificar:

- [ ] Node/Next/Nest e test runner estão fixados.
- [ ] Same-origin está implementado.
- [ ] Sessão server-side está revogável.
- [ ] OIDC usa state/nonce/PKCE.
- [ ] Existe pelo menos um admin ativo por wiki.
- [ ] Auto-revogação administrativa é bloqueada.
- [ ] `blocked` sempre vence membership.
- [ ] Tombstone resolve autoria após anonimização.
- [ ] Histórico possui política de visibilidade.
- [ ] Namespace de slug/redirect é único.
- [ ] Constraints de revisão impedem referências cruzadas.
- [ ] Idempotency-Key tem persistência durável.
- [ ] Outbox tem claiming, lease, retry e dead state.
- [ ] Worker pode ser escalado separadamente.
- [ ] API não depende de wiki implícita em autorização.
- [ ] CSP foi testada no ambiente real.
- [ ] Redis possui comportamento degradado seguro.
- [ ] Restore inclui reconciliação do outbox.

