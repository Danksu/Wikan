# Wikan — Segurança e Threat Model

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Objetivo de segurança

A Wikan aceita conteúdo produzido por usuários e concede poderes editoriais progressivos. O modelo de segurança deve proteger três propriedades: **confidencialidade** de sessão e dados pessoais, **integridade** de conteúdo e permissões e **disponibilidade** do serviço editorial.

Segurança não pode depender de esconder botões. O backend deve rejeitar toda operação não autorizada, inclusive quando o atacante conhece a rota, altera IDs, envia campos extras ou repete requests em paralelo.

## 2. Ameaças prioritárias

| ID | Ameaça | Impacto |
|---|---|---|
| T-01 | XSS armazenado em Markdown, títulos, categorias, resumos ou mídia | Execução no navegador de leitores/editor; roubo de sessão e vandalismo. |
| T-02 | CSRF em mutations com cookie | Criação, edição, exclusão ou alteração administrativa em nome de vítima. |
| T-03 | Mass assignment e privilege escalation | Usuário comum se promove ou altera estado de outro usuário. |
| T-04 | IDOR | Leitura ou alteração de página, revisão, request ou auditoria fora do escopo. |
| T-05 | Corrida em edição, slug ou aprovação | Perda de conteúdo, memberships duplicadas ou estado inconsistente. |
| T-06 | OAuth mal validado ou lockout do admin | Conta associada de forma errada ou impossibilidade de recuperar operação. |
| T-07 | Spam, vandalismo e SEO poisoning | Conteúdo de baixa qualidade, flood, links abusivos e perda de confiança. |
| T-08 | Upload malicioso | XSS, polyglot, consumo de recursos ou execução no host. |
| T-09 | Vazamento de PII/secrets em logs | Exposição de identidade, tokens, cookies e dados operacionais. |
| T-10 | DoS por busca, payload ou renderização | Indisponibilidade e custo operacional inesperado. |

## 3. Autenticação OIDC

O backend deve validar issuer, audience, assinatura, `exp`, `iat` quando aplicável, `nonce`, `state` e PKCE. O callback só aceita o redirect registrado. O `sub` do provider é persistido como identidade; e-mail não reassocia contas.

Sessões são server-side em Redis ou store equivalente, com token aleatório de alta entropia no cookie. O servidor deve rotacionar a sessão após login, expirar sessão por inatividade e idade máxima configuradas, invalidar no logout e revogar todas as sessões ao bloquear usuário. Tokens não devem ser colocados em URLs, logs ou mensagens de erro.

O bootstrap do primeiro admin ocorre por procedimento operacional e não por endpoint HTTP temporário. A recuperação deve continuar possível se o Google estiver indisponível, por meio de acesso ao ambiente e comando auditado.

## 4. Autorização

Toda mutation passa por policy centralizada com escopo da wiki e recurso. O serviço deve negar por padrão quando não houver policy explícita. O frontend pode ocultar controles, mas isso é apenas UX.

Os DTOs devem ser allowlist. O servidor nunca deve copiar objetos recebidos diretamente para Prisma. Campos protegidos incluem `role`, `status`, `wikiId`, `createdBy`, `updatedBy`, `currentRevisionId`, `deletedAt` e `authorId`.

## 5. XSS e conteúdo

O pipeline de conteúdo é:

```text
limite de tamanho → parse AST → validar nós → validar protocolos
→ render allowlist → sanitizar no backend → armazenar fonte
→ sanitizar novamente na leitura
```

Saída inicial permitida: estrutura de texto, tabela controlada, código sem execução e links HTTP/HTTPS. Proibir scripts, atributos `on*`, CSS inline, iframes, objects, embeds, SVG e protocolos ativos.

Aplicar CSP em produção com, no mínimo, `default-src 'self'`, `object-src 'none'`, `base-uri 'self'` e `frame-ancestors 'none'`. O frontend não deve usar `dangerouslySetInnerHTML` com conteúdo não derivado de resposta sanitizada e testada do backend.

Títulos, nomes de categoria, slugs, resumos e mensagens de auditoria também são entradas não confiáveis. Escapar por contexto em HTML, atributo, URL e texto; não confiar em uma única sanitização global.

## 6. CSRF e headers

Como a sessão usa cookie, toda mutation com browser deve exigir token anti-CSRF e validar `Origin` ou `Referer` conforme a política. Cookies devem ser `HttpOnly`, `Secure` em produção e `SameSite=Lax` ou mais restritivo.

Headers de segurança mínimos:

| Header | Política |
|---|---|
| `Content-Security-Policy` | Sem execução de conteúdo editorial; restringir objetos e frames. |
| `X-Content-Type-Options` | `nosniff`. |
| `Referrer-Policy` | Política restritiva, como `strict-origin-when-cross-origin`. |
| `Permissions-Policy` | Desabilitar APIs não utilizadas. |
| `Strict-Transport-Security` | Ativar em produção após HTTPS estar garantido. |
| `Cache-Control` | Não armazenar responses privadas ou administrativas em CDN compartilhada. |

## 7. IDOR e enumeração

