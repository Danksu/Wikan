# Wikan — Auditoria crítica da arquitetura e especificação

**Data:** 18 de agosto de 2026  
**Escopo:** cada arquivo de `docs/spec/` foi analisado individualmente usando o checklist de auditoria crítica (arquitetura, dados, segurança, performance, UX, concorrência, escalabilidade, DevOps, testes e coerência de regras), seguido de uma revisão cruzada entre os documentos.

> Esta auditoria identifica problemas e riscos. Ela **não altera os documentos normativos** automaticamente.

## Resumo executivo

A nova arquitetura está muito mais madura do que uma especificação inicial: monólito modular, PostgreSQL como fonte de verdade, sessões server-side, policies, revisões imutáveis, optimistic concurrency, outbox, auditoria, threat model, backup/restore e gates de qualidade estão bem definidos.

Os principais riscos restantes não são de "falta de arquitetura", mas de **contratos ainda ambíguos ou contraditórios entre documentos**. Os itens que devem ser resolvidos antes da implementação do núcleo são:

1. **Anonimização vs `page_revisions.author_id NOT NULL`** — há conflito direto entre o modelo de privacidade e o schema.
2. **Sessão/Redis e modo degradado** — Redis é obrigatório para sessão e rate limit, mas o comportamento operacional quando Redis cai ainda deixa decisões importantes abertas.
3. **Idempotência da API** — o conceito está documentado, mas não existe ainda um modelo persistente claro para armazenar/reproduzir resultados de `Idempotency-Key`.
4. **Namespace de slugs e redirects** — o banco protege `pages(wiki_id, slug)` e `page_redirects(wiki_id, from_slug)` separadamente, mas não existe constraint que impeça colisão entre slug atual e redirect.
5. **Regras de administração** — falta explicitar proteção contra remoção/bloqueio do último admin, auto-revogação acidental e perda total de governança.
6. **Outbox worker** — a arquitetura define o outbox, mas não fecha como múltiplos workers fazem claiming/leases, como ocorre shutdown, e como eventos permanentemente falhos são operacionalmente tratados.
7. **Multi-wiki** — `wiki_id` está bem presente no domínio, mas alguns contratos públicos ainda deixam `wiki` opcional, o que cria ambiguidade de URL e de autorização.
8. **Histórico público** — é necessário definir com precisão quando revisões antigas deixam de ser públicas, especialmente após soft delete, anonimização ou conteúdo que posteriormente se torna sensível.
9. **Stack está parcialmente aberta** — em 18/08/2026 Node 24 é LTS e Node 26 é Current; o documento fixa Node 22 como baseline sem explicar a estratégia de suporte. O próprio Node recomenda versões LTS para produção. citeturn0search2
10. **Next.js está em 16.x Active LTS em agosto de 2026**, enquanto o documento aceita "15 ou superior". Isso deixa uma faixa de versões com comportamento diferente e pode permitir acidentalmente uma versão não desejada; o ideal é fixar uma major suportada no lockfile/engines. citeturn1search1turn2search7

---

# Auditoria individual — 01-product-and-scope.md

### P1 — Licença editorial está sendo decidida como requisito de produto sem resolver governança de terceiros
`09-privacy-and-governance.md` fixa CC BY-SA 4.0, enquanto este documento trata governança como princípio. Isso precisa de uma decisão explícita sobre conteúdo importado, conteúdo de terceiros, atribuição, compatibilidade de fontes e processo de takedown.

**Risco:** conteúdo pode ser publicado sem que a comunidade saiba se possui direito de relicenciar aquele material.

**Correção:** transformar licença em decisão de governança com política de contribuição, atribuição, remoção por copyright e tratamento de material incompatível.

### P1 — "Histórico público" precisa de política de visibilidade
O escopo inclui histórico público, mas não define claramente o que ocorre com revisões antigas após exclusão, anonimização, conteúdo abusivo ou incidente de privacidade.

**Correção:** definir estados de visibilidade para página/revisão e regras de acesso ao histórico.

### P2 — Métricas podem se tornar identificáveis mesmo sem PII direta
"Editores ativos", reversões e tempo até decisão podem permitir inferências sobre pessoas em comunidades pequenas.

**Correção:** definir agregação mínima, retenção e acesso às métricas.

