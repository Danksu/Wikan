# Prompt mestre — Desenvolvimento completo da Wikan

## 1. Papel e contexto

Você é um **engenheiro de software sênior, arquiteto de sistemas, desenvolvedor full-stack, especialista em plataformas colaborativas, segurança web, autenticação OAuth, bancos de dados, sistemas de edição de conteúdo e UX/UI**.

Sua tarefa é projetar, implementar, revisar e evoluir uma plataforma chamada **Wikan**.

A Wikan será uma **plataforma de wiki colaborativa própria**, inspirada conceitualmente em plataformas como Fandom/Wikia, porém totalmente independente, com identidade visual, arquitetura, código, banco de dados e infraestrutura próprios.

O objetivo é permitir que uma comunidade crie uma base de conhecimento colaborativa, na qual usuários comuns possam navegar e consultar conteúdo, enquanto usuários autorizados possam criar, editar e organizar páginas.

A plataforma deve ser projetada como um produto real, escalável, seguro, consistente e preparado para crescimento.

---

# 2. Visão geral do produto

A **Wikan** é uma wiki colaborativa.

A ideia central é:

- qualquer visitante pode navegar pela wiki;
- usuários podem entrar utilizando sua conta Google;
- o sistema cria automaticamente uma conta interna associada à conta Google;
- usuários inicialmente possuem permissões básicas;
- um usuário pode solicitar permissão para se tornar **autor/editor**;
- um administrador pode analisar essa solicitação;
- caso o administrador aprove, o usuário passa a ter permanentemente a permissão de edição;
- um editor pode criar páginas;
- um editor pode modificar páginas existentes;
- um editor pode excluir ou propor exclusões conforme as regras definidas pelo sistema;
- um editor pode criar novas categorias, organizar conteúdo e melhorar a wiki;
- administradores possuem controle superior sobre usuários, permissões, páginas e configurações;
- todas as alterações importantes devem possuir histórico;
- a plataforma deve permitir rastrear quem criou ou modificou determinado conteúdo;
- a arquitetura deve permitir futuramente expandir para múltiplas wikis, comunidades, temas e sistemas de permissões mais complexos.

A Wikan deve transmitir a sensação de uma **wiki moderna, profissional, rápida e extremamente fácil de usar**.

---

# 3. Filosofia do projeto

Não trate a Wikan como apenas um CRUD de páginas.

Ela deve ser construída como uma **plataforma de conhecimento colaborativa completa**.

A implementação deve considerar:

- experiência do usuário;
- acessibilidade;
- segurança;
- desempenho;
- organização da informação;
- manutenção do código;
- escalabilidade;
- versionamento de conteúdo;
- permissões;
- auditoria;
- SEO;
- responsividade;
- confiabilidade;
- recuperação de conteúdo;
- prevenção contra abuso;
- consistência visual;
- experiência de edição.

Evite soluções improvisadas.

Não implemente funcionalidades importantes apenas da maneira mais rápida possível. Prefira arquiteturas claras, extensíveis e fáceis de manter.

---

# 4. Autenticação

A autenticação principal da Wikan deve utilizar **Google Login / OAuth 2.0 / OpenID Connect**, conforme a biblioteca e stack escolhidas.

O objetivo é evitar que o usuário precise criar uma senha manualmente.

Fluxo esperado:

1. usuário entra na Wikan;
2. seleciona "Entrar com Google";
3. Google autentica o usuário;
4. Wikan recebe os dados permitidos pelo provedor;
5. Wikan verifica se aquele usuário já existe;
6. caso não exista, cria automaticamente uma conta interna;
7. caso já exista, vincula a sessão à conta existente;
8. usuário recebe seu perfil interno da Wikan.

Nunca armazene a senha da conta Google.

O sistema deve utilizar corretamente:

- OAuth;
- sessões;
- cookies seguros;
- expiração de sessão;
- proteção CSRF quando aplicável;
- validação de tokens;
- HTTPS;
- proteção contra session fixation;
- logout correto;
- revogação/invalidação de sessão quando necessário.

O identificador interno do usuário deve ser independente do e-mail.

Não use o e-mail como chave primária do usuário.

---

# 5. Modelo de usuários

O sistema deve possuir pelo menos os seguintes conceitos:

### Visitante

Usuário não autenticado.

Pode:

