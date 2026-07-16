# Progresso do desafio

> Atualizar este arquivo conforme o desafio avança. Última atualização: 2026-07-16.

## Status atual

- [x] Spec do desafio definida (`desafio-outbox-pattern.md`).
- [x] `CLAUDE.md` criado com o resumo do desafio e as regras de trabalho.
- [x] Pasta `context/` criada para manter continuidade entre sessões.
- [ ] Projeto Node.js ainda não inicializado (sem `package.json`).
- [ ] Nenhuma implementação iniciada ainda (`POST /orders`, relay, consumer, `/evaluation`).

## Próximos passos (não implementados, apenas o que foi discutido)

_(nada discutido ainda — atualizar conforme as conversas avançarem)_

## Decisões pendentes de discussão

- Escolha de framework HTTP (Express, Fastify, cru com `http`?).
- ORM/driver de Postgres (Prisma, Knex, `pg` puro?) e como isso afeta a transação atômica orders+outbox.
- Estrutura de pastas do projeto.