---

# Auditoria individual — 02-stack.md

### P1 — Baseline Node 22 precisa de política de manutenção
Em 18/08/2026, Node 24 é LTS e Node 26 é Current; Node 22 também aparece como LTS no cronograma oficial. O problema não é Node 22 ser inválido, mas o documento dizer "LTS atual" e simultaneamente fixar 22 sem explicar por quê. citeturn0search2

**Correção:** fixar `engines` e imagem Docker em uma major LTS explícita e definir política de upgrade.

### P1 — "Next.js 15 ou superior" é aberto demais
Next.js 16 é Active LTS e 15 está em Maintenance LTS em agosto de 2026. Aceitar "15 ou superior" pode produzir builds diferentes e comportamento de cache/roteamento diferente entre ambientes. citeturn1search1turn2search7

**Correção:** escolher uma major, fixar versão via lockfile e `packageManager`/`engines`, e atualizar por ADR quando houver breaking change.

### P2 — Test runner indefinido
"Vitest ou Jest" é compatível com a ideia, mas impede uma especificação totalmente determinística.

**Correção:** escolher um no bootstrap e registrar no documento imediatamente.

### P1 — OIDC ainda depende de comportamento de biblioteca não escolhido
A tabela exige capacidades corretas, mas não define quem gerencia state/nonce/PKCE, como secrets são armazenados, nem como chaves/JWKS são rotacionadas/cacheadas.

**Correção:** ADR ou escolha concreta de biblioteca e fluxo.

### P2 — OpenAPI "a partir dos contratos" pode divergir dos DTOs
É necessário um gate que compare schema publicado, implementação e exemplos.

---

# Auditoria individual — 03-architecture.md

### P1 — Outbox worker não tem mecanismo de claiming/lease especificado
Com duas instâncias da API, ambas podem tentar processar o mesmo evento. Idempotência reduz dano, mas não substitui um mecanismo eficiente de concorrência.

**Correção:** definir `SELECT ... FOR UPDATE SKIP LOCKED`, lease/visibility timeout ou mecanismo equivalente, além de política de retry.

### P1 — Redis locks aparecem sem necessidade clara
O desenho mostra Redis para "locks curtos", enquanto a concorrência editorial é resolvida com transação e lock PostgreSQL. Dois mecanismos de locking para o mesmo domínio podem gerar inconsistência conceitual.

**Correção:** remover Redis locks do domínio editorial ou documentar exatamente quais locks são somente operacionais e por quê.

### P1 — Fronteira Next.js → NestJS precisa fechar o modelo de cookies/origens
Se frontend e API forem origens diferentes, entram CORS, SameSite, CSRF e propagação de cookies/request IDs. Se forem mesma origem por reverse proxy, o desenho fica mais simples.

**Correção:** definir topologia de produção e origem canônica antes da implementação.

### P1 — Health readiness pode impedir recuperação quando Redis está degradado
O documento diz que `/health/ready` verifica dependências necessárias, mas o documento de observabilidade permite modo degradado quando Redis cai.

**Correção:** separar dependências críticas de dependências degradáveis e definir exatamente quando readiness deve falhar.

### P2 — Worker e API no mesmo processo dificultam operação
Não é necessariamente errado, mas workers de outbox podem competir por CPU/memória com requests.

**Correção:** manter mesmo código/módulo, porém permitir processo separado de worker desde o início, mesmo que a imagem seja a mesma.

---

# Auditoria individual — 04-domain-and-authorization.md

### P0 — Falta proteção explícita contra perda do último administrador
A matriz permite `admin` gerenciar memberships e usuários, mas não existe regra impedindo remover/bloquear/rebaixar todos os admins.

**Correção:** invariant `cada wiki ativa deve possuir pelo menos um admin ativo`, com proteção transacional.

### P1 — Auto-revogação de admin não está definida
Um admin pode potencialmente revogar a própria membership ou bloquear a própria conta.

**Correção:** negar auto-demotion/auto-block administrativo, ou exigir procedimento especial de recuperação/confirmação.

### P1 — Bloqueio de usuário e membership precisam de precedência formal
Um usuário `blocked` pode continuar tendo membership `admin/editor`, mas as policies precisam deixar explícito que `blocked` vence qualquer membership.

