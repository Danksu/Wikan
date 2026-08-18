# Wikan — Especificação Modular

**Versão:** 3.0
**Status:** índice normativo e guia de navegação
**Última revisão:** 18 de agosto de 2026
**Autor:** Manus AI

> Este arquivo é o ponto de entrada da especificação da Wikan. Os requisitos detalhados estão separados por tema em `docs/spec/`. Uma decisão normativa deve ser atualizada no documento temático responsável e refletida neste índice quando alterar escopo, dependências ou critérios de aceite.

## 1. Como ler este spec

A Wikan não deve ser implementada seguindo um documento monolítico. Primeiro leia o escopo e a stack; depois arquitetura, domínio, banco e segurança; em seguida os contratos editoriais e de API; por fim operação, qualidade e roadmap.

| Ordem | Documento | Responsabilidade |
|---:|---|---|
| 1 | [Produto e escopo](docs/spec/01-product-and-scope.md) | Objetivo, usuários, MVP, v1.1, não-objetivos e definição de pronto. |
| 2 | [Stack](docs/spec/02-stack.md) | Runtime, frameworks, banco, cache, ambientes e padrões de código. |
| 3 | [Arquitetura](docs/spec/03-architecture.md) | Monólito modular, módulos, fronteiras, transações e outbox. |
| 4 | [Domínio e autorização](docs/spec/04-domain-and-authorization.md) | Usuários, identities, memberships, policies e governança. |
| 5 | [Banco de dados](docs/spec/05-database.md) | Entidades, constraints, foreign keys, índices, migrations e retenção. |
| 6 | [Conteúdo editorial](docs/spec/06-editorial-content.md) | Markdown, sanitizer, páginas, revisões, conflitos, slugs e redirects. |
| 7 | [API REST](docs/spec/07-api-contract.md) | Rotas, contratos, erros, paginação, idempotência e versionamento. |
| 8 | [Segurança](docs/spec/08-security.md) | Threat model, XSS, CSRF, IDOR, mass assignment, abuso e sessões. |
| 9 | [Privacidade e governança](docs/spec/09-privacy-and-governance.md) | PII, anonimização, licença editorial, auditoria e retenção. |
| 10 | [UX, acessibilidade e SEO](docs/spec/10-ux-accessibility-seo.md) | Fluxos, estados, editor acessível, responsividade e indexação. |
| 11 | [Busca, performance e cache](docs/spec/11-search-performance-cache.md) | FTS, paginação, metas p95, cache e gatilhos de escala. |
| 12 | [Observabilidade e SLA](docs/spec/12-observability-sla.md) | SLI/SLO, error budget, incidentes, RPO e RTO. |
| 13 | [DevOps, backup e recuperação](docs/spec/13-devops-backups-and-recovery.md) | CI/CD, deploy, migrations, backup, restore e rollback. |
| 14 | [Testes e qualidade](docs/spec/14-testing-quality.md) | Pirâmide de testes, segurança negativa, concorrência, E2E e gates. |
| 15 | [Roadmap e aceite](docs/spec/15-roadmap-acceptance.md) | Milestones, dependências, critérios AC-01 a AC-18 e rastreabilidade. |
| 16 | [ADRs](docs/spec/adr/README.md) | Processo para registrar decisões técnicas e mudanças de escopo. |

## 2. Decisões centrais

| Tema | Decisão |
|---|---|
| Arquitetura | Monólito modular em monorepo; Next.js para web, NestJS para API, PostgreSQL para domínio. |
| API | REST versionada em `/api/v1`; frontend não acessa banco diretamente. |
| Identidade | Google OIDC no MVP; vínculo por `(provider, provider_sub)`; e-mail não é chave. |
| Sessão | Server-side, revogável e protegida por cookie seguro. |
| Primeiro admin | Bootstrap operacional por comando; sem endpoint público de emergência. |
| Autorização | Policies centralizadas no backend, com escopo por wiki e recurso. |
| Multi-wiki | `Wiki` existe desde o dia 1; UI de múltiplas wikis é backlog. |
| Revisões | Snapshot completo do Markdown por revisão; histórico não é apagado. |
| Conflitos | `expectedRevisionId`, compare-and-swap e `409`; nunca overwrite silencioso. |
| Conteúdo | Markdown restrito, HTML livre desabilitado, links internos explícitos. |
| Slugs | Únicos por wiki; mudança cria redirect 301 e canonical correto. |
| Busca | PostgreSQL FTS na v1.1 atrás de `SearchProvider`; motor externo depende de gatilho. |
| Segurança | Threat model específico de wiki, testes negativos e controles no backend. |
| Licença | Conteúdo publicado em CC BY-SA 4.0; código mantém licença do repositório. |
| Operação | SLO interno, RPO 24h, RTO banco 4h, backup restaurável e CI obrigatório. |

## 3. Escopo resumido

O MVP inclui leitura pública, login Google, conta interna, bootstrap admin, `user/editor/admin`, request e aprovação de editor, páginas, Markdown restrito, revisões, diff, reversão, soft delete, restauração, auditoria, autorização, rate limiting, CI, backup e E2E.

A v1.1 inclui categorias, drafts persistidos, mídia, FTS, notificações, follows e moderação preventiva. Discussões, reputação, badges, múltiplas wikis na UI, colaboração em tempo real, aplicativo móvel, microserviços e Kubernetes não bloqueiam o MVP.

## 4. Ordem de implementação

```text
M0 Fundação e ADRs
  ↓
M1 Identidade e sessão
  ↓
M2 Memberships, policies e requests
  ↓
M3 Páginas, Markdown, revisões e conflitos
  ↓
M4 Moderação, reversão, soft delete e auditoria
  ↓
M5 Observabilidade, backup, restore, E2E, acessibilidade e SEO
  ↓
M6 v1.1 condicionada a métricas e estabilidade
```

Não iniciar M6 enquanto os critérios do MVP estiverem falhando. O [roadmap detalhado e a matriz de aceite](docs/spec/15-roadmap-acceptance.md) são a referência para dependências e release.

## 5. Critério de conclusão do MVP

O MVP só está concluído quando o fluxo abaixo passar em staging:

```text
visitante lê
  → login Google
  → usuário interno criado
  → request de editor
  → admin aprova
  → editor cria página
  → página é publicada
  → editor altera
  → revisão e diff são registrados
  → admin reverte sem apagar histórico
```

Além do fluxo feliz, devem passar os testes de autorização negativa, XSS, CSRF, IDOR, mass assignment, concorrência, acessibilidade, backup e restauração definidos nos documentos temáticos.

## 6. Governança da especificação

O spec temático é a fonte de verdade do assunto que nomeia. Mudanças de stack, banco, identidade, licença, SLO, RPO/RTO, roles, histórico ou escopo exigem ADR em [`docs/spec/adr/`](docs/spec/adr/README.md), atualização dos arquivos afetados e revisão dos critérios de aceite.

A linguagem normativa é deliberada: **DEVE** indica requisito obrigatório, **NÃO DEVE** indica proibição e **PODE** indica extensão não obrigatória. Formulações abertas em pontos críticos não substituem decisão técnica.

## 7. Rastreabilidade das auditorias

A estrutura modular responde às auditorias dos Issues [#1][1], [#2][2], [#3][3] e [#4][4]. O [roadmap](docs/spec/15-roadmap-acceptance.md) contém a matriz completa que liga cada recomendação ao documento responsável.

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[3]: https://github.com/Danksu/Wikan/issues/3 "Issue #3 — Plano de mitigação e implementação"

[4]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