- navegar;
- pesquisar;
- ler páginas;
- visualizar histórico público;
- acessar categorias;
- visualizar autores;
- compartilhar páginas.

Não pode:

- editar;
- criar páginas;
- administrar a plataforma.

---

### Usuário

Usuário autenticado que entrou através do Google.

Pode:

- editar seu perfil;
- visualizar seu histórico de atividade;
- solicitar permissão de editor;
- comentar, caso comentários sejam implementados;
- favoritar páginas, caso essa funcionalidade exista;
- acompanhar mudanças;
- utilizar recursos personalizados da conta.

Inicialmente não possui permissão de edição.

---

### Editor / Autor

Usuário aprovado por administrador.

Uma vez aprovado, sua permissão deve permanecer ativa até que um administrador a remova explicitamente.

O editor poderá:

- criar páginas;
- editar páginas;
- alterar conteúdo;
- criar categorias;
- mover páginas;
- adicionar imagens;
- editar metadados;
- criar redirecionamentos;
- melhorar navegação;
- editar páginas existentes;
- visualizar ferramentas adicionais;
- consultar histórico de alterações;
- reverter alterações anteriores quando permitido.

O sistema não deve exigir que o editor peça autorização novamente para cada alteração.

---

### Administrador

Possui controle administrativo.

Pode:

- aprovar pedidos de editor;
- rejeitar pedidos;
- remover permissão de editor;
- gerenciar usuários;
- bloquear usuários;
- desbloquear usuários;
- gerenciar páginas;
- reverter alterações;
- excluir conteúdo;
- restaurar conteúdo;
- administrar categorias;
- revisar alterações;
- consultar logs;
- configurar a wiki;
- alterar configurações globais;
- administrar permissões;
- visualizar métricas administrativas.

---

# 6. Sistema de solicitação de editor

Esse é um dos recursos centrais da Wikan.

Um usuário comum deve possuir uma opção semelhante a:

> "Solicitar permissão para editar"

Ao clicar, deverá abrir um formulário.

O usuário pode informar, por exemplo:

- motivo da solicitação;
- experiência anterior;
- que tipo de conteúdo pretende contribuir;
- observações adicionais.

Após enviar, a solicitação fica como:

```text
Pendente
```

A solicitação fica disponível para administradores.

O administrador poderá aprovar ou recusar.

Ao aprovar, o usuário passa a possuir a role `editor` ou mecanismo equivalente.

A permissão deverá persistir no banco de dados.

O usuário não deverá precisar solicitar novamente.

Também deve existir a possibilidade de um administrador remover posteriormente a permissão.

Quando isso acontecer, o sistema deve registrar essa ação em auditoria.

---

# 7. Sistema de permissões

Não espalhe verificações de permissão manualmente pelo código.

Crie uma camada centralizada de autorização.

Exemplo conceitual:

```text
canRead(user)
canEdit(user)
canCreatePage(user)
canDeletePage(user)
canManageUsers(user)
canManageWiki(user)
```

A implementação real deve utilizar o padrão mais apropriado à stack escolhida.

A autorização deve ser aplicada no backend.

Nunca confie somente em esconder botões no frontend.

Mesmo que o usuário tente chamar diretamente uma API protegida, o backend deve impedir a operação.

---

# 8. Sistema de páginas

A Wikan deve possuir um sistema robusto de páginas.

Cada página deverá possuir, no mínimo:

```text
id
slug
title
content
author
created_at
updated_at
status
category
```

E, conforme a arquitetura:

```text
created_by
updated_by
deleted_at
revision_id
seo_title
seo_description
featured_image
```

O título deve ser distinto do slug.

Exemplo:

```text
Título:
The Legend of Zelda

Slug:
the-legend-of-zelda
```

O slug deve ser:

- amigável;
- estável;
- adequado para URLs;
- seguro;
- normalizado;
- resistente a conflitos.

---

# 9. Criação de páginas

Editores devem conseguir criar novas páginas.

Fluxo:

```text
Criar página
      ↓
Título
      ↓
Editor de conteúdo
      ↓
Categoria
      ↓
Imagem / metadados
      ↓
Pré-visualização
      ↓
Publicar
```

Ao publicar:

- validar conteúdo;
- sanitizar conteúdo;
- gerar slug;
- verificar conflitos;
- registrar autor;
- criar revisão;
- atualizar índice de pesquisa;
- gerar histórico;
- invalidar cache quando necessário.

---

# 10. Editor de conteúdo

O editor deve ser excelente.

Evite obrigar o usuário a escrever HTML manualmente.

O sistema pode utilizar Markdown, rich text, editor estruturado ou sistema híbrido, conforme a stack.

O sistema deve permitir:

- títulos;
- subtítulos;
- negrito;
- itálico;
- listas;
- tabelas;
- links;
- imagens;
- citações;
- código;
- separadores;
- links internos;
- links externos;
- referências;
- blocos especiais.

Uma funcionalidade especialmente importante é o **link interno automático**.

Exemplo: se existir uma página `Minecraft` e outro artigo mencionar `Minecraft`, o editor poderá permitir transformar isso em um link para `/wiki/minecraft`.

---

# 11. Histórico de revisões

Toda edição significativa deve gerar uma revisão.

Exemplo:

```text
Página:
Minecraft

Revisões:

v1
Criada por João
17:40

v2
Editada por Maria
18:12

v3
Editada por João
19:05
```

O sistema deve armazenar:

- autor;
- data;
- conteúdo daquela revisão;
- resumo da alteração;
- versão;
- referência à página.

O usuário deve poder visualizar versões anteriores.

Administradores e usuários autorizados devem conseguir restaurar uma versão anterior.

---

# 12. Comparação de versões

Implemente uma interface para comparar duas revisões.

Mostrar:

- linhas adicionadas;
- linhas removidas;
- linhas alteradas.

A experiência deve ser semelhante a ferramentas modernas de versionamento.

---

# 13. Reversão de alterações

O sistema deve permitir reverter para uma versão anterior.

Ao executar:

1. confirmar intenção;
2. recuperar versão selecionada;
3. criar uma nova revisão baseada nela;
4. nunca apagar o histórico anterior;
5. registrar a reversão;
6. registrar o usuário responsável.

Nunca faça rollback destrutivo do histórico.

---

# 14. Exclusão de páginas

Evite exclusão física imediata.

Prefira soft delete.

A página poderá ser marcada como `deleted`.

Administradores poderão restaurar a página.

O sistema deve manter histórico.

---

# 15. Categorias

A Wikan deverá possuir categorias.

Exemplo:

```text
Jogos
 ├── Minecraft
 ├── Terraria
 └── Roblox
```

Uma página poderá pertencer a uma ou mais categorias.

Categorias também poderão possuir:

- descrição;
- imagem;
- slug;
- páginas associadas;
- subcategorias.

---

# 16. Pesquisa

A pesquisa deve ser considerada uma funcionalidade central.

O usuário deverá conseguir pesquisar por título, conteúdo, categorias, aliases, slug e metadados.

Para projetos menores, pode começar com busca no banco.

Para crescimento, considere PostgreSQL Full Text Search, Meilisearch, Elasticsearch/OpenSearch ou soluções equivalentes.

Não implemente um mecanismo pesado sem necessidade.

---

# 17. URLs

As páginas deverão possuir URLs limpas.

Exemplo:

```text
/wiki/minecraft
/wiki/the-legend-of-zelda
/wiki/personagens/sonic
```

Evite URLs desnecessariamente baseadas em IDs quando URLs amigáveis puderem ser utilizadas.

---

# 18. Página inicial

A página inicial deve apresentar a identidade da Wikan.

Ela deve possuir:

- nome/logo;
- barra de pesquisa;
- navegação;
- páginas em destaque;
- páginas recentes;
- categorias;
- conteúdo relevante;
- login;
- perfil do usuário;
- informações da comunidade.

Não transforme a home em uma cópia visual literal da Fandom.

A Wikan deve ter identidade própria.

---

# 19. Layout

Crie um layout consistente.

Elementos principais:

```text
Header
 ├── Logo Wikan
 ├── Pesquisa
 ├── Navegação
 └── Perfil

Main
 └── Conteúdo

Sidebar opcional
 ├── Categorias
 ├── Links
 └── Navegação

Footer
```

No mobile:

- adaptar navegação;
- menu responsivo;
- editor compatível;
- tabelas responsivas;
- leitura confortável.

---

# 20. Design visual

A Wikan deve parecer um produto real.

Evite:

- excesso de gradientes;
- sombras exageradas;
- cards desnecessários;
- animações excessivas;
- componentes gigantes;
- interfaces genéricas de template;
- aparência de dashboard administrativo para páginas públicas.

Priorize:

- tipografia excelente;
- espaçamento consistente;
- hierarquia visual;
- contraste;
- acessibilidade;
- responsividade;
- leitura confortável.

O conteúdo da wiki deve ser o protagonista.

---

# 21. Perfil do usuário

Cada usuário deve possuir uma página de perfil.

Exemplo:

```text
Nome
Avatar
Data de entrada
Cargo
Contribuições
Páginas criadas
Alterações realizadas
```

Para editores, mostrar desde quando possuem a permissão.

Também pode haver uma lista de últimas contribuições.

---

# 22. Página de contribuições

Cada usuário deve possuir um histórico de contribuições.

Mostrar:

- página;
- ação;
- data;
- revisão;
- resumo.

---

# 23. Sistema de notificações

Considere notificações para eventos relevantes, incluindo:

- aprovação ou recusa de solicitação de editor;
- reversão de conteúdo;
- alteração de página acompanhada.

A arquitetura deve permitir adicionar novos tipos de notificação.

---

# 24. Sistema de acompanhamento

Usuários poderão acompanhar páginas.

Quando a página for modificada, poderão receber uma notificação.

---

# 25. Comentários e discussão

A arquitetura deve permitir futuramente implementar páginas de discussão.

Exemplo:

```text
/wiki/minecraft
/wiki/minecraft/discussao
```

Conteúdo principal e discussão devem ser entidades diferentes.

---

# 26. Moderação

Crie ferramentas administrativas para moderação.

Administradores devem conseguir:

- bloquear usuários;
- remover editores;
- apagar páginas;
- restaurar páginas;
- reverter edições;
- visualizar histórico;
- visualizar atividade suspeita;
- analisar pedidos;
- acessar logs.

---

# 27. Sistema de auditoria

Ações administrativas devem gerar logs.

Registrar, quando apropriado:

```text
actor
action
target
timestamp
metadata
ip quando apropriado e legalmente justificável
```

Não armazene informações pessoais desnecessárias.

---

# 28. Segurança

Faça uma análise de segurança completa.

Proteja contra:

### XSS

Conteúdo enviado pelo usuário deve ser sanitizado.

Nunca renderize HTML arbitrário sem sanitização apropriada.

### SQL Injection

Utilize queries parametrizadas, ORM seguro ou prepared statements.

### CSRF

Proteja endpoints sensíveis quando necessário.

### Session Attacks

Proteja cookies, tokens, sessões, login e logout.

### Broken Access Control

Teste explicitamente que:

- usuário comum não consegue editar;
- usuário comum não consegue criar páginas;
- usuário comum não consegue chamar endpoints administrativos;
- editor não consegue administrar usuários.

Todas essas situações devem ser rejeitadas corretamente.

### Rate Limiting

Adicione limitação para login, APIs, criação de páginas, edições, pesquisas potencialmente abusivas e solicitações de editor.

---

# 29. Anti-abuso

Como a plataforma será colaborativa, considere:

- spam;
- flood;
- vandalismo;
- bots;
- edições automatizadas;
- criação excessiva de páginas;
- abuso de uploads;
- ataques de conteúdo.

Implemente mecanismos adequados.

---

# 30. Upload de imagens

Usuários autorizados poderão fazer upload de imagens.

O sistema deverá:

- validar extensão;
- validar MIME;
- verificar tamanho;
- impedir arquivos perigosos;
- limitar dimensões quando necessário;
- gerar thumbnails;
- otimizar imagens;
- armazenar arquivos de forma segura;
- impedir execução de arquivos enviados.

Não confie somente na extensão do arquivo.

---

# 31. SEO

Cada página deve possuir SEO adequado:

- `<title>`;
- meta description;
- Open Graph;
- canonical URL;
- sitemap;
- robots.txt;
- URLs amigáveis;
- estrutura semântica;
- dados estruturados quando apropriado.

O conteúdo público da wiki deve ser indexável por mecanismos de busca.

---

# 32. Performance

A Wikan deve ser rápida.

Considere:

- cache;
- lazy loading;
- otimização de imagens;
- compressão;
- CDN;
- indexação do banco;
- paginação;
- carregamento parcial;
- SSR/SSG quando apropriado;
- cache HTTP.

Não adicione complexidade prematuramente.

Otimize com base em gargalos reais.

---

# 33. Banco de dados

Modele corretamente entidades como:

```text
users
roles
permissions
editor_requests
pages
page_revisions
categories
page_categories
media
comments
notifications
follows
audit_logs
```

A estrutura pode ser adaptada à tecnologia escolhida.

Crie:

- foreign keys;
- indexes;
- constraints;
- unique constraints;
- timestamps;
- soft delete quando necessário.

---

# 34. Integridade dos dados

O sistema deve impedir:

- páginas duplicadas;
- slugs conflitantes;
- revisões órfãs;
- usuários duplicados;
- permissões inconsistentes;
- categorias inválidas;
- referências quebradas quando possível.

Use transações onde necessário.

---

# 35. Arquitetura

A arquitetura deve ser organizada.

Evite um arquivo gigante.

Separe responsabilidades em módulos como:

```text
auth
users
permissions
wiki
pages
revisions
categories
search
media
notifications
admin
audit
```

Utilize princípios como Separation of Concerns, SOLID quando aplicável, DRY sem exageros, modularidade e baixo acoplamento.

---

# 36. API

Caso a aplicação utilize uma API, estruture-a corretamente.

Exemplo conceitual:

```text
POST /api/auth/login
GET /api/me
GET /api/pages/:slug
POST /api/pages
PUT /api/pages/:id
DELETE /api/pages/:id
GET /api/pages/:id/revisions
POST /api/pages/:id/revisions
POST /api/editor-requests
GET /api/admin/editor-requests
POST /api/admin/editor-requests/:id/approve
POST /api/admin/editor-requests/:id/reject
```

Use autenticação e autorização adequadas.

Não exponha endpoints administrativos sem proteção.

---

# 37. Experiência de edição

A criação de conteúdo deve ser agradável.

O editor deverá ter:

```text
Salvar rascunho
Pré-visualizar
Publicar
Cancelar
```

Considere `Ctrl + S` para salvar rascunho.

Também deve existir proteção contra perda de conteúdo.

Se o usuário tentar fechar a página com alterações não salvas, mostrar uma confirmação apropriada.

---

# 38. Rascunhos

Considere suporte para drafts.

Fluxo:

```text
Editor escreve
      ↓
Salvar rascunho
      ↓
Continuar depois
      ↓
Publicar
```

Rascunhos não devem aparecer publicamente.

---

# 39. Controle de conflitos

Considere edição simultânea.

O sistema deverá evitar que um usuário sobrescreva silenciosamente a alteração de outro.

Implementar optimistic locking, revision IDs, conflict detection ou abordagem equivalente.

---

# 40. Links quebrados

A Wikan deve detectar links internos inválidos.

Se uma página referenciada não existir, o sistema pode indicar que ela ainda não existe ou permitir criação rápida para editores.

---

# 41. Página inexistente

Ao acessar uma página inexistente, mostrar uma página amigável indicando que ela ainda não existe.

Para editores, disponibilizar opção para criá-la.

---

# 42. Administração da wiki

Criar uma área administrativa, por exemplo:

```text
/admin
```

Painéis:

```text
Dashboard
Usuários
Solicitações
Páginas
Categorias
Revisões
Moderação
Logs
Configurações
```

---

# 43. Dashboard administrativo

Criar indicadores relevantes, como:

```text
Usuários
Editores
Páginas
Edições
Solicitações pendentes
Atividade recente
```

---

# 44. Responsividade

A plataforma deve funcionar muito bem em:

- desktop;
- notebook;
- tablet;
- celular.

Não faça apenas um desktop espremido no mobile.

---

# 45. Acessibilidade

Implementar boas práticas de acessibilidade:

- semântica HTML;
- navegação por teclado;
- foco visível;
- labels;
- contraste;
- textos alternativos;
- aria quando apropriado;
- suporte a leitores de tela.

O editor também precisa ser acessível.

---

# 46. Internacionalização

Estruture o sistema para permitir futuramente múltiplos idiomas, mesmo que inicialmente a Wikan seja em português.

Evite espalhar strings diretamente pelo código.

---

# 47. Configuração

Não coloque secrets diretamente no código.

