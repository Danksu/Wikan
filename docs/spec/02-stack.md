# Wikan — Stack e Padrões Tecnológicos

**Status:** decisão técnica normativa para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Objetivo

Este documento fecha as escolhas tecnológicas que antes estavam deixadas para “a stack escolhida”. A implementação deve seguir estas decisões. Uma substituição exige ADR com comparação de custo, segurança, compatibilidade, migração e impacto no cronograma.

## 2. Stack aprovada

| Camada | Escolha | Baseline |
|---|---|---|
| Runtime | Node.js | Linha LTS atual compatível com o deploy; baseline de desenvolvimento Node 22 LTS. |
| Linguagem | TypeScript | `strict: true`, sem `any` implícito em código de domínio. |
| Frontend | Next.js App Router + React | Next.js 15 ou superior compatível com o runtime escolhido. |
| Backend | NestJS | NestJS 11 ou superior. |
| API | REST/JSON | `/api/v1`, OpenAPI publicado a partir dos contratos. |
| Banco | PostgreSQL | PostgreSQL 16 ou superior no MVP. |
| ORM | Prisma | Versão fixada pelo lockfile e atualizada por ADR quando houver breaking change. |
| Cache/infra efêmera | Redis | Redis 7 ou superior para sessões, rate limit e locks curtos. |
| Storage | S3-compatível | Obrigatório apenas na v1.1 para mídia; bucket privado e URLs assinadas. |
| Busca | PostgreSQL FTS | Atrás da interface `SearchProvider`; motor externo é evolução condicionada a métricas. |
| Testes | Vitest ou Jest no backend; Playwright em E2E | Uma única escolha por camada, fixada no bootstrap do projeto. |
| Qualidade | ESLint, Prettier, typecheck, dependency audit | Todos executados no CI. |
| Empacotamento | Docker e Docker Compose | Imagens imutáveis por digest em produção. |
| CI/CD | GitHub Actions | Pull request, staging e produção separados. |

## 3. Decisões de aplicação

### 3.1 Monorepo modular

O repositório deve conter `apps/web`, `apps/api` e `packages/contracts`. O frontend pode importar tipos e schemas compartilhados, mas não deve importar módulos de infraestrutura do backend. O backend deve ser capaz de rodar e ser testado sem iniciar o navegador.

### 3.2 Backend como autoridade

O backend NestJS é responsável por autenticação, sessões, autorização, parsing de entrada, regras de negócio, migrations, acesso ao banco, outbox e auditoria. O Next.js chama a API e renderiza respostas; ele não deve fazer queries diretas no PostgreSQL nem replicar regras administrativas.

### 3.3 TypeScript estrito

O projeto deve habilitar `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` quando a compatibilidade permitir, `noImplicitOverride` e regras de lint para promises não aguardadas. Conversões `as` em código de domínio devem ser justificadas; dados externos entram por schema validator antes de receber tipo interno.

### 3.4 Contratos de entrada e saída

DTOs REST devem ser schemas explícitos. A aplicação deve rejeitar campos desconhecidos em mutations sensíveis, especialmente `role`, `wikiId`, `createdBy`, `status`, `deletedAt` e `currentRevisionId`. Respostas públicas não devem vazar e-mail, IDs de provider, IP, metadata administrativa ou conteúdo de draft.

### 3.5 Dependências

Cada dependência de runtime deve ter justificativa e licença compatível. O lockfile deve ser versionado. Dependências transitivas críticas devem ser auditadas no CI e atualizadas em janela planejada. Não adicionar uma biblioteca para resolver uma função simples já coberta pela plataforma sem avaliar tamanho, manutenção e superfície de ataque.

## 4. Bibliotecas e capacidades esperadas

A escolha exata de pacote pode variar dentro das decisões abaixo, mas o comportamento não pode variar:

| Necessidade | Capacidade obrigatória |
|---|---|
| OIDC | Biblioteca que valide issuer, audience, nonce, state, PKCE, assinatura e expiração. |
| Sessão | Store server-side revogável; não usar JWT stateless como única sessão administrativa do MVP. |
| Validação | Schema validation no limite da API e nas mensagens de outbox. |
| Markdown | Parser que produza AST/HTML controlado, sem permitir HTML arbitrário. |
| Sanitização | Sanitizer server-side com allowlist de tags, atributos e protocolos. |
| Hash | SHA-256 ou equivalente criptográfico para `content_hash` e idempotência. |
| IDs | UUID ou ULID gerados no backend; nunca IDs sequenciais expostos como autorização. |
| Logs | Logger estruturado com redaction de cookies, tokens e secrets. |
| HTTP | Cliente com timeout, retry somente para operações idempotentes e tracing de request. |
| Storage | Cliente S3 com bucket privado, políticas mínimas e URLs assinadas de curta duração. |

## 5. Ambientes

| Ambiente | Banco | Dados | Acesso |
|---|---|---|---|
| Local | PostgreSQL/Redis via Compose | Dados descartáveis e seed sintético | Desenvolvedores; secrets locais fora do Git. |
| CI | Serviços efêmeros | Fixtures isoladas por job | Sem dados reais; OAuth mockado. |
| Staging | Infra próxima à produção | Dados sintéticos ou anonimizado | Testes E2E, restauração e validação operacional. |
| Produção | Serviços gerenciados ou containers aprovados | PII e conteúdo real | Acesso mínimo, auditado e com aprovação. |

As configurações devem ser validadas no startup. A aplicação deve falhar cedo se faltar `DATABASE_URL`, segredo de sessão, configuração OIDC ou chave de criptografia. Não é permitido iniciar em produção com valores default conhecidos.

## 6. Configuração mínima

```text
NODE_ENV
APP_BASE_URL
API_BASE_URL
DATABASE_URL
REDIS_URL
SESSION_SECRET
OIDC_GOOGLE_CLIENT_ID
OIDC_GOOGLE_CLIENT_SECRET
OIDC_GOOGLE_ISSUER
WIKAN_BOOTSTRAP_GOOGLE_SUB   # somente bootstrap inicial; opcional depois
LOG_LEVEL
SENTRY_DSN                   # opcional no local, recomendado em staging/prod
STORAGE_ENDPOINT              # v1.1
STORAGE_BUCKET                # v1.1
STORAGE_ACCESS_KEY             # v1.1
STORAGE_SECRET_KEY             # v1.1
```

Secrets devem vir de secret manager ou variáveis protegidas no CI. O arquivo `.env.example` contém apenas nomes e valores fictícios. Logs e diagnósticos nunca podem imprimir valores completos dessas variáveis.

## 7. Padrões de código

O backend deve ser organizado por módulo de domínio, não por uma pasta global de controllers gigantes. Cada módulo expõe casos de uso, policies, portas de infraestrutura e adaptadores. Queries Prisma ficam em repositórios ou serviços de persistência; controllers não contêm transações complexas nem regras editoriais.

O frontend deve separar páginas públicas, componentes de conteúdo e componentes administrativos. Componentes de conteúdo não podem renderizar HTML arbitrário sem passar pela resposta sanitizada do backend. Formulários devem exibir estados de validação, loading, erro, sucesso e conflito.

## 8. Observabilidade técnica da stack

Cada requisição deve receber `request_id`. O backend deve registrar duração, rota normalizada, status, resultado da autorização e falha de dependência. O cliente HTTP deve aplicar timeout configurável e propagação de request ID. Workers de outbox devem ser idempotentes, limitar tentativas e enviar eventos persistentemente falhos para uma fila de dead letter ou estado equivalente.

## 9. Processo para trocar a stack

Uma proposta de mudança deve conter:

1. problema que a stack atual não resolve;
2. alternativas avaliadas;
3. impacto em segurança e licenças;
4. impacto em schema, API e operação;
5. plano de migração reversível;
6. benchmark ou evidência;
7. impacto nos testes e no treinamento de manutenção.

A mudança só pode entrar depois de aprovada em ADR e acompanhada de uma atualização deste arquivo e do índice principal.

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[3]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
