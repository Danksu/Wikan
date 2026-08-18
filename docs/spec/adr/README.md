# Wikan — Architecture Decision Records

ADRs registram decisões que afetam arquitetura, segurança, dados, operação, licença ou compatibilidade. O spec temático define o estado aprovado; o ADR explica por que a decisão foi tomada e como revertê-la.

## Quando criar um ADR

Criar um ADR antes de:

- trocar Next.js, NestJS, PostgreSQL, Prisma, Redis ou o modelo de sessão;
- alterar snapshot de revisão para delta ou compactação;
- introduzir motor externo de busca, fila ou microserviço;
- mudar o escopo multi-wiki;
- alterar roles, provider de identidade ou bootstrap admin;
- alterar licença editorial ou política de anonimização;
- alterar SLO, RPO, RTO ou retenção;
- adicionar uma feature ao MVP originalmente definido.

## Template

```markdown
# ADR-NNNN — Título da decisão

- Status: proposed | accepted | superseded | rejected
- Data: YYYY-MM-DD
- Responsáveis: nomes ou equipe
- Relacionados: links para issues, specs e ADRs

## Contexto

Qual problema ou decisão exige registro?

## Requisitos e restrições

Quais invariantes, riscos, custos, compatibilidades e prazos precisam ser respeitados?

## Alternativas consideradas

| Alternativa | Benefícios | Custos/riscos |
|---|---|---|
| A | ... | ... |
| B | ... | ... |

## Decisão

O que será feito, de forma normativa?

## Consequências

O que fica mais simples, mais difícil ou diferente?

## Plano de migração/rollback

Como aplicar, validar, reverter ou conviver com versões anteriores?

## Critérios de aceite

Como provar que a decisão foi implementada?
```

## Regras

Um ADR aceito deve atualizar os specs temáticos afetados e o índice `WIKAN_SPEC.md`. Um ADR superseded deve apontar para o novo documento. Não usar ADR para esconder requisitos vagos; decisões ainda abertas devem permanecer claramente marcadas como abertas.