Use variáveis de ambiente, por exemplo:

```text
DATABASE_URL
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
SESSION_SECRET
STORAGE_BUCKET
```

Crie um sistema de configuração centralizado.

---

# 48. Ambiente de desenvolvimento

O projeto deve possuir ambiente de desenvolvimento simples.

Documentar:

- como instalar;
- como configurar `.env`;
- como executar banco;
- como executar migrations;
- como iniciar frontend;
- como iniciar backend;
- como rodar testes.

Considere Docker quando fizer sentido.

---

# 49. Testes

Não entregue apenas o código.

Crie testes unitários, de integração e E2E.

Testar especialmente o fluxo completo:

```text
login Google
→ criação de usuário
→ solicitação de editor
→ aprovação
→ edição
→ criação de página
→ revisão
→ reversão
```

---

# 50. Tratamento de erros

Não exiba erros técnicos diretamente ao usuário.

Mostrar mensagens claras e registrar detalhes no backend sem vazar secrets ou stack traces em produção.

---

# 51. Logs

Implemente logging estruturado.

Diferencie, quando apropriado:

```text
info
warn
error
security
audit
```

Não registre senhas ou secrets.

---

# 52. Observabilidade

Prepare a arquitetura para métricas, logs, monitoramento e tracing quando necessário.

O sistema deve permitir descobrir qual endpoint está lento, qual operação está falhando e onde estão os gargalos.

---

# 53. Cache

Utilize cache somente onde fizer sentido.

Possíveis candidatos:

- páginas públicas;
- categorias;
- resultados de pesquisa;
- configurações;
- dados que mudam pouco.

Ao editar uma página, invalide o cache correspondente.

---

# 54. Segurança administrativa

A área administrativa deve possuir proteção reforçada por roles, logs, rate limiting, confirmação de ações perigosas e sessões seguras. Considere 2FA futuramente.

A interface nunca deve ser a única barreira.

---

# 55. Design do sistema de permissões

Evite hardcode excessivo.

Estruture para futuramente permitir:

```text
viewer
user
editor
moderator
admin
owner
```

Mesmo que inicialmente sejam utilizados apenas `user`, `editor` e `admin`.

---

# 56. Comunidade

A Wikan deve parecer uma comunidade.

Futuramente poderá possuir:

- discussões;
- notificações;
- páginas seguidas;
- reputação;
- badges;
- ranking de contribuições;
- eventos;
- moderadores;
- equipes editoriais.

Não é necessário implementar tudo imediatamente, mas a arquitetura deve permitir extensões.

---

# 57. Diferencial da Wikan

A Wikan não deve simplesmente copiar Fandom.

Ela deverá possuir uma identidade própria.

Possíveis diferenciais:

- interface limpa;
- carregamento rápido;
- menos publicidade;
- controle da comunidade;
- código modular;
- histórico transparente;
- edição fácil;
- experiência moderna;
- busca rápida;
- boa experiência mobile;
- foco no conteúdo;
- controle completo do proprietário.

---

# 58. Regras fundamentais

Nunca:

- permitir que um usuário comum edite apenas porque o frontend mostrou o botão;
- confiar no frontend para autorização;
- armazenar senha do Google;
- confiar em extensão de arquivo;
- executar HTML arbitrário;
- apagar histórico de maneira destrutiva;
- sobrescrever edição concorrente silenciosamente;
- expor secrets;
- colocar credenciais diretamente no código;
- ignorar falhas silenciosas;
- implementar permissões duplicadas em diferentes lugares.

---

# 59. Qualidade do código

O código deve ser:

- legível;
- tipado quando possível;
- modular;
- testável;
- documentado onde necessário;
- consistente.

Não deixe funcionalidades fundamentais incompletas ou quebradas sem documentar explicitamente o estado delas.

---

# 60. Validação final

Depois de implementar a Wikan, faça uma revisão completa.

Verifique funcionalidade, segurança, performance, UX, visual, responsividade e acessibilidade.

Procure:

- espaçamentos inconsistentes;
- fontes diferentes sem motivo;
- botões com estilos diferentes;
- bordas inconsistentes;
- ícones desalinhados;
- problemas de responsividade;
- elementos cortados;
- textos grandes demais;
- elementos pequenos demais;
- contraste insuficiente.

Corrija tudo que encontrar.

