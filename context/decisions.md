# Decisões técnicas

Registro de decisões relevantes tomadas durante as consultorias com o Claude, com o raciocínio por trás (não só o "o quê", mas o "porquê"). Formato livre, uma entrada por decisão.

---

## Relay marca `sent` na confirmação de enfileiramento, não de processamento downstream

**Decisão:** o relay/worker atualiza o status do evento da outbox para `sent` assim que o broker confirma que a mensagem foi aceita na fila — não espera os sistemas consumidores confirmarem que processaram o evento.

**Por quê:** esperar confirmação de processamento downstream acoplaria o relay aos consumidores (se um consumidor cair, o relay travaria). Além disso, implementar esse round-trip de confirmação exigiria outro backend/protocolo de ack entre broker e sistemas externos, fora do escopo do desafio. Isso também bate exatamente com o que o `desafio-outbox-pattern.md` pede: *"Só marca o evento como sent depois de confirmar que foi enfileirado com sucesso"*.

**Nota de esboço:** o desenho original (`outbox-pattern.png`) mostrava o Worker sendo notificado só depois de `Enviar Mensagem para outros sistemas`. Na prática o fluxo correto é: Worker sinaliza o broker → broker confirma enfileiramento → Worker já marca `sent`. O envio aos outros sistemas e o processamento (com idempotência) acontecem depois, de forma assíncrona, e são responsabilidade do *consumer*, não do relay.

---

## Stack: Express + Prisma + arquitetura MVC

**Decisão:** HTTP framework = Express; driver/ORM de Postgres = Prisma; estrutura de pastas = MVC.

**Por quê:** não detalhado na conversa — registrar caso o raciocínio venha à tona depois.

**Atualização (2026-07-19):** durante o setup inicial, o pacote `postgres` (query builder) foi instalado por engano seguindo um tutorial da Supabase, e um `db.js` chegou a ser escrito usando ele em vez do Prisma. Confirmado com Lucas que foi acidental — Prisma segue sendo o ORM oficial do projeto. `db.js` precisa ser reescrito para usar `@prisma/client` antes de remover o pacote `postgres`, senão quebra o import.

**Ponto em aberto (resolvido abaixo):** MVC nasceu para apps request/response (controller responde a uma rota, model é a entidade, view é a resposta). O relay (processo separado, fazendo polling) e o consumer (reagindo a jobs do BullMQ) não são controllers no sentido clássico — não respondem a uma requisição HTTP.

**Resolução:** pasta irmã `workers/` ao lado de `controllers/`, `models/`, `routes/` — `relay.js` e `consumer.js` moram lá, fora do fluxo MVC da API. Avaliadas também: (2) pasta por domínio `outbox/` agrupando tudo relacionado ao pattern; (3) camada `services/` compartilhada entre controllers e workers para não duplicar a lógica de negócio (ex: transação order+outbox). Lucas optou pela mais simples (workers/ irmã); se controller e worker precisarem repetir lógica de acesso a dados/transação, pode valer revisitar a opção 3.

---

## Infra gerenciada na nuvem (Supabase + Upstash) em vez de Docker local

**Decisão:** Postgres via Supabase (free tier) e Redis via Upstash (free tier), em vez de subir os dois com Docker Compose local.

**Por quê:** Lucas não tem experiência com Docker; no Windows, o Docker Desktop depende do WSL2 como backend, o que adiciona atrito de setup antes mesmo de chegar no `docker-compose.yml`. Serviços gerenciados eliminam essa barreira — só precisa de connection strings no `.env`. Trade-off consciente: depende de internet, e foge do "ambiente local completo"; aceitável porque o foco do desafio é a lógica do outbox pattern, não a orquestração de infra.

**Pontos de atenção registrados:**
- BullMQ (via `ioredis`) exige conexão TCP nativa do Redis (`redis://` ou `rediss://`) — a Upstash também oferece uma API REST separada (`https://...`) que **não** serve para o BullMQ. Certificar de copiar a string certa no dashboard.
- Supabase oferece conexão direta (porta 5432) e via pooler/PgBouncer (porta 6543). Como a API é um processo Node long-running (não serverless), a conexão direta tende a ser mais simples com Prisma; se optar pelo pooler, pode ser necessário `pgbouncer=true` na connection string para migrations funcionarem corretamente.

---

## Linha da outbox é atualizada (`update`), nunca deletada

**Decisão:** ao processar um evento, o worker faz `UPDATE` no status da linha da outbox (`pending` → `sent`), em vez de deletar a linha.

**Por quê:** o `GET /evaluation` precisa auditar a transição `pending → sent` de um evento específico. Deletar a linha destruiria esse histórico. Delete seria uma otimização de espaço relevante só em produção com alto volume — fora do escopo deste estudo, que usa testes controlados.