**Correção:** documentar precedência de estados: `user.status` → membership status/role → resource policy.

### P1 — Anonimização não está alinhada ao schema
Este documento diz que autoria vira `Usuário removido`, mas `page_revisions.author_id` é `NOT NULL` no documento de banco.

**Correção:** escolher uma única estratégia: usuário tombstone/anônimo permanente, ou `author_id NULL` com snapshot/label histórico. Não deixar a decisão para "conforme decisão legal" no schema normativo.

### P2 — Admin global vs admin por wiki precisa ser explicitamente resolvido
O documento fala em admin por wiki, mas bootstrap/recuperação podem sugerir uma autoridade operacional global. Isso deve ser separado de role de conteúdo.

---

# Auditoria individual — 05-database.md

### P0 — `page_revisions.author_id NOT NULL` contradiz anonimização
O FK é `ON DELETE RESTRICT`, mas a seção de anonimização permite "set null" posteriormente. Isso é impossível com a coluna `NOT NULL` sem alteração de schema.

**Correção:** usar um usuário tombstone `deleted-user` ou tornar `author_id` nullable e armazenar a representação histórica separadamente.

### P1 — Constraint de estados está incompleta
`status` de `wikis`, `users`, `memberships`, `editor_requests` e `pages` é `varchar`, mas somente `role` possui `CHECK` explícito no trecho. Aplicação pode criar estados inválidos.

**Correção:** CHECK constraints ou enums para estados críticos.

### P1 — Slug atual e redirect ocupam namespaces separados
`pages` garante `unique(wiki_id, slug)` e redirects garantem `unique(wiki_id, from_slug)`, mas o banco não impede:

```text
page: /minecraft
redirect: /minecraft -> /outro
```

A aplicação pode verificar, mas o banco não garante integridade global do namespace.

**Correção:** modelar aliases/redirects e páginas em namespace compartilhado, ou usar estratégia de reservation/trigger cuidadosamente desenhada.

### P1 — `unique(page_id, content_hash, version)` é redundante
Como `unique(page_id, version)` já existe, a combinação tripla não impede nada adicional.

Se a intenção é impedir conteúdo duplicado, `version` não deveria participar — embora proibir duas revisões com conteúdo igual possa conflitar com reversões legítimas. O melhor é tratar `content_hash` como deduplicação informativa, não constraint de negócio.

### P1 — `current_revision_id` precisa de invariant forte
O documento diz que uma página publicada deve possuir revisão atual, mas não mostra a constraint completa nem garante que `current_revision_id` pertence à mesma `page_id`.

**Correção:** considerar FK composta `(page_id, current_revision_id)` para uma revisão da própria página, ou outra modelagem equivalente.

### P1 — `based_on_revision_id` também deveria impedir referência cruzada entre páginas
Hoje uma revisão pode apontar para revisão de outra página se a aplicação errar.

**Correção:** FK composta envolvendo `page_id` quando o banco puder expressar a invariável.

### P2 — Payload de outbox sem limite/contrato
`jsonb payload` pode crescer sem limite operacional.

**Correção:** payload mínimo, schema por `event_type`, limite de bytes e regra de não incluir conteúdo integral/PII.

### P2 — Auditoria "append-only" não é realmente garantida pelo schema
A descrição diz append-only, mas não há mecanismo de banco que impeça UPDATE/DELETE por uma credencial da aplicação.

**Correção:** permissões de banco separadas, trigger ou tabela/role de escrita dedicada se o nível de proteção justificar.

---

# Auditoria individual — 06-editorial-content.md

### P1 — Regras de slug estão mais complexas que o necessário e sem algoritmo fechado
"Unicode normalizado e transliterado quando necessário" pode gerar colisões (`ação`/`acao`, caracteres diferentes com mesma transliteração etc.).

**Correção:** especificar algoritmo, normalização, colisão e fallback determinístico.

### P1 — Redirect e links internos têm duas estratégias parcialmente sobrepostas
O documento diz que links armazenam alvo lógico e que mudança de slug continua funcionando; depois usa redirect para slug antigo. Se o alvo lógico for resolvido no render, o redirect pode ser apenas compatibilidade externa. Isso precisa ser explicitado.

### P1 — Histórico público não define política pós-anonimização/remoção
Uma revisão antiga pode conter PII que a versão atual não contém. Sanitização de segurança não resolve privacidade.

