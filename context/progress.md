# Progresso do desafio

> Atualizar este arquivo conforme o desafio avança. Última atualização: 2026-07-19.

## Status atual

- [x] Spec do desafio definida (`desafio-outbox-pattern.md`).
- [x] `CLAUDE.md` criado com o resumo do desafio e as regras de trabalho.
- [x] Pasta `context/` criada para manter continuidade entre sessões.
- [x] Stack escolhida: Express (HTTP) + Prisma (ORM/Postgres) + arquitetura MVC (estrutura de pastas).
- [x] Estrutura de pastas: MVC padrão (`controllers/`, `models/`, `routes/`) + pasta irmã `workers/` para relay e consumer, que ficam fora do fluxo MVC.
- [x] Infra decidida: Supabase (Postgres) + Upstash (Redis), gerenciados na nuvem — sem Docker local (ver `decisions.md`).
- [x] `package.json` inicializado com as dependências principais.
- [x] `DATABASE_URL` (Supabase, conexão direta 5432) configurada no `.env`, que já está protegido pelo `.gitignore`.
- [~] `db.js` foi escrito usando o pacote `postgres` por engano (seguindo tutorial da Supabase) — **pendente remover o pacote `postgres` do projeto** assim que o acesso a dados for reescrito com `@prisma/client`.
- [ ] Upstash (Redis) ainda não configurada — `REDIS_URL` pendente.
- [ ] Prisma ainda não inicializado (sem `prisma/schema.prisma`, sem `datasource`/`generator`, sem migration).
- [ ] Nenhuma implementação de rota/worker iniciada ainda (`POST /orders`, relay, consumer, `/evaluation`).

## Próximos passos (não implementados, apenas o que foi discutido)

- **Próximo passo imediato:** criar o `prisma/schema.prisma` (bloco `datasource`, bloco `generator`, depois os `model`s de `orders` e `outbox`), rodar `prisma migrate dev` para criar as tabelas e gerar o client.
- Depois disso: reescrever `db.js` (ou equivalente) para usar `@prisma/client` em vez do pacote `postgres`, e então remover a dependência `postgres` do `package.json`.
- Configurar Upstash (Redis) e preencher `REDIS_URL` no `.env` — usar a string TCP (`redis://`/`rediss://`), não a REST (`https://`).
- Dependências de dev ainda pendentes de decisão: test runner (Jest ou Vitest).

## Decisões pendentes de discussão

- Test runner: Jest vs Vitest (ou `node:test` nativo).