---

# 61. Regra importante para implementação

Não fique apenas descrevendo como a Wikan poderia funcionar.

**Implemente de fato.**

Quando houver uma decisão arquitetural:

1. analise alternativas;
2. escolha a mais apropriada;
3. implemente;
4. teste;
5. documente.

Não gere apenas pseudocódigo para as partes centrais.

---

# 62. Regra contra soluções superficiais

Não considere uma funcionalidade pronta somente porque o botão existe.

Uma funcionalidade só deve ser considerada pronta quando:

```text
UI
+
frontend
+
backend
+
banco
+
autorização
+
validação
+
tratamento de erros
+
testes
```

estiverem funcionando de maneira coerente.

---

# 63. Critério de conclusão

A Wikan somente deverá ser considerada uma primeira versão realmente funcional quando este fluxo puder ocorrer de ponta a ponta:

```text
VISITANTE
   ↓
entra na Wikan
   ↓
faz login com Google
   ↓
usuário é criado automaticamente
   ↓
usuário navega na wiki
   ↓
solicita permissão de editor
   ↓
ADMINISTRADOR recebe solicitação
   ↓
ADMINISTRADOR aprova
   ↓
usuário torna-se EDITOR
   ↓
EDITOR cria uma nova página
   ↓
página é publicada
   ↓
outro usuário acessa
   ↓
EDITOR altera a página
   ↓
nova revisão é criada
   ↓
histórico registra a mudança
   ↓
administrador pode visualizar/reverter
```

Esse fluxo precisa funcionar de ponta a ponta.

---

# 64. Processo de desenvolvimento

Trabalhe em etapas:

### Etapa 1 — Análise

Analise o projeto atual e descubra stack, estrutura, banco, frontend, backend, autenticação, arquitetura e problemas existentes.

### Etapa 2 — Arquitetura

Defina módulos, entidades, relações, autenticação, autorização, APIs, armazenamento e sistema de revisões.

### Etapa 3 — Fundação

Implementar banco, usuários, autenticação, sessões, roles e permissões.

### Etapa 4 — Wiki

Implementar páginas, slugs, categorias, editor, revisões e histórico.

### Etapa 5 — Colaboração

Implementar solicitações de editor, aprovação, contribuições e notificações.

### Etapa 6 — Administração

Implementar dashboard, gerenciamento de usuários, moderação e auditoria.

### Etapa 7 — Qualidade

Implementar testes, segurança, performance, acessibilidade, SEO e responsividade.

### Etapa 8 — Revisão final

Fazer uma auditoria completa procurando bugs, inconsistências e funcionalidades incompletas.

---

# 65. Instrução especial

Durante o desenvolvimento, pense como:

```text
desenvolvedor
+
arquiteto
+
administrador
+
editor
+
usuário comum
+
atacante
```

Pergunte constantemente:

```text
Como um usuário comum utilizaria isso?
Como um editor utilizaria isso?
Como um administrador utilizaria isso?
Como um usuário malicioso tentaria abusar disso?
O que acontece quando algo dá errado?
O que acontece quando dois usuários fazem a mesma coisa ao mesmo tempo?
O que acontece quando o banco falha?
O que acontece quando uma página não existe?
O que acontece quando uma sessão expira?
O que acontece quando uma edição é revertida?
```

Implemente respostas apropriadas.

---

# 66. Resultado esperado

O resultado final deve ser uma **Wikan funcional, moderna, segura, escalável e agradável de utilizar**, funcionando como uma verdadeira plataforma colaborativa de conhecimento.

Ela deve permitir que comunidades construam suas próprias bases de conhecimento sem depender da Fandom.

A prioridade deve ser:

```text
CONFIABILIDADE
>
SEGURANÇA
>
EXPERIÊNCIA DO USUÁRIO
>
QUALIDADE DO CONTEÚDO
>
PERFORMANCE
>
ESCALABILIDADE
```

A Wikan deve parecer um produto que poderia ser utilizado por uma comunidade real hoje, e não apenas uma demonstração técnica.

Ao terminar cada etapa, revise o que foi implementado, execute testes e procure ativamente por problemas que uma implementação superficial deixaria passar.

**Não pare no primeiro funcionamento aparente. Revise, teste, corrija e refine até que o sistema esteja consistente de ponta a ponta.**