**Correção:** definir se admins podem ver todas as revisões, o que visitantes podem ver após delete/anonymization e quando uma revisão deve ser redigida.

### P2 — `Ctrl/Cmd+S` está semanticamente estranho no MVP
O documento diz que não há drafts persistidos, enquanto UX define Ctrl+S como salvamento local/preview. O comportamento precisa ser descrito como "salvar rascunho local" e não como save da página.

### P2 — Sanitização duplicada exige teste de equivalência
Parse → render → sanitize e depois sanitize na leitura pode ter diferenças de versão. Isso é defensivo, mas deve haver testes para garantir que o conteúdo não muda de maneira inesperada entre publicação e leitura.

---

# Auditoria individual — 07-api-contract.md

### P0 — Idempotência está descrita, mas o mecanismo de persistência não está definido
A API promete repetir `Idempotency-Key` sem duplicar revisão/auditoria, mas nenhum armazenamento de chave + request hash + status + resposta é especificado.

**Correção:** criar contrato de idempotência: chave, ator, endpoint, request hash, status, response body, expiração e comportamento quando payload divergir.

### P1 — `Idempotency-Key` aparece como header, mas exemplos também usam `idempotencyKey` no body
Isso cria duas fontes de verdade.

**Correção:** header somente; não aceitar chave duplicada no JSON.

### P1 — `/pages/:slug?wiki=` não fecha o futuro multi-wiki
O documento diz que `wiki` é opcional. Com múltiplas wikis, ausência de wiki gera ambiguidade ou fallback implícito.

**Correção:** escolher um contexto obrigatório: `/wikis/:wikiSlug/pages/:pageSlug`, host/subdomain por wiki, ou resolver explicitamente uma única wiki pública no MVP.

### P1 — Histórico público por ID pode expor conteúdo antigo de recurso excluído
A rota precisa de policy de visibilidade por página/revisão, não somente por página atual.

### P2 — 401/403/404 não estão totalmente uniformes
Alguns critérios dizem `403`, segurança recomenda `404` para não enumeração. Isso é defensável, mas deve ser uma regra endpoint-by-endpoint no OpenAPI.

### P2 — PUT e PATCH não são diferenciados
`PUT /pages/:id` cria uma nova revisão, mas semanticamente é uma operação de edição parcial/command. Não é um problema funcional, mas uma API de comando explícita (`POST /pages/:id/edits`) poderia reduzir ambiguidades.

---

# Auditoria individual — 08-security.md

### P1 — CSP mínima está incompleta para uma aplicação Next.js real
`default-src 'self'` é um bom início, mas a aplicação real precisa definir `script-src`, `style-src`, `img-src`, `connect-src`, fontes e, se necessário, nonce/hash. Uma CSP simplificada pode quebrar o frontend ou levar a exceções inseguras.

**Correção:** definir CSP baseada nos recursos reais e testar em staging com Report-Only antes de enforcement.

### P1 — OIDC replay/CSRF precisa de armazenamento de estado transitório
State/nonce/PKCE são exigidos, mas não está especificado onde ficam, seu TTL e como são consumidos uma única vez.

**Correção:** state one-time, TTL curto e vinculação à sessão/browser.

### P1 — Rate limit não possui estratégia de fail-open/fail-closed por operação
Se Redis cair, login, editor requests e admin mutations podem ter comportamentos diferentes.

**Correção:** matriz explícita de degradação por endpoint.

### P1 — Recuperação administrativa fora do Google é poderosa demais sem controles operacionais detalhados
Um comando capaz de promover admin é necessário, mas deve exigir acesso ao ambiente, confirmação, audit trail e proteção contra execução automatizada indevida.

### P2 — Não há política explícita de CORS
Isso deveria estar fechado junto com a topologia Next→Nest.

### P2 — Rate limits fixos podem ser contornados por múltiplas contas/IPs
A especificação já usa IP + sessão + usuário, mas abuso distribuído ainda exige bloqueio e moderação operacional. Isso precisa ser assumido como limite, não como prevenção completa.

---

# Auditoria individual — 09-privacy-and-governance.md

