# Wikan — Domínio, Identidade e Autorização

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Modelo de identidade

A identidade externa e o usuário interno são entidades diferentes. Um `User` possui UUID próprio e pode ter uma ou mais identidades externas no futuro. No MVP há Google OIDC, mas o schema não deve impedir a adição de outro provider.

```text
User 1 ─── N UserIdentity
User 1 ─── N WikiMembership
Wiki 1 ─── N WikiMembership
```

`provider_sub` é o identificador estável retornado pelo provider OIDC. E-mail, nome e avatar são metadados sincronizáveis e não podem ser usados para reassociar uma identidade automaticamente.

## 2. Estados de usuário

| Estado | Pode autenticar | Pode ler | Pode solicitar | Pode editar | Tratamento |
|---|---:|---:|---:|---:|---|
| `active` | Sim | Sim | Conforme membership | Conforme membership | Estado normal. |
| `blocked` | Não | Conteúdo público sem sessão | Não | Não | Sessões existentes são invalidadas. |
| `anonymized` | Não | Conteúdo público | Não | Não | PII e identities são removidos; autoria histórica vira `Usuário removido`. |
| `pending_deletion` | Conforme política | Sim | Não | Não | Janela operacional antes da anonimização definitiva. |

A alteração de estado é administrativa, auditada e protegida por transação. Um usuário não pode alterar seu próprio estado por endpoint de perfil.

## 3. Memberships por wiki

A role não deve ser um campo livre em `users`. O acesso editorial é representado por `wiki_memberships`:

| Role | Escopo | Permissões principais |
|---|---|---|
| `user` | Uma wiki | Ler como autenticado, editar perfil e solicitar editor. |
| `editor` | Uma wiki | Criar, editar, revisar e reverter conforme as policies. |
| `admin` | Uma wiki | Governar usuários, conteúdo, configurações, auditoria e recuperação. |

A UI pública pode exibir “usuário”, “editor” ou “administrador”, mas o backend trabalha com permissões explícitas. Uma pessoa pode ser `editor` em uma wiki e `user` em outra sem criar outra conta.

## 4. Solicitação de editor

### 4.1 Estados

```text
PENDING ── approve ──> APPROVED
   │
   └──── reject ──────> REJECTED

APPROVED ── revoke membership ──> REVOKED
PENDING ── cancel/expire ────────> CANCELLED
```

O MVP deve permitir um único `PENDING` por `(user_id, wiki_id)`. Requests anteriores permanecem para auditoria. Aprovar duas vezes deve produzir o mesmo resultado lógico, sem memberships ou auditorias duplicadas.

### 4.2 Dados mínimos

O formulário deve coletar motivação, experiência relevante opcional, tipo de conteúdo pretendido e aceite dos termos editoriais. O backend limita tamanho, remove campos desconhecidos e aplica rate limiting. Um motivo de rejeição deve ser selecionável ou escrito pelo administrador, sem expor observações internas desnecessárias.

### 4.3 Aprovação

O administrador abre a fila paginada, consulta perfil e histórico público e aprova ou rejeita. A aprovação ocorre em transação: bloquear o request, verificar que ainda está pendente, alterar o estado, criar ou atualizar membership e registrar `EDITOR_REQUEST_APPROVED`. Em caso de corrida, uma operação vence e a outra recebe resposta idempotente ou conflito controlado.

## 5. Policies

Policies devem possuir nomes estáveis e receber `actor`, `wiki` e `resource` quando aplicável:

```text
wiki.read_public(actor, wiki)
editor_request.create(actor, wiki)
editor_request.review(actor, request)
page.create(actor, wiki)
page.read(actor, page)
page.update(actor, page)
page.revert(actor, page)
page.delete(actor, page)
page.restore(actor, page)
user.block(actor, target)
membership.grant(actor, target, wiki)
membership.revoke(actor, membership)
audit.read(actor, wiki)
```

A policy não deve ler dados de um request HTTP diretamente. O use case prepara o contexto e chama a policy. Policies devem ser determinísticas e possuir testes unitários com matriz de atores e estados.

## 6. Matriz de autorização

| Ação | Visitante | `user` | `editor` | `admin` |
|---|---:|---:|---:|---:|
| Ler página publicada | Sim | Sim | Sim | Sim |
| Ler página excluída | Não | Não | Não | Sim, na moderação |
| Ver histórico público | Sim | Sim | Sim | Sim |
| Criar request de editor | Não | Sim | Não necessário | Não necessário |
| Criar página | Não | Não | Sim | Sim |
| Editar página | Não | Não | Sim | Sim |
| Alterar slug | Não | Não | Sim, se policy permitir | Sim |
| Reverter | Não | Não | Sim, se policy permitir | Sim |
| Soft delete | Não | Não | Não | Sim |
| Restaurar | Não | Não | Não | Sim |
| Aprovar/rejeitar request | Não | Não | Não | Sim |
| Gerenciar membership | Não | Não | Não | Sim |
| Bloquear usuário | Não | Não | Não | Sim |
| Ler auditoria | Não | Não | Não | Sim |

A tabela é uma visão de produto; o backend deve avaliar também `status`, escopo `wiki_id`, proteção da página, autor da operação e políticas de moderação futuras.

## 7. Bootstrap e recuperação administrativa

O primeiro admin é criado por comando operacional que recebe `provider_sub` verificado. O comando deve ser idempotente e registrar o executor, timestamp, wiki, identidade promovida e request ID operacional. Se o bootstrap por variável de ambiente for usado em staging, ela deve expirar depois da primeira utilização.

A recuperação deve permitir invalidar sessões, alterar secrets, verificar memberships e promover um novo admin sem abrir um endpoint HTTP público. Todos os procedimentos ficam documentados em `docs/spec/13-devops-backups-and-recovery.md`.

## 8. Mass assignment e campos protegidos

DTOs de perfil podem aceitar apenas `displayName` e preferências permitidas. DTOs de edição podem aceitar título, conteúdo, resumo, slug solicitado e `expectedRevisionId`, mas nunca `createdBy`, `updatedBy`, `status`, `revisionId`, `wikiId` ou membership.

DTOs administrativos devem ser separados dos DTOs públicos. Serializadores não podem simplesmente persistir todo o objeto recebido. Testes devem tentar enviar campos proibidos em todos os endpoints de user, page e membership.

## 9. Auditoria de autorização

As ações seguintes exigem auditoria: bootstrap, login administrativo, aprovação, rejeição, concessão, revogação, bloqueio, desbloqueio, publicação, reversão, soft delete, restauração, mudança de slug, leitura de logs e exportação/anonimização de dados.

A auditoria deve registrar ator pseudonimizado, ação, alvo, wiki, resultado, request ID, timestamp e metadata mínima. Falhas de autorização podem ser logadas como evento de segurança, mas não devem revelar conteúdo privado na resposta pública.

## 10. Critérios de aceite

| ID | Critério |
|---|---|
| DA-01 | Alterar o e-mail do provider não cria nova conta nem quebra a autoria. |
| DA-02 | Um usuário bloqueado não consegue operar com sessão antiga. |
| DA-03 | Duas aprovações simultâneas criam uma única membership. |
| DA-04 | Nenhum endpoint de perfil aceita alteração de role ou estado. |
| DA-05 | Um editor de outra wiki não consegue acessar ou alterar recurso por troca de ID. |
| DA-06 | Toda concessão, revogação e ação administrativa relevante aparece na auditoria. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
