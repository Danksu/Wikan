# Wikan — Conteúdo e Fluxo Editorial

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Princípio editorial

O conteúdo publicado é um artefato comunitário, não uma linha mutável sem memória. Toda mudança significativa deve indicar autor, momento, resumo e base de edição. O sistema deve tornar simples corrigir erros e igualmente simples descobrir o que aconteceu.

## 2. Formato do conteúdo

O MVP usa Markdown restrito armazenado como fonte. O editor é uma área de texto acessível com toolbar de comandos comuns, preview e aviso de alterações não salvas. Não há `contenteditable`, HTML livre nem colaboração em tempo real no MVP.

O perfil mínimo aceita:

| Elemento | Requisito |
|---|---|
| Títulos | `##` a `####`; o título da página é separado do corpo. |
| Ênfase | Negrito, itálico e riscado. |
| Listas | Ordenadas e não ordenadas. |
| Citação | Blockquote sem HTML arbitrário. |
| Código | Inline e bloco com linguagem opcional, sem execução. |
| Tabela | Sintaxe Markdown controlada e saída responsiva. |
| Links | Externos com protocolo permitido e internos por sintaxe explícita. |
| Referências | Links nomeados ou seção de referências; não executar conteúdo embutido. |

Imagens, upload, blocos especiais, templates e shortcodes ficam para a v1.1. O parser deve rejeitar ou tratar como texto qualquer HTML bruto não permitido.

## 3. Links internos

O MVP usa links internos explícitos:

```text
[[minecraft|Minecraft]]
[[personagens/sonic]]
```

O editor pode sugerir páginas existentes por autocomplete, mas o backend resolve o destino no momento da renderização. Se o destino não existir, a UI apresenta link quebrado e oferece criação rápida a editores autorizados. A aplicação não deve comparar cada palavra do texto com todos os títulos em cada renderização.

O formato interno deve armazenar o alvo lógico, não apenas uma URL pronta. Isso permite que mudança de slug continue funcionando por redirect e que o link permaneça associado à wiki correta.

## 4. Pipeline de conteúdo

```text
Markdown recebido
  → limite de tamanho e validação de encoding
  → parse para AST
  → validação de nós e protocolos
  → normalização
  → render HTML permitido
  → sanitização server-side
  → persistência da fonte Markdown + hash
  → renderização segura em leitura
```

A sanitização deve ocorrer também na leitura. Assim, uma correção do sanitizer protege conteúdo antigo sem exigir que todas as revisões sejam regravadas imediatamente. O HTML renderizado deve ser derivado, nunca editado como fonte.

Allowlist inicial de saída: `h2`, `h3`, `h4`, `p`, `br`, `strong`, `em`, `del`, `ul`, `ol`, `li`, `blockquote`, `pre`, `code`, `table`, `thead`, `tbody`, `tr`, `th`, `td` e `a`. Permitir somente `http` e `https` em links. Proibir `javascript:`, `vbscript:`, `data:`, atributos `on*`, `style`, `iframe`, `object`, `embed`, SVG e scripts.

## 5. Criação de página

O fluxo mínimo é:

```text
Editor seleciona “Nova página”
  → informa título
  → escreve Markdown
  → visualiza preview
  → confirma licença editorial
  → publica
```

O backend deriva um slug sugerido, normaliza-o e verifica unicidade com constraint `(wiki_id, slug)`. O cliente não pode escolher `created_by`, `wiki_id`, `status` ou revisão inicial. A publicação cria página, revisão v1, auditoria e outbox na mesma transação.

O título deve ser validado por tamanho, caracteres de controle e nomes reservados. O conteúdo deve obedecer limite configurável; uma página que exceda o limite recebe erro claro antes de abrir uma transação grande.

## 6. Edição de página

Ao abrir uma página para edição, o cliente recebe:

```json
{
  "pageId": "uuid",
  "slug": "minecraft",
  "title": "Minecraft",
  "content": "...",
  "currentRevisionId": "uuid",
  "updatedAt": "2026-08-18T00:00:00Z"
}
```

O formulário envia `title`, `content`, `summary`, `requestedSlug` opcional e `expectedRevisionId`. O resumo é obrigatório para uma edição publicada. O backend carrega o recurso, avalia policy, compara a revisão e cria a próxima versão em transação.