### P0 — CC BY-SA 4.0 é uma decisão legal/produto que precisa de processo de contribuição completo
O documento escolhe a licença, mas ainda não fecha atribuição de conteúdo de terceiros, importação, compatibilidade de licenças, copyright takedown, counter-notice e tratamento de material não licenciável.

**Correção:** criar política editorial/legal separada antes do lançamento público.

### P1 — Exportação não define se inclui histórico completo
Para uma wiki, contribuições e revisões podem ser dados relevantes do titular. O formato deve definir exatamente o que é exportado.

### P1 — Anonimização precisa de tombstone ou author snapshot
Mesmo problema do banco: não basta dizer que a autoria vira `Usuário removido` se a FK exige usuário real.

### P1 — Retenção fixa de 180 dias para auditoria pode conflitar com segurança/incidentes
É necessário separar retenção padrão, retenção legal e retenção de investigação.

### P2 — "e-mail para recuperação" é citado como finalidade, mas o sistema é OIDC
Definir se recuperação é realmente necessária e quem fornece esse mecanismo.

---

# Auditoria individual — 10-ux-accessibility-seo.md

### P1 — `Ctrl/Cmd+S` conflita conceitualmente com ausência de drafts persistidos
Precisa ser explicitamente save local, não save editorial.

### P1 — SEO e cache/revalidation não fecham o mecanismo de atualização
O documento diz que sitemap deve ser invalidado em publicação/delete/restore/slug change, mas não define se isso ocorre via outbox, ISR/cache tag, job ou regeneração.

**Correção:** ligar cada invalidação a um evento operacional concreto.

### P2 — Meta WCAG 2.2 AA precisa de definição de "violação crítica"
axe-core não cobre todos os critérios WCAG. O documento corretamente exige teste manual, mas deve definir quais falhas bloqueiam release.

### P2 — Página pública e admin podem compartilhar componentes, mas o design system não define contrato de temas/cores para estados de segurança
Definir tokens para erro, warning, success, blocked, draft e deleted.

---

# Auditoria individual — 11-search-performance-cache.md

### P1 — Chave de cache por `revisionId` pode reduzir o benefício do cache
Para recuperar `page:{wiki}:{slug}:revision:{revision}`, alguém precisa primeiro descobrir a revisão atual. Isso pode exigir uma consulta extra ou uma segunda chave de ponteiro.

**Correção:** documentar cache de resolução `wiki+slug -> currentRevisionId` ou usar tag/invalidation key coerente.

### P1 — Interface `SearchProvider` não define idempotência
Outbox pode reenviar `index()`/`remove()`. O provider precisa aceitar repetição segura.

### P1 — Trigger de 50.000 páginas é arbitrário
Pode ser razoável como heurística, mas p95/CPU/janela de indexação são os sinais mais importantes. O requisito de "sete dias consecutivos" pode atrasar uma migração necessária.

**Correção:** tratar número de páginas como sinal, não gate rígido.

### P2 — Search v1.1 não define stemming, idioma ou ranking
Para uma wiki em português, isso afeta relevância significativamente.

### P2 — FTS não está no MVP, mas não existe busca mínima definida
O produto pode precisar ao menos de navegação por título para descoberta inicial. Se isso ficar fora, a decisão deve ser explícita.

---

# Auditoria individual — 12-observability-sla.md

### P1 — SLO de publicação não é facilmente observável
"Sem perda ou corrupção" é uma propriedade de integridade, não um SLI HTTP simples.

**Correção:** separar SLO de disponibilidade da publicação de invariantes de integridade, que devem ser 100% e testadas/monitoradas por checks.

### P1 — RPO de 24h combina com backup diário, mas não com todos os cenários
Se houver falha logo antes do dump diário, a perda pode chegar perto do limite. Se o produto crescer, WAL/PITR pode ser necessário.

### P1 — Readiness <500ms pode ser incompatível com dependências reais
Uma checagem de banco/Redis pode exceder isso sob cold start sem representar indisponibilidade.

### P2 — Não há SLO para login/OIDC
Autenticação é fluxo crítico do produto.

### P2 — Não há alerta explícito para crescimento de disco/revisões
Há métricas de banco, mas a tabela de alertas deveria incluir armazenamento próximo do limite.

---

# Auditoria individual — 13-devops-backups-and-recovery.md

