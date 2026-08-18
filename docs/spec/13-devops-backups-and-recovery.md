# Wikan — DevOps, Deploy, Backup e Recuperação

**Status:** normativo para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Objetivo

A operação deve ser reproduzível, auditável e recuperável. O projeto não está pronto quando roda apenas na máquina do autor; deve possuir instruções para instalar, migrar, testar, publicar, monitorar, fazer backup e restaurar.

## 2. Ambientes

| Ambiente | Propósito | Regras |
|---|---|---|
| Local | Desenvolvimento e debugging | Docker Compose; dados descartáveis; OAuth mock ou credenciais de dev. |
| CI | Verificação automática | Banco e Redis efêmeros; sem dados reais; jobs isolados. |
| Staging | Validação próxima à produção | Dataset sintético/anonimizado; E2E, restore, benchmark e smoke deploy. |
| Produção | Usuários reais | Acesso mínimo, secrets protegidos, backups externos e aprovação de deploy. |

Nenhum ambiente deve reutilizar secrets de outro. Produção não deve aceitar `NODE_ENV=development`, valores default ou migrations manuais sem registro.

## 3. CI obrigatório

Cada pull request deve executar:

1. instalação pelo lockfile;
2. lint e formatação;
3. typecheck;
4. testes unitários;
5. testes de integração com PostgreSQL/Redis;
6. testes de segurança negativos;
7. teste de migrations em banco vazio;
8. build do frontend e backend;
9. validação OpenAPI;
10. scan de dependências e secrets;
11. smoke test da imagem Docker.

Merge não deve ser permitido com job obrigatório falhando. O CI deve publicar artefatos de teste e logs sem PII ou secrets.

## 4. Pipeline de deploy

```text
Pull request
  → CI completo
  → revisão de código
  → merge main
  → imagem imutável
  → deploy automático em staging
  → smoke/E2E/restore conforme janela
  → aprovação operacional
  → produção gradual
  → health/readiness
  → monitoramento e rollback se necessário
```

A imagem deve ser identificada por commit/digest, não por `latest`. Configuração e secrets entram em runtime. O deploy deve possuir health check, shutdown gracioso e capacidade de voltar à versão anterior compatível.

## 5. Migrations

Migrations Prisma devem ser versionadas e nunca alteradas depois de aplicadas. Antes de produção:

- aplicar em banco vazio no CI;
- aplicar cópia representativa em staging;
- estimar locks e duração;
- revisar impacto em índices e constraints;
- preparar migração em duas etapas se houver incompatibilidade;
- confirmar backup válido;
- registrar versão do schema e da aplicação.

Migrations destrutivas não podem executar sem plano de retenção, aprovação e teste de restauração. Não editar tabelas manualmente para contornar falha sem registrar incidente e follow-up de migration.

## 6. Backup PostgreSQL

Backups devem ser automáticos, criptografados, armazenados fora do host principal e sujeitos a controle de acesso. A estratégia inicial deve combinar:

| Tipo | Frequência | Uso |
|---|---|---|
| Dump lógico comprimido | Diário | Restauração completa e portabilidade. |
| Snapshot/backup gerenciado | Conforme provider | Recuperação rápida de infraestrutura. |
| Backup de configuração | A cada mudança | Reproduzir deploy sem secrets em claro. |
| Teste de restore | Mensal e antes de release maior | Verificar que o backup é realmente utilizável. |

O banco de produção não deve ser backupado apenas em volume local. Segredos de criptografia devem ser armazenados separadamente e rotacionados com procedimento de recuperação documentado.

## 7. Procedimento de restauração

O procedimento deve ser automatizado por script e documentado:

1. declarar incidente e congelar mutations se necessário;
2. identificar backup e integridade/checksum;
3. provisionar banco isolado;
4. restaurar dump/snapshot;
5. aplicar somente migrations compatíveis;
6. verificar contagem e integridade de pages/revisions/audit;
7. validar login, leitura, histórico e publicação em staging;
8. apontar aplicação para banco restaurado;
9. monitorar erros e outbox;
10. registrar dados perdidos, tempo e decisões.

Nunca testar restore diretamente no banco de produção. O script deve falhar se receber destino não autorizado ou arquivo ausente.

## 8. RPO e RTO

O MVP adota RPO de 24 horas e RTO de 4 horas para banco, com objetivo de leitura em 2 horas. Se a infraestrutura não sustentar essas metas, o release deve ser bloqueado ou a promessa deve ser explicitamente rebaixada. A meta deve ser medida por teste, não apenas escrita em documento.

## 9. Rollback de aplicação

Rollback de código só é seguro quando schema é compatível. Mudanças incompatíveis devem usar expand/contract:

```text
expand schema compatível
  → deploy que escreve ambos formatos
  → backfill e validação
  → deploy que lê novo formato
  → contract somente após janela segura
```

Não executar rollback de código que espera colunas removidas. Em caso de migration problemática, restaurar em ambiente isolado e seguir plano de recuperação, não apagar dados para fazer a aplicação “subir”.

## 10. Secrets

Secrets devem estar em secret manager ou variáveis protegidas. O repositório não deve conter valores reais em `.env`, fixtures, screenshots, logs ou exemplos. CI deve executar secret scanning. Rotação deve cobrir sessão, OIDC, banco, Redis, storage e chaves de backup.

O procedimento de bootstrap do primeiro admin usa acesso operacional e deve remover variável temporária após uso. Não manter endpoint de setup em produção.

## 11. Runbooks mínimos

Criar runbooks para:

| Runbook | Resultado |
|---|---|
| `deploy.md` | Publicar versão e verificar health. |
| `rollback.md` | Voltar aplicação sem quebrar schema. |
| `backup.md` | Criar, verificar e listar backups. |
| `restore.md` | Restaurar em ambiente isolado e validar. |
| `bootstrap-admin.md` | Criar primeiro admin e recuperar acesso. |
| `incident.md` | Declarar severidade, conter e produzir postmortem. |
| `security.md` | Revogar sessões, secrets e investigar abuso. |

## 12. Critérios de aceite

| ID | Critério |
|---|---|
| OPS-01 | Um desenvolvedor consegue iniciar stack local a partir da documentação e lockfile. |
| OPS-02 | CI bloqueia merge por lint, typecheck, testes, build, migration ou secret scan. |
| OPS-03 | Deploy em staging executa migrations e smoke tests automaticamente. |
| OPS-04 | Backup externo é criado e restauração mensal é comprovada. |
| OPS-05 | Rollback de aplicação possui plano compatível com migrations. |
| OPS-06 | Bootstrap e recuperação do primeiro admin não dependem de endpoint público. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/2 "Issue #2 — Auditoria crítica do prompt mestre"

[2]: https://github.com/Danksu/Wikan/issues/3 "Issue #3 — Plano de mitigação e implementação"