UUIDs não são autorização. Cada handler deve carregar o recurso com `wiki_id`, status e relação necessária e executar policy antes de retornar dados ou modificar estado. Quando um recurso não é visível, responder `404` genérico para reduzir enumeração.

Perfis públicos devem expor somente nome de exibição, avatar opcional, data de entrada e contribuições públicas paginadas. Não expor e-mail, provider subject, IP ou requests administrativos.

## 8. Rate limiting e anti-abuso

O rate limiter usa Redis e diferencia IP, sessão e usuário quando disponível. Respostas excedentes são `429` com `Retry-After`; o sistema não deve bloquear uma rede inteira por uma única conta sem mecanismo de investigação.

| Operação | Limite inicial |
|---|---:|
| Início/callback OIDC | 10/min por IP |
| Pesquisa pública | 60/min por IP |
| Solicitação de editor | 3/24h por usuário e 10/dia por IP |
| Criação de página | 10/h por editor |
| Edição de página | 60/h por editor |
| Reversão | 30/h por editor; admin configurável |
| Admin mutations | 30/min por admin |
| Upload v1.1 | 20/h por usuário; 5 MiB/arquivo |

Anti-abuso adicional no MVP: auditoria, possibilidade de bloquear usuário, reversão e limites de tamanho. Na v1.1: revisão das primeiras edições, trust score simples, proteção de páginas e detecção de padrões.

## 9. Upload seguro — v1.1

Upload não é dependência do MVP. Quando implementado, o pipeline obrigatório será:

1. aceitar somente JPEG, PNG, WebP ou GIF;
2. limitar bytes e dimensões antes do processamento;
3. validar assinatura real do arquivo, não extensão/MIME informado;
4. processar em área de quarentena com biblioteca atualizada;
5. re-encodar e remover metadata desnecessária;
6. gerar nome aleatório, nunca usar nome original como caminho;
7. armazenar em bucket privado fora da webroot;
8. servir com `Content-Type` correto e `X-Content-Type-Options: nosniff`;
9. gerar thumbnails com limites de memória;
10. aplicar rate limit e registrar auditoria sem conteúdo sensível.

SVG deve ser rejeitado no primeiro release de mídia. URLs remotas de imagem não devem ser buscadas pelo servidor sem política anti-SSRF.

## 10. SQL injection e integridade de entrada

Usar Prisma parametrizado ou queries preparadas. Queries raw exigem revisão específica, parâmetros separados e teste. Validar limites de tamanho, encoding, profundidade JSON, número de itens e filtros antes de consultar.

O servidor não deve aceitar filtros arbitrários, nomes de coluna ou trechos de ordenação vindos diretamente do cliente. Mapear campos permitidos para expressões internas.

## 11. Segredos e logs

Nunca registrar senhas, client secret, `SESSION_SECRET`, cookies, tokens OIDC, Authorization header, conteúdo integral de página ou payload administrativo completo. IP deve ser minimizado/pseudonimizado conforme política de privacidade.

Falhas de segurança devem possuir `request_id`, actor pseudonimizado, rota normalizada, ação e resultado. Logs de auditoria são diferentes de logs técnicos e têm controle de acesso próprio.

## 12. Threat scenarios

| Cenário | Resposta esperada |
|---|---|
| Usuário envia `role=admin` em `PUT /me` | DTO rejeita campo; role permanece igual; evento de segurança pode ser registrado. |
| Usuário troca `pageId` por UUID de outra wiki | Policy nega e não revela conteúdo. |
| Dois admins aprovam request ao mesmo tempo | Uma transação vence; uma membership; resultado idempotente. |
| Editor publica dezenas de páginas | Rate limit; auditoria; admin pode bloquear e reverter. |
| Conteúdo contém `<img onerror>` | Sanitizer remove atributo e teste confirma ausência. |
| Cookie é enviado por site externo | CSRF token/origin check retorna `403`. |
| Google callback é repetido | State/nonce/PKCE e idempotência impedem criação duplicada. |
| Admin perde acesso ao Google | Procedimento operacional de recuperação, sem endpoint público temporário. |

## 13. Testes obrigatórios de segurança

O CI deve executar testes de payload XSS em conteúdo e metadata, mass assignment em todas as mutations de usuário e página, IDOR por wiki e recurso, CSRF sem token, session fixation, replay de callback, concorrência, rate limiting e redaction de logs.

Antes do lançamento público, executar revisão manual de threat model e teste de penetração direcionado ao fluxo editorial. Vulnerabilidade crítica bloqueia release; alta exige plano, responsável e prazo aprovado.

## 14. Critérios de aceite

| ID | Critério |
|---|---|
| SEC-01 | Nenhum teste XSS consegue executar ou persistir protocolo/permissão perigosa. |
| SEC-02 | Mutation cross-site sem token/origin válido retorna `403`. |
| SEC-03 | Nenhum endpoint permite mass assignment de role, estado ou escopo. |
| SEC-04 | Trocar IDs não contorna policy nem enumera recurso privado. |
| SEC-05 | Logout, bloqueio e revogação invalidam sessões existentes. |
| SEC-06 | Limites críticos retornam `429` e não geram efeitos parciais. |
| SEC-07 | Logs de teste não contêm secrets, cookies ou tokens. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[3]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
