# Wikan — Testes e Qualidade

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Estratégia

A qualidade da Wikan deve ser verificada em camadas. Testes unitários protegem invariantes; integração verifica banco, Redis, sessões e transações; E2E verifica o fluxo que o usuário realmente executa. Nenhuma camada substitui as demais.

O projeto deve testar casos negativos deliberadamente. Em uma plataforma com roles, é tão importante provar que um editor pode publicar quanto provar que um usuário comum não consegue publicar chamando a API diretamente.

## 2. Pirâmide de testes

| Camada | Escopo | Frequência |
|---|---|---|
| Unidade | Policies, parser, sanitizer, slug, transições, hashes e casos de uso puros | Todo PR |
| Integração | Prisma, constraints, transações, sessão, Redis, outbox e controllers | Todo PR |
| Segurança | XSS, CSRF, IDOR, mass assignment, OIDC replay, rate limit | Todo PR; suíte expandida antes de release |
| Concorrência | Edição, slug, aprovação, bloqueio e idempotência | Todo PR de domínio; carga antes do release |
| E2E | Fluxo visitante → login → request → aprovação → edição → reversão | Main/staging e release |
| Acessibilidade | axe-core, teclado, foco, leitor de tela e zoom | Todo PR de UI; manual antes de release |
| Performance | p95 de leitura, busca, histórico, publicação e admin | Staging e release maior |
| Operacional | migrations, backup, restore, health e rollback | Staging e mensal |

## 3. Matriz de autorização

| Ator | GET página | POST page | PUT page | Request editor | Approve | Admin users |
|---|---:|---:|---:|---:|---:|---:|
| Visitante | 200 | 401/403 | 401/403 | 401 | 401/403 | 401/403 |
| Usuário | 200 | 403 | 403 | 201 | 403 | 403 |
| Editor | 200 | 201 | 200 | 200/409 | 403 | 403 |
| Admin | 200 | 201 | 200 | 200 | 200 | 200 |

A matriz deve ser executada com resources da própria wiki e de outra wiki. O status exato pode ser `401`, `403` ou `404` conforme a política de não enumeração, mas nunca pode haver efeito não autorizado.

## 4. Testes de domínio

Testar transições:

- request `pending → approved` cria membership;
- request `pending → rejected` exige motivo;
- request aprovado não é aprovado duas vezes com efeitos duplicados;
- membership revogada não autoriza edição;
- usuário bloqueado não opera com sessão anterior;
- página publicada sempre aponta para revisão;
- reversão cria revisão nova;
- slug alterado cria redirect;
- soft delete não apaga histórico;
- anonimização remove PII e preserva autoria não identificável.

## 5. Testes de banco

Migrations devem ser testadas em banco vazio e em banco com fixture. Testar unique slug, unique provider subject, request pendente parcial, unique membership, foreign keys restrict/set null, índices principais e constraints de status.

Testar rollback ou procedimento de recuperação para migration que altera campo usado por versão anterior. Não confiar somente em mocks de Prisma para regras que dependem de constraint ou isolation level.

## 6. Testes de concorrência

Casos obrigatórios:

```text
A e B leem revision v1
A publica → v2
B publica com expected v1 → 409
```

```text
Admin A e Admin B aprovam request pending
→ uma membership ativa
→ uma transição efetiva
→ auditoria coerente
```

```text
Editor A e Editor B sugerem o mesmo slug
→ uma operação vence
→ outra recebe PAGE_SLUG_CONFLICT
```

Executar requests em paralelo real contra banco de teste, não apenas chamar função duas vezes em sequência.

## 7. Testes de segurança

### 7.1 XSS

Payloads devem cobrir script, atributos de evento, links `javascript:`, `data:`, SVG, iframes, CSS, HTML em títulos e Markdown. Confirmar que o conteúdo permitido continua legível.

### 7.2 CSRF e sessão

Testar mutation sem token, origin inválida, cookie `SameSite`, rotação de sessão, logout, expiração e revogação após bloqueio. Repetição de callback OIDC deve ser rejeitada ou idempotente sem criar identidades duplicadas.

### 7.3 IDOR e mass assignment

Para cada endpoint com ID, tentar trocar por recurso de outra wiki, revisão não relacionada, request de terceiro e usuário administrativo. Enviar campos protegidos em JSON e confirmar que não mudam no banco.

### 7.4 Rate limit

Ultrapassar cada bucket e verificar `429`, `Retry-After`, ausência de efeitos parciais e recuperação após janela. Testar por IP, sessão e usuário para evitar bypass simples.

## 8. Testes E2E críticos

O cenário principal:

1. visitante abre página publicada;
2. inicia login Google mockado;
3. sistema cria usuário e sessão;
4. usuário envia request;
5. admin abre fila e aprova;
6. usuário consulta membership e vira editor;
7. editor cria página;
8. visitante lê a nova página;
9. editor publica alteração;
10. segundo editor causa conflito;
11. admin consulta histórico;
12. admin reverte uma revisão;
13. página continua acessível e histórico completo.

Adicionar cenários de página inexistente, slug redirect, soft delete/restauração, bloqueio e logout.

## 9. Acessibilidade e UI

Automatizar axe-core em páginas públicas e admin. Testar teclado do início ao fim, foco após dialogs, mensagens `aria-live`, texto aumentado, zoom de 200%, contraste e navegação por headings. Fazer teste manual com leitor de tela para login, leitura, request, editor e aprovação.

Uma violação crítica ou bloqueadora de fluxo impede release. Violações menores devem possuir issue, responsável e prazo.

## 10. Performance e resiliência

Staging deve gerar dataset sintético de páginas, revisões, requests e usuários. Medir p50/p95/p99, 5xx, 429, conflitos e saturação de banco/Redis. Testar queda de Redis, atraso de outbox, busca indisponível e timeout de OIDC sem confirmar publicação falsa.

## 11. Gates de qualidade

| Gate | Bloqueia merge/release? |
|---|---:|
| Lint/typecheck falhou | Sim |
| Teste unitário ou integração falhou | Sim |
| Teste de autorização negativo falhou | Sim |
| Build falhou | Sim |
| Migration em banco vazio falhou | Sim |
| Secret scan encontrou valor | Sim |
| Teste E2E crítico falhou | Release sim; PR conforme escopo |
| axe-core crítico | Sim |
| p95 fora da meta sem justificativa | Release sim |
| Teste de backup/restore falhou | Release sim |

## 12. Fixtures e dados de teste

Fixtures devem usar nomes, e-mails e IDs sintéticos. Não copiar banco de produção para desenvolvimento sem anonimização formal. Factories devem criar wiki, users com cada role, identities, memberships, pages, revisions, requests e audit logs suficientes para testar isolamento.

## 13. Critérios de aceite

| ID | Critério |
|---|---|
| TEST-01 | CI executa unidade, integração, segurança, build e migrations. |
| TEST-02 | Matriz de autorização possui casos positivos e negativos. |
| TEST-03 | Concorrência é testada em paralelo contra banco real de teste. |
| TEST-04 | E2E cobre o fluxo editorial completo. |
| TEST-05 | axe-core e teste manual cobrem os fluxos críticos. |
| TEST-06 | Restore, rollback e modo degradado possuem teste operacional. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[3]: https://github.com/Danksu/Wikan/issues/3 "Issue #3 — Plano de mitigação e implementação"