### P1 — Não há estratégia explícita de PITR/WAL
Para o RPO de 24h, dump diário pode ser suficiente no MVP, mas o documento deve deixar claro que o objetivo é deliberadamente limitado e qual é o caminho de evolução.

### P1 — Chaves de criptografia do backup precisam de estratégia de recuperação independente
O documento menciona armazenamento separado, mas não define escrow, rotação, acesso de emergência e teste de recuperação.

### P1 — Rollback de aplicação não cobre worker/outbox e versão de eventos
Uma versão nova pode ter criado eventos que a versão anterior não entende.

**Correção:** versionar event types/payloads ou garantir compatibilidade backward/forward durante rollback.

### P1 — Restore não define como lidar com Redis
Sessões, rate limits e locks podem ser perdidos. Isso provavelmente é aceitável, mas precisa ser explicitamente tratado como estado efêmero.

### P2 — Deploy gradual é citado sem estratégia concreta
Definir rolling, blue/green ou canary e como migrations expand/contract se comportam em cada caso.

---

# Auditoria individual — 14-testing-quality.md

### P1 — E2E de conflito não especifica dois usuários autenticados distintos
"Segundo editor" precisa ser uma sessão/conta diferente, ou o teste não valida concorrência real entre atores.

### P1 — Não há teste explícito do último-admin invariant
Esse é um dos riscos mais graves do modelo de autorização e deve estar na suíte de domínio.

### P1 — Não há teste explícito de histórico após anonimização
Precisa validar simultaneamente privacidade, integridade editorial e FK.

### P1 — Não há teste de idempotency-key com payload divergente
O contrato exige esse comportamento, mas a matriz de testes não o destaca.

### P2 — Testes de migration deveriam validar schema contra uma versão anterior da aplicação
Banco vazio é necessário, mas não cobre compatibilidade de rollback/rolling deploy.

### P2 — axe-core não é suficiente para WCAG AA
O documento já exige teste manual, mas o gate deve listar critérios críticos manuais.

---

# Auditoria individual — 15-roadmap-acceptance.md

### P1 — M5 concentra muitos riscos operacionais no final
Logs, SLO, backup, restore, E2E, acessibilidade e SEO entram todos em uma milestone. Isso pode criar um "big bang" de qualidade perto do lançamento.

**Correção:** mover skeleton de observabilidade, backup e restore para M0/M1; M5 deve fechar, não começar, essas capacidades.

### P1 — Auditoria de arquitetura está listada como resolvida, mas alguns problemas continuam abertos
A matriz de rastreabilidade afirma que vários problemas foram resolvidos, porém os conflitos encontrados nesta auditoria ainda existem, especialmente anonimização/FK, idempotência, último admin e namespace de redirects.

**Correção:** adicionar uma coluna `status` (`resolved`, `partial`, `open`) e só marcar como resolvido após teste/critério de aceite.

### P2 — Definition of Done exige "métricas" para toda tarefa
Isso pode ser exagerado para tarefas sem impacto observável.

**Correção:** "métricas quando aplicável" e obrigatórias para features com impacto operacional.

### P2 — Não há milestone explícita para threat-model re-review
O threat model deveria ser revisado após identidade, conteúdo e deploy, não apenas criado em M0.

---

# Problemas cruzados entre documentos

## 1. Anonimização × banco × histórico — CRÍTICO

`04`, `05` e `09` não possuem uma única solução consistente para autoria após anonimização.

**Decisão necessária:**

```text
Opção A:
User tombstone permanente
  author_id continua NOT NULL
  nome público = Usuário removido
  PII removida

Opção B:
author_id nullable
  snapshot de autoria não-PII
  histórico preservado sem FK obrigatória
```

A Opção A tende a simplificar integridade histórica.

## 2. Último admin — CRÍTICO

O modelo permite governança por wiki, mas não declara o invariant mínimo de recuperação.

Adicionar:

```text
Uma wiki ativa nunca pode ficar sem pelo menos um admin ativo.
```

Esse invariant precisa existir em use case + teste de concorrência.

## 3. Idempotência — CRÍTICO

API, arquitetura e testes prometem idempotência, mas falta entidade/tabela.

Modelo sugerido:

```text
idempotency_records
- id
- actor_id
- wiki_id
- key
- method
- route
- request_hash
- status
- response_status
- response_body
- created_at
- expires_at
```

