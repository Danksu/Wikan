# Wikan — Privacidade, Termos e Governança Editorial

**Status:** normativo antes do lançamento público  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Princípio

A Wikan deve coletar e conservar apenas dados necessários para autenticação, segurança, governança e preservação da autoria. O produto não deve presumir que a licença do software determina a licença do conteúdo editorial.

Este documento define comportamento técnico. A política de privacidade, os termos de uso e o texto de contribuição devem ser revisados por profissional habilitado antes de uso público, especialmente em relação à legislação aplicável aos usuários e à operação.

## 2. Dados tratados

| Categoria | Dados | Finalidade |
|---|---|---|
| Identidade | Provider, subject, e-mail snapshot, verificação, nome, avatar | Criar sessão, exibir autoria e permitir recuperação. |
| Conta | UUID, status, memberships, timestamps | Autorizar e governar participação. |
| Conteúdo | Páginas, revisões, resumos, slugs, redirects | Operar a wiki e preservar histórico. |
| Segurança | Request ID, IP pseudonimizado, user agent minimizado, eventos | Prevenir abuso e investigar incidentes. |
| Preferências | Locale, preferências de UI futuras | Personalizar experiência sem afetar autorização. |

E-mail e provider subject não devem ser exibidos em perfil público. IP completo deve ser evitado em auditoria; quando necessário para segurança, usar hash com segredo operacional separado e retenção limitada.

## 3. Direitos e operações de conta

A conta deve possuir operações para:

1. visualizar dados mantidos sobre ela;
2. solicitar exportação em formato estruturado;
3. solicitar exclusão ou anonimização;
4. consultar memberships e requests próprios;
5. sair de sessões ativas quando o produto oferecer essa interface.

A exportação deve incluir dados de conta e lista de contribuições, mas não deve expor segredos internos, IP bruto ou dados de terceiros. O arquivo deve expirar e ser acessível somente ao titular autenticado.

## 4. Anonimização e histórico

Excluir uma conta não deve apagar automaticamente conteúdo publicado por ela. Para preservar integridade editorial e referências, a conta é anonimizada: identities e PII são removidas, referências editoriais apontam para uma representação `Usuário removido` e a revisão continua com data, versão e hash.

O procedimento deve:

1. verificar identidade e autorização do pedido;
2. abrir solicitação auditada sem copiar PII para metadata;
3. invalidar sessões;
4. remover identities, e-mail, avatar e campos pessoais;
5. alterar status para `anonymized`;
6. preservar conteúdo e autoria histórica de forma não identificável;
7. registrar somente o mínimo necessário da operação.

Se houver obrigação de preservar registros de segurança ou disputa, a retenção deve ser justificada, limitada e protegida.

## 5. Licença editorial

O conteúdo publicado pela Wikan deve ser oferecido sob **Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**, salvo se a comunidade definir uma licença compatível em documento posterior. A licença deve aparecer nos termos de contribuição, na interface de publicação e em página de informação da wiki.

No primeiro publish, o usuário deve aceitar a versão atual dos termos editoriais. O aceite registra usuário, wiki, versão do texto e timestamp. Alterações futuras da licença ou dos termos devem definir se exigem novo aceite antes de publicar.

A licença do código do repositório permanece separada da licença do conteúdo. Imagens ou materiais de terceiros devem possuir origem e licença compatíveis; a v1.1 deve guardar metadata de mídia e atribuição quando houver upload.

## 6. Governança editorial

A Wikan deve possuir regras públicas sobre:

| Tema | Regra mínima |
|---|---|
| Autoria | Toda revisão registra usuário ou `Usuário removido`. |
| Reversão | Reversão cria nova revisão e não apaga histórico. |
| Bloqueio | Bloqueio interrompe ações e invalida sessões; motivo pode ser privado. |
| Correção | Erros podem ser corrigidos por nova edição; não editar o passado. |
| Conteúdo abusivo | Admin pode soft-delete, preservar evidência e restaurar ou reverter. |
| Conflito | Editor deve usar diff e não sobrescrever silenciosamente. |
| Mídia futura | Upload deve respeitar licença, privacidade e segurança de arquivos. |

Políticas de conteúdo específicas da comunidade devem ser configuráveis em documentação da wiki, não codificadas como regras dispersas na aplicação.

## 7. Auditoria e retenção

| Registro | Retenção padrão | Observação |
|---|---:|---|
| Revisões publicadas | Enquanto a wiki existir | Não apagar no fluxo editorial. |
| Auditoria administrativa | 180 dias | Pode ser ampliada por justificativa e controle de acesso. |
| Logs técnicos | 30 dias | Redacted, sem tokens e PII desnecessária. |
| Sessões | Até expiração ou revogação | Armazenar somente identificador necessário. |
| Requests editoriais | Enquanto necessários à governança | Após prazo, minimizar motivo e PII. |
| Backups | Conforme política operacional | Criptografados e com janela definida. |

A exclusão de um log por expiração deve ser observável em métrica, mas não deve recriar a PII em outro lugar.

## 8. Acesso administrativo

Acesso a dados pessoais, auditoria, exportações e lixeira deve ser restrito a admins com necessidade operacional. Toda consulta sensível deve registrar ator, finalidade técnica, recurso e resultado. A UI administrativa deve mostrar dados mínimos por padrão e exigir ação deliberada para detalhes.

O admin não pode consultar sessões ou secrets por uma rota comum. Procedimentos de infraestrutura ficam no ambiente operacional e seguem o spec de DevOps.

## 9. Incidentes de privacidade

O sistema deve permitir identificar escopo, timestamps, endpoints e atores afetados sem registrar conteúdo excessivo. Incidentes devem possuir severidade, responsável, contenção, análise de causa, correção e revisão de retenção. O procedimento de comunicação externa deve ser definido pela operação responsável e não improvisado em código.

## 10. Critérios de aceite

| ID | Critério |
|---|---|
| GOV-01 | Primeiro publish exige aceite versionado dos termos editoriais. |
| GOV-02 | Exportação não inclui secrets, tokens, IP bruto ou dados de terceiros. |
| GOV-03 | Anonimização remove PII e preserva histórico com `Usuário removido`. |
| GOV-04 | Conteúdo editorial e licença do software aparecem como conceitos separados. |
| GOV-05 | Consulta de auditoria sensível é autorizada e auditada. |
| GOV-06 | Retenções estão configuradas e documentadas para logs, auditoria e backups. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[2]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"
