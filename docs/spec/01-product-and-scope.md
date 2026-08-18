# Wikan — Produto e Escopo

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Propósito

A Wikan é uma plataforma de wiki colaborativa para comunidades criarem, revisarem e consultarem conhecimento. O produto deve oferecer leitura pública rápida, um caminho controlado para novos contribuidores obterem permissão editorial e um histórico confiável de todas as alterações relevantes.

O produto não deve ser tratado como um CRUD de páginas nem como uma cópia visual da Fandom. O núcleo de valor é a combinação de **conteúdo estruturado**, **autoria rastreável**, **reversibilidade**, **governança comunitária** e **experiência de edição acessível**.

## 2. Problema que o produto resolve

Comunidades frequentemente armazenam conhecimento em documentos dispersos, posts efêmeros ou plataformas cuja moderação, histórico e identidade visual não controlam. A Wikan oferece um espaço próprio em que:

1. qualquer visitante pode consultar conteúdo publicado;
2. uma pessoa autenticada pode construir confiança antes de receber poderes editoriais;
3. editores podem publicar sem perder o histórico;
4. administradores podem auditar, moderar, restaurar e reverter;
5. a arquitetura permite evolução sem exigir reescrita do núcleo editorial.

## 3. Princípios de produto

| Princípio | Aplicação obrigatória |
|---|---|
| Integridade antes de velocidade | Nenhuma publicação pode deixar página sem revisão, apagar histórico ou sobrescrever edição concorrente silenciosamente. |
| Segurança no servidor | O backend é a autoridade para identidade, autorização, validação e regras de negócio. |
| Leitura em primeiro lugar | Páginas públicas devem privilegiar legibilidade, carregamento previsível e navegação clara. |
| Contribuição gradual | O usuário começa com acesso de leitura e solicita permissão editorial; a comunidade controla a promoção. |
| Transparência | Revisões, autoria, resumo de alteração e ações administrativas devem ser rastreáveis. |
| Complexidade proporcional | Só introduzir cache, workers ou serviços externos quando houver requisito, métrica ou risco que os justifique. |
| Acessibilidade por padrão | Acessibilidade é requisito de aceite, não uma revisão visual posterior. |

## 4. Usuários e tarefas principais

### 4.1 Visitante

O visitante chega por busca, link compartilhado ou navegação direta. Ele deve conseguir encontrar uma página, entender quando o conteúdo foi atualizado, consultar referências e navegar entre links internos sem criar uma conta.

### 4.2 Usuário autenticado

O usuário entra com Google OIDC, edita seu nome de exibição e consulta sua atividade. O principal caminho de contribuição é enviar uma solicitação de editor com contexto suficiente para o administrador avaliar sua intenção.

### 4.3 Editor

O editor cria e altera páginas, escreve um resumo da mudança, visualiza o histórico e reverte conteúdo conforme a política da wiki. Ele não deve precisar de aprovação individual para cada edição no MVP, mas todas as ações permanecem auditáveis e sujeitas à moderação.

### 4.4 Administrador

O administrador governa a wiki. Ele aprova ou rejeita solicitações, concede ou revoga memberships, bloqueia usuários, restaura páginas, reverte alterações, consulta auditoria e executa procedimentos operacionais. O sistema deve sempre oferecer um caminho de recuperação administrativa independente da UI comum.

## 5. Escopo do MVP

O MVP obrigatório contém os seguintes fluxos:

| Área | Incluído no MVP |
|---|---|
| Leitura | Home, páginas publicadas, histórico público, página inexistente e redirects. |
| Identidade | Google OIDC, conta interna, identidade por `provider_sub`, sessão revogável e logout. |
| Governança | Solicitação, aprovação, rejeição e revogação de editor; bloqueio de usuário; auditoria. |
| Conteúdo | Criar, editar, publicar, soft delete, restaurar, histórico, diff textual e reversão não destrutiva. |
| Editor | Textarea Markdown acessível, toolbar, preview, aviso de alterações não salvas e links internos explícitos. |
| Segurança | Autorização server-side, CSRF, XSS, IDOR, mass assignment, rate limiting e threat model. |
| Operação | CI, migrations, health checks, logs estruturados, backup e restauração testada. |
| Qualidade | Testes unitários, integração, E2E, segurança negativa, concorrência e acessibilidade. |

## 6. Fora do MVP

Os itens a seguir não devem bloquear o primeiro lançamento:

| Recurso | Destino |
|---|---|
| Categorias hierárquicas | v1.1 |
| Drafts persistidos e autosave no servidor | v1.1 |
| Upload e processamento de imagens | v1.1 |
| PostgreSQL FTS com filtros avançados | v1.1 |
| Notificações in-app e e-mail transacional | v1.1 |
| Páginas seguidas e watchlists | v1.1 |
| Revisão obrigatória das primeiras edições | v1.1 |
| Discussões, badges, reputação e ranking | backlog |
| Múltiplas wikis expostas na interface | backlog; a entidade `Wiki` existe desde o dia 1 |
| Colaboração em tempo real | backlog |
| Aplicativo móvel, monetização e publicidade | não-objetivo do MVP |
| Microserviços, Kubernetes e Elasticsearch | não-objetivo do MVP |

## 7. Definição de pronto por funcionalidade

Uma funcionalidade só pode ser marcada como pronta quando possuir:

1. comportamento de UI documentado;
2. contrato de API ou justificativa explícita de que é somente leitura estática;
3. regra de negócio e invariantes;
4. persistência e migrations, quando aplicável;
5. autorização e validação no backend;
6. estados de loading, vazio, sucesso e erro;
7. logs e auditoria para ações sensíveis;
8. testes positivos e negativos;
9. documentação operacional quando houver impacto em produção.

Um botão sem backend, uma rota sem autorização ou uma migration sem teste de restauração não constitui funcionalidade pronta.

## 8. Métricas de produto do MVP

A instrumentação deve medir o funil editorial sem coletar PII desnecessária:

| Métrica | Objetivo |
|---|---|
| Visitantes que leem uma página | Verificar descoberta e utilidade do conteúdo. |
| Usuários autenticados | Medir conversão de leitura para participação. |
| Solicitações pendentes e tempo até decisão | Identificar gargalo de governança. |
| Editores ativos e publicações válidas | Medir contribuição real. |
| Conflitos `409` por edição | Avaliar necessidade de melhorar UX de merge. |
| Reversões e páginas bloqueadas | Acompanhar saúde editorial e abuso. |
| Erros `401/403/429/5xx` | Detectar problemas de autorização, abuso e confiabilidade. |

Métricas não devem ser usadas para criar ranking social no MVP. O objetivo inicial é validar o fluxo e a confiabilidade, não maximizar volume de edições.

## 9. Critério de sucesso do lançamento

O MVP é aceitável quando um visitante consegue ler conteúdo, autenticar-se, solicitar edição, ser aprovado por um administrador, criar uma página, editá-la, consultar seu histórico e permitir que um administrador reverta uma revisão sem apagar nenhuma informação histórica. Esse fluxo deve passar em ambiente de staging com os testes de segurança, concorrência, acessibilidade e restauração de backup aprovados.

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[3]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