## 7. Conflitos

Se `expectedRevisionId` não corresponder à revisão atual, responder `409 Conflict` com código `PAGE_REVISION_CONFLICT`, revisão atual e a revisão que originou o formulário. A UI deve preservar o texto digitado localmente e oferecer:

1. visualizar lado a lado a versão local e a atual;
2. copiar blocos de uma versão para a outra;
3. recarregar descartando a versão local somente após confirmação;
4. tentar publicar novamente com nova revisão esperada.

Não existe sobrescrita silenciosa. Um overwrite forçado, se aprovado para administradores no futuro, deve ser uma ação separada, confirmada e auditada.

## 8. Revisões

Cada criação, edição significativa ou reversão cria uma revisão snapshot com:

| Campo | Finalidade |
|---|---|
| `version` | Ordenação e identificação humana. |
| `content_markdown` | Fonte completa daquela versão. |
| `content_hash` | Verificação e deduplicação. |
| `summary` | Contexto editorial. |
| `author_id` | Autoria da operação. |
| `based_on_revision_id` | Base carregada pelo editor ou revisão revertida. |
| `created_at` | Momento de publicação. |

A versão não é apagada pelo fluxo normal. O histórico deve ser consultável com paginação e permitir selecionar duas revisões para diff textual. O conteúdo de uma revisão antiga deve ser renderizado com as regras de segurança atuais.

## 9. Reversão

Reverter significa criar uma nova revisão cujo conteúdo é o snapshot selecionado. O resumo automático deve indicar `Reversão para vN` e o administrador pode adicionar motivo. A revisão revertida permanece no histórico e o autor original continua atribuído à revisão original.

A v1.1 pode oferecer rollback em lote de um usuário ou janela temporal. No MVP, a reversão é individual para manter o comportamento auditável e simples de testar.

## 10. Slugs e redirects

O slug é distinto do título e deve ser:

- minúsculo;
- Unicode normalizado e transliterado quando necessário;
- composto por caracteres seguros para URL;
- limitado em comprimento;
- único dentro da wiki;
- estável por padrão.

Alterar o título não altera o slug. Alterar o slug exige permissão editorial, cria `page_redirects` com status 301 e atualiza a URL canônica. Redirects antigos devem apontar diretamente ao slug atual; não criar cadeias.

Slugs reservados para rotas do sistema, como `admin`, `api`, `login`, `health` e `search`, devem ser rejeitados. A tentativa de usar slug de outra wiki ou de criar ciclo de redirect recebe erro de domínio.

## 11. Soft delete e restauração

Excluir página no MVP significa marcar `status = deleted`, preencher `deleted_at`, remover a página da leitura pública e gerar auditoria/outbox. O histórico não é removido. Administrador pode visualizar a lixeira, conferir preview de uma revisão e restaurar a página.

Ao restaurar, o sistema deve verificar se o slug está livre ou se o redirect ainda representa a mesma página. Se existir conflito real, a restauração deve falhar com instrução operacional; não renomear silenciosamente nem sobrescrever outra página.

## 12. Drafts e perda local

Draft persistido é v1.1. No MVP, a UI pode manter cópia temporária local e avisar sobre alterações não salvas ao fechar ou navegar. O aviso deve ser acessível e não impedir logout ou navegação após confirmação.

Quando drafts entrarem, eles terão entidade e política próprias, nunca aparecerão em leitura pública, busca, sitemap ou histórico publicado e usarão a mesma sanitização de conteúdo.

## 13. Critérios de aceite

| ID | Critério |
|---|---|
| EC-01 | HTML e protocolos perigosos não são executados nem persistidos como saída permitida. |
| EC-02 | Criar página gera página, revisão v1, auditoria e outbox atomicamente. |
| EC-03 | Editar exige `expectedRevisionId` e resumo; revisão divergente retorna `409`. |
| EC-04 | Reversão cria nova revisão e preserva todas as anteriores. |
| EC-05 | Mudança de slug cria redirect 301 direto e canonical atualizado. |
| EC-06 | Soft delete remove da leitura pública e restauração preserva conteúdo e histórico. |
| EC-07 | Histórico e diff são paginados e não expõem draft nem PII desnecessária. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/4 "Issue #4 — Auditoria detalhada da especificação"
