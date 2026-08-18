# Wikan — Roadmap e Critérios de Aceite

**Status:** normativo para planejamento do MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Ordem de implementação

A implementação deve seguir dependências de risco, não somente ordem visual de telas. Segurança, schema e bootstrap vêm antes de funcionalidades editoriais; observabilidade e restore devem existir antes do lançamento.

## 2. Milestones

| Milestone | Objetivo | Saída mínima |
|---|---|---|
| M0 — Fundação | Repositório, stack, CI, configuração, Docker, ADRs e threat model | Monorepo inicia localmente, CI verde e schema inicial revisado. |
| M1 — Identidade | OIDC, users, identities, sessão e bootstrap admin | Login, logout, revogação e primeiro admin funcionam em staging. |
| M2 — Autorização | Memberships, policies, requests e bloqueio | Matriz de autorização e aprovação concorrente passam. |
| M3 — Núcleo editorial | Pages, Markdown, sanitizer, slugs, revisões e histórico | Editor cria, edita, consulta diff e recebe conflito corretamente. |
| M4 — Moderação | Reversão, soft delete, restauração e auditoria | Admin consegue investigar e reparar conteúdo sem apagar histórico. |
| M5 — Operação | Logs, SLO, backup, restore, E2E, acessibilidade e SEO | Release candidate passa gates de qualidade e recuperação. |
| M6 — v1.1 | Categorias, drafts, mídia, FTS, notificações e prevenção | Iniciar somente após métricas e estabilidade do MVP. |

## 3. Dependências de entrega

| Dependência | Bloqueia |
|---|---|
| Schema e migrations | Todos os módulos persistentes. |
| Identidade e sessão | Requests, memberships, conteúdo e admin. |
| Policies | Toda mutation e dados administrativos. |
| Sanitizer | Publicação e leitura de conteúdo. |
| Revisões/conflitos | Edição, reversão, moderação e auditoria editorial. |
| Auditoria/outbox | Publicação, aprovação, bloqueio, reversão e side effects. |
| Backup/restore | Release público. |

## 4. Critérios de aceite end-to-end

| ID | Critério observável |
|---|---|
| AC-01 | Visitante lê página publicada sem sessão e não vê draft/excluída. |
| AC-02 | Login Google válido cria uma conta interna por `provider_sub`. |
| AC-03 | Logout e bloqueio invalidam a sessão no servidor. |
| AC-04 | Primeiro admin é criado pelo comando operacional documentado. |
| AC-05 | Usuário envia uma request pendente e recebe estado claro. |
| AC-06 | Admin aprova/rejeita/revoga com transação e auditoria. |
| AC-07 | Usuário não publica mesmo chamando API diretamente. |
| AC-08 | Editor cria página com slug único e revisão v1. |
| AC-09 | Conteúdo malicioso não executa nem passa pela allowlist. |
| AC-10 | Edição com revisão desatualizada retorna `409` e preserva texto local. |
| AC-11 | Histórico registra autor, versão, hash, resumo e timestamp. |
| AC-12 | Diff compara duas revisões e pagina histórico. |
| AC-13 | Reversão cria revisão nova; nenhuma revisão anterior é apagada. |
| AC-14 | Soft delete retira página da leitura e restore mantém histórico. |
| AC-15 | Mudança de slug cria 301 sem cadeia/ciclo. |
| AC-16 | Rate limit, CSRF, IDOR e mass assignment passam testes negativos. |
| AC-17 | E2E editorial completo passa em staging. |
| AC-18 | Backup é restaurado e metas RPO/RTO têm evidência. |

## 5. Definition of Ready

Uma tarefa está pronta para desenvolvimento quando possui objetivo, escopo, ator, comportamento de sucesso e erro, policy, modelo de dados impactado, API/UI afetada, teste planejado e critério de aceite. Se depender de decisão aberta, deve referenciar ADR ou RFC.

## 6. Definition of Done

Uma tarefa está concluída quando código, migration, testes, documentação, métricas e segurança correspondentes estão integrados. A feature não deve depender de alteração manual em produção nem deixar estados sem tratamento.

## 7. Backlog de v1.1

A v1.1 deve ser priorizada por evidência do MVP:

| Recurso | Pré-condição |
|---|---|
| Categorias | Queries e escopo de wiki definidos; árvore sem ciclos. |
| Drafts | Política de privacidade e cleanup definidas; drafts fora de busca/sitemap. |
| Mídia | Pipeline de re-encode, storage privado e licença editorial. |
| FTS | Dataset e benchmark justificam busca indexada. |
| Notificações | Evento, canal, preferências e retenção definidos. |
| Follows | Índices e rate limit para evitar fan-out descontrolado. |
| Pending changes | Trust model e UX de revisão documentados. |

## 8. Matriz de rastreabilidade das auditorias

| Problema apontado | Documento que resolve |
|---|---|
| Escopo amplo sem MVP | `01-product-and-scope.md` e este arquivo. [1][2] |
| Stack e API indefinidas | `02-stack.md`, `03-architecture.md` e `07-api-contract.md`. [1][4] |
| Multi-wiki futura sem entidade raiz | `04-domain-and-authorization.md` e `05-database.md`. [1][2] |
| Schema sem constraints/índices | `05-database.md`. [1][4] |
| Bootstrap do primeiro admin | `04-domain-and-authorization.md` e `13-devops-backups-and-recovery.md`. [2][4] |
| XSS/upload/CSRF genéricos | `06-editorial-content.md` e `08-security.md`. [1][3][4] |
| IDOR/mass assignment | `04-domain-and-authorization.md`, `07-api-contract.md` e `08-security.md`. [2][4] |
| Concorrência vaga | `03-architecture.md`, `05-database.md`, `06-editorial-content.md` e `14-testing-quality.md`. [1][2][4] |
| Redirects ausentes | `05-database.md` e `06-editorial-content.md`. [4] |
| Busca/cache sem gatilho | `11-search-performance-cache.md`. [1][2][4] |
| Privacidade/licença ausentes | `09-privacy-and-governance.md`. [2] |
| SLA/backups/DevOps insuficientes | `12-observability-sla.md` e `13-devops-backups-and-recovery.md`. [1][3] |
| Acessibilidade e UX não mensuráveis | `10-ux-accessibility-seo.md` e `14-testing-quality.md`. [1][4] |

## 9. Regras de mudança

Alterar a ordem do MVP, introduzir uma dependência externa, mudar a licença, remover histórico, trocar identidade ou alterar um SLO exige atualização dos documentos relacionados, ADR e revisão dos critérios de aceite. Não manter uma decisão somente em issue ou conversa informal.

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[3]: https://github.com/Danksu/Wikan/issues/3 "Issue #3 — Plano de mitigação e implementação"

[4]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