Unique por `(actor_id, method, route, key)`.

## 4. Namespace de página/redirect — ALTO

O sistema precisa decidir se redirects são aliases históricos de um mesmo namespace ou uma entidade separada. Atualmente o banco permite estados que o domínio diz serem inválidos.

## 5. Multi-wiki — ALTO

`wiki_id` está bem modelado, mas `/pages/:slug?wiki=` não é uma API que escala elegantemente para multi-wiki.

Sugestão:

```text
/api/v1/wikis/:wikiSlug/pages/:pageSlug
```

ou resolver wiki pelo host/subdomínio.

## 6. Outbox — ALTO

Adicionar explicitamente:

- claim/lease;
- `locked_at`/`locked_by` se necessário;
- backoff;
- jitter;
- limite de tentativas;
- DLQ/estado terminal;
- métricas de idade;
- compatibilidade de payload;
- idempotência por handler.

## 7. Redis — ALTO

Definir claramente quais capacidades dependem dele:

| Capacidade | Se Redis cair |
|---|---|
| Sessão | bloquear login ou usar estratégia alternativa segura |
| Rate limit | limite conservador/fail-closed para admin |
| Cache | bypass para PostgreSQL |
| Locks | não usar para invariantes editoriais |

## 8. Histórico e privacidade — ALTO

Criar uma matriz de visibilidade:

| Estado | Visitante | Usuário | Editor | Admin |
|---|---:|---:|---:|---:|
| Página publicada | ✓ | ✓ | ✓ | ✓ |
| Revisão publicada | ? | ? | ? | ✓ |
| Página deleted | — | — | — | ✓ |
| Revisão de página deleted | — | — | — | ✓ |
| Draft | — | próprio | próprio | ✓ |
| Auditoria | — | — | — | ✓ |

O `?` deve ser resolvido antes da implementação.

## 9. Next.js / NestJS / cache — ALTO

A arquitetura deve definir se o Next renderiza páginas públicas dinamicamente, estaticamente ou por cache/revalidation. Next.js suporta renderização estática/dinâmica e modelos próprios de cache; misturar isso com Redis sem uma estratégia única pode gerar conteúdo desatualizado. citeturn2search1turn2search7

## 10. Stack versionada — MÉDIO

Trocar "15 ou superior" por versão major suportada e definir janela de atualização. Next.js recomenda produção em Active/Maintenance LTS; em agosto de 2026, 16.x está Active LTS e 15.x Maintenance LTS. citeturn1search1

---

# Prioridade de correção

## P0 — resolver antes de escrever o núcleo

1. Anonimização + `author_id`.
2. Último admin / recuperação de governança.
3. Idempotency storage e contrato.
4. Namespace page/redirect.
5. Política de visibilidade de histórico.
6. Topologia Next/API + cookie/CSRF/CORS.

## P1 — resolver antes do MVP estar completo

7. Outbox claiming/leases.
8. Redis failure modes.
9. Multi-wiki no contrato de URL.
10. Constraints de status e invariantes compostas.
11. Compatibilidade de rollback de eventos/outbox.
12. Política de CSP real do frontend.
13. Stack major version pinning.
14. SLOs mensuráveis e alertas de storage.

## P2 — melhorar antes do lançamento público

15. Busca mínima/título no MVP ou decisão explícita de ausência.
16. Ranking/idioma do FTS.
17. Refinamento de UX do Ctrl/Cmd+S.
18. Governança de licença/takedown completa.
19. Threat-model re-review por milestone.
20. Redução do big-bang do M5.

---

# Veredito

**A arquitetura é boa e implementável, mas ainda não está 100% fechada.**

O maior perigo agora não é adicionar mais tecnologia; é começar a implementação enquanto algumas invariantes importantes continuam apenas implícitas.

A recomendação é **não reescrever a arquitetura**. Em vez disso, fechar os P0 e os P1, transformar cada decisão em constraint/policy/teste quando possível e depois iniciar M0/M1.

A arquitetura já está suficientemente estruturada para um monólito modular sério. O próximo salto de qualidade deve ser transformar as regras escritas em **invariantes verificáveis**, especialmente no banco, policies, idempotência, outbox e testes concorrentes.
