# Wikan — Observabilidade, SLO e SLA

**Status:** normativo operacional para o MVP  
**Versão:** 1.0  
**Última revisão:** 18 de agosto de 2026

## 1. Escopo

Este documento define como medir confiabilidade e como responder a incidentes. Um SLA é uma promessa operacional que só deve ser publicada para usuários quando houver monitoramento, suporte e capacidade compatíveis. Antes disso, os valores abaixo são SLOs internos e objetivos de staging.

## 2. Termos

> **SLO:** objetivo mensurável de confiabilidade, como disponibilidade mensal ou p95 de resposta.
>
> **SLA:** compromisso externo com consequência definida quando o objetivo não é atingido.
>
> **SLI:** métrica que mede o comportamento observado usado para calcular o SLO.
>
> **Error budget:** margem de falha permitida antes de pausar mudanças de risco e priorizar confiabilidade.

O MVP deve operar com SLOs internos. Um SLA público só pode ser anunciado depois de pelo menos três meses de métricas confiáveis e processo de incidentes testado.

## 3. SLOs iniciais

| Serviço | SLI | SLO mensal inicial |
|---|---|---:|
| Leitura pública | Requests HTTP válidos sem erro 5xx ou timeout | 99,5% |
| API autenticada | Requests válidos sem erro 5xx, excluindo `4xx` do cliente | 99,5% |
| Publicação editorial | Comando confirmado sem perda ou corrupção de revisão | 99,9% |
| Integridade histórica | Revisão aceita preservada e recuperável | 100% |
| Backup | Backup válido dentro da janela definida | 99,9% das janelas |
| Outbox | Eventos processados ou visíveis para retry/dead letter | 99,5% em até 15 minutos |
| Acessibilidade | Fluxos críticos sem regressão bloqueadora | 100% dos releases |

Disponibilidade não inclui manutenção previamente comunicada, requests inválidos, limites `429`, falhas causadas por integrações externas fora do controle quando a aplicação informar estado degradado, nem abuso deliberado que exceda políticas.

## 4. Metas de latência

| Fluxo | SLI | Meta p95 |
|---|---|---:|
| Página pública | Tempo de resposta completo | < 800 ms |
| Histórico | Página de até 100 itens | < 1.000 ms |
| Search v1.1 | Consulta FTS paginada | < 1.000 ms |
| API de publicação | Até confirmar commit | < 1.500 ms |
| Admin listagens | Requests, usuários e auditoria | < 1.200 ms |
| Health readiness | Verificação de dependências | < 500 ms |

Registrar p50, p95, p99, taxa de erro, volume, status e dependência responsável. Não registrar conteúdo de página nem tokens para medir latência.

## 5. RPO e RTO

| Meta | Valor inicial | Significado |
|---|---:|---|
| RPO banco | 24 horas | Perda máxima aceitável em cenário de recuperação de backup. |
| RTO banco | 4 horas | Tempo máximo inicial para restaurar serviço essencial. |
| RTO leitura | 2 horas | Objetivo para tornar páginas públicas novamente acessíveis. |
| RTO publicação | 4 horas | Objetivo para reabrir mutations editoriais depois de incidente. |
| Retenção de backup | 30 dias ou política aprovada | Deve incluir cópias fora do host principal. |

Se o produto exigir RPO/RTO menores, revisar custo, replicação, backups e SLA antes de anunciar.

## 6. Error budget

Para cada SLO, calcular mensalmente a margem de falha. Enquanto houver budget, mudanças normais podem seguir. Quando o budget for consumido em 50%, a equipe deve abrir investigação; em 100%, mudanças não urgentes de alto risco devem ser pausadas até correção ou aprovação explícita.

Corrigir integridade de revisão, perda de dados, vulnerabilidade crítica ou indisponibilidade de recuperação sempre tem prioridade sobre o budget de uma feature.

## 7. Monitoramento

Métricas mínimas:

| Grupo | Métricas |
|---|---|
| HTTP | Requests, status, duração, rota normalizada, tamanho de resposta. |
| Auth | Login bem-sucedido/falho, callback rejeitado, sessão revogada. |
| Editorial | Publicações, conflitos, reversões, soft deletes, drafts futuros. |
| Segurança | `401/403`, `429`, payload rejeitado, tentativas mass assignment, CSRF. |
| Banco | Conexões, locks, latência, erros, tamanho, crescimento de revisões. |
| Redis | Memória, evictions, latência, sessões, rate limits. |
| Outbox | Pendentes, idade do evento mais antigo, retries, falhas permanentes. |
| Backup | Último sucesso, tamanho, duração, restauração de teste. |

## 8. Alertas

| Severidade | Exemplo | Resposta |
|---|---|---|
| P0 | Perda/corrupção de histórico, acesso admin comprometido ou indisponibilidade total | Acionar imediatamente, conter, preservar evidência e comunicar responsáveis. |
| P1 | Leitura indisponível, publicação falhando em massa, backup ausente | Engenheiro de plantão inicia resposta em até 30 minutos. |
| P2 | Degradação significativa, outbox atrasado, erro de busca | Investigar em horário de operação e criar plano. |
| P3 | Erro isolado, métrica anômala sem impacto amplo | Registrar e priorizar no backlog. |

Os tempos são objetivos internos até a existência de suporte formal. Nunca esconder um incidente alterando métrica, removendo logs ou desabilitando alertas sem registro.

## 9. Processo de incidente

1. Detectar e declarar incidente com severidade.
2. Nomear comandante, responsável técnico e responsável por comunicação.
3. Conter impacto sem destruir evidência.
4. Registrar timeline com UTC e request IDs.
5. Restaurar serviço ou modo degradado seguro.
6. Validar integridade do banco e das revisões.
7. Comunicar status de forma proporcional.
8. Encerrar somente após métrica estável e checklist de recuperação.
9. Produzir postmortem sem culpabilização, com causa, impacto, ações e responsáveis.

## 10. Modo degradado

Se Redis falhar, o sistema pode bloquear login ou operar com limites conservadores; nunca deve aceitar mutations administrativas sem sessão segura. Se busca falhar, páginas por slug continuam disponíveis. Se outbox atrasar, publicação permanece válida, mas o painel operacional deve indicar side effects pendentes. Se banco estiver indisponível, o sistema deve responder erro seguro e não simular sucesso.

## 11. Critérios de aceite

| ID | Critério |
|---|---|
| SLA-01 | SLOs têm métricas calculáveis sem PII desnecessária. |
| SLA-02 | Backup e restauração produzem evidência da meta RPO/RTO. |
| SLA-03 | Alertas cobrem 5xx, latência, banco, Redis, outbox e backup. |
| SLA-04 | Incidentes possuem severidade, comandante, timeline e postmortem. |
| SLA-05 | O sistema tem comportamento seguro quando Redis, busca ou outbox falham. |
| SLA-06 | Nenhum SLA público é anunciado antes de dados operacionais suficientes. |

## Referências

[1]: https://github.com/Danksu/Wikan/issues/1 "Issue #1 — Auditoria crítica completa da Wikan"

[2]: https://github.com/Danksu/Wikan/issues/3 "Issue #3 — Plano de mitigação e implementação"
