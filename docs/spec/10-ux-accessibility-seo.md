# Wikan — UX, Acessibilidade e SEO

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Direção de experiência

A Wikan deve parecer uma plataforma de conhecimento, não um dashboard aplicado a páginas públicas. A leitura precisa ser confortável, a navegação previsível e a contribuição possível sem conhecimento de HTML.

O design visual deve possuir identidade própria, tipografia legível, espaçamento consistente, contraste adequado e componentes reutilizáveis. Gradientes, sombras, animações e cards só entram quando melhorarem compreensão ou feedback.

## 2. Arquitetura de informação

### 2.1 Navegação pública

O cabeçalho deve apresentar marca, pesquisa, navegação principal e estado de sessão. A página pública deve conter título, metadados editoriais, corpo, navegação interna e referências quando houver. O rodapé deve incluir informações da comunidade, licença editorial e links de privacidade.

### 2.2 Navegação administrativa

A área `/admin` deve ficar visualmente separada da leitura pública e possuir menu para dashboard, requests, usuários, páginas, lixeira, auditoria e configurações. O backend continua sendo a barreira real; a navegação é somente uma representação da autorização.

### 2.3 Página inexistente

Uma URL sem página deve informar que o conteúdo não existe, preservar o termo pesquisado e oferecer criação somente a editores autorizados. Não exibir stack trace, IDs internos ou mensagem ambígua.

## 3. Fluxos críticos

| Fluxo | Resultado esperado |
|---|---|
| Ler | Visitante chega ao conteúdo em poucas ações, entende estado e autoria. |
| Login | Usuário retorna à página original após OIDC ou recebe erro recuperável. |
| Solicitar editor | Formulário informa finalidade, limites e estado pendente. |
| Criar | Editor escreve, visualiza preview, aceita termos e publica. |
| Editar | Editor vê revisão base, salva resumo e recebe confirmação clara. |
| Conflito | Conteúdo local é preservado e diff oferece recuperação. |
| Reverter | Admin vê revisão alvo, confirma motivo e recebe nova revisão criada. |
| Bloquear | Admin vê impacto, confirma e sistema invalida sessões. |

## 4. Editor acessível

O editor deve usar `<textarea>` ou componente equivalente com nome acessível, toolbar com botões de teclado e preview alternável. Cada botão deve possuir label, estado de pressionado quando aplicável e ordem de foco lógica.

Requisitos mínimos:

- `Ctrl/Cmd+S` deve disparar salvamento local/preview sem submeter uma publicação inesperada;
- alterações não salvas devem ser anunciadas e confirmadas antes de sair;
- erro de validação deve ser associado ao campo e anunciado;
- conflito deve possuir título, descrição e foco inicial no resumo do problema;
- preview deve ser navegável por teclado e não executar conteúdo editorial;
- o usuário deve poder editar sem depender de mouse, cor ou ícone isolado.

Colaboração em tempo real e rich text não fazem parte do MVP. A decisão de usar textarea é deliberada para reduzir superfície de acessibilidade e complexidade.

## 5. Estados de interface

Toda tela de lista ou mutation deve definir:

| Estado | Comportamento |
|---|---|
| Loading | Skeleton ou indicador associado à região, sem deslocamento excessivo. |
| Vazio | Explicação clara e ação útil quando existir. |
| Erro | Mensagem segura, request ID opcional para suporte e retry quando recuperável. |
| Sucesso | Confirmação textual e atualização coerente do estado. |
| Sem acesso | Explicar que a ação exige outra permissão sem vazar recurso privado. |
| Offline/timeout | Preservar entrada local quando possível e permitir tentar novamente. |

## 6. Responsividade

A UI deve funcionar em desktop, notebook, tablet e celular. O mobile não pode ser apenas um desktop comprimido. Tabelas devem possuir estratégia de overflow ou transformação legível; menus devem ser operáveis por teclado e toque; o editor deve permitir leitura e escrita sem zoom horizontal.

Breakpoints devem ser definidos no design system, não repetidos em componentes aleatórios. Testar viewport estreito, fonte aumentada, zoom de 200% e orientação alterada.

## 7. Acessibilidade

Meta: WCAG 2.2 AA para os fluxos de leitura, login, request, edição e administração. Requisitos mínimos:

| Área | Requisito |
|---|---|
| Semântica | Landmarks, headings em ordem e elementos nativos antes de ARIA. |
| Teclado | Todas as funções operáveis sem mouse; foco visível e não perdido. |
| Contraste | Texto, controles e estados com contraste adequado; não depender apenas de cor. |
| Formulários | Label, instrução, erro e estado vinculados ao controle. |
| Leitor de tela | Mudanças assíncronas e conflito anunciados com live regions adequadas. |
| Imagens | Alt text contextual; mídia decorativa marcada como tal. |
| Movimento | Respeitar `prefers-reduced-motion`. |
| Conteúdo | Código, tabelas, links e citações com semântica útil. |

CI deve executar axe-core ou equivalente. Teste manual com leitor de tela deve cobrir pelo menos home, leitura, login, request, editor e aprovação administrativa.

## 8. SEO técnico

Páginas publicadas devem fornecer:

- `<title>` derivado de título seguro;
- meta description limitada e controlada;
- canonical auto-referente;
- Open Graph e Twitter Card quando aplicável;
- sitemap apenas de páginas publicadas;
- `robots.txt` sem bloquear conteúdo público por engano;
- headings semânticos;
- links internos resolvíveis;
- redirect 301 para slug antigo;
- status HTTP correto para página inexistente e excluída.

Drafts, requests, auditoria, perfil administrativo e páginas excluídas não devem ser indexáveis. O sitemap deve ser invalidado depois de publicação, exclusão, restauração ou mudança de slug.

## 9. Design system mínimo

Antes de construir telas, definir tokens de cor, tipografia, spacing, radii, focus ring, z-index, estados de botão, campos, links, banners, tables, dialog e toast. Componentes devem possuir estados disabled, loading, error e focus.

A página pública deve usar largura de leitura adequada e não exibir painel administrativo ou métricas internas. A interface administrativa pode priorizar densidade, desde que mantenha acessibilidade e hierarquia.

## 10. Critérios de aceite

| ID | Critério |
|---|---|
| UX-01 | Todos os fluxos críticos possuem loading, vazio, erro e sucesso definidos. |
| UX-02 | O editor é operável por teclado, possui labels e preserva conteúdo local em conflito. |
| UX-03 | axe-core não encontra violações críticas nos fluxos definidos. |
| UX-04 | Leitor de tela consegue navegar por título, corpo, formulário e mensagens de erro. |
| UX-05 | Página publicada possui canonical, metadata e status HTTP coerentes. |
| UX-06 | Layout funciona em mobile e com zoom de 200% sem perda de conteúdo essencial. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
