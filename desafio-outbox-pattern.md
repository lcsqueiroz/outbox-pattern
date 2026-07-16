# Desafio Técnico: Outbox Pattern com Fila de Eventos

> 💡 Este é um desafio técnico auto-proposto, criado por mim ([lcsqueiroz](https://github.com/lcsqueiroz)) como parte do meu portfólio de projetos backend. A ideia é simular um problema real de consistência em sistemas distribuídos — garantir que um evento não se perca nem se duplique quando o processo falha no meio do caminho — definindo as regras antes de iniciar a implementação.

---

## 📑 Sobre o Desafio

Quando uma operação precisa, ao mesmo tempo, gravar um dado no banco **e** publicar um evento numa fila (ex: "pedido confirmado" → notificar outro serviço), fazer as duas coisas como escritas separadas cria uma janela de inconsistência: se o processo cair entre as duas operações, ou o evento se perde (dado salvo, ninguém avisado), ou é publicado sem o dado ter sido de fato persistido. O padrão Outbox resolve isso gravando o evento na mesma transação do dado de negócio, numa tabela "outbox", e usando um processo separado para publicá-lo na fila de forma confiável depois.

Este projeto se conecta diretamente ao [Processador de Webhooks Idempotente](../desafio-idempotent-webhook-processor.md): o Outbox garante que o evento será entregue pelo menos uma vez (at-least-once), e o consumidor do lado de lá precisa ser idempotente para lidar com possíveis reentregas — as duas peças resolvem, juntas, o problema de consistência ponta a ponta.

## Objetivo

Criar um serviço que, ao executar uma ação de negócio (confirmar um pedido), grave o dado e o evento correspondente na mesma transação de banco de dados, e implementar um processo separado (relay) que publique os eventos pendentes numa fila (BullMQ/Redis), garantindo que nenhum evento se perca mesmo se o processo cair entre a gravação e a publicação.

---

## 🎯 Núcleo obrigatório

Este é o escopo que define a entrega mínima do desafio — deve ser 100% concluído.

### `POST /orders`
- Recebe um payload de pedido (ex: `{ "customer": "...", "amount": 1000 }`).
- Em uma única transação de banco de dados:
  - Insere o pedido na tabela `orders`.
  - Insere um evento correspondente (`order.confirmed`) na tabela `outbox`, com status `pending`.
- Se qualquer uma das duas escritas falhar, a outra deve ser revertida (nenhum pedido sem evento, nenhum evento sem pedido).

### Relay de publicação (Outbox Worker)
- Processo separado que, periodicamente (polling simples, ex: a cada 2 segundos), busca eventos com status `pending` na tabela `outbox`.
- Publica cada evento pendente na fila (BullMQ).
- Só marca o evento como `sent` **depois** de confirmar que foi enfileirado com sucesso.

### Consumer da fila
- Processa o evento publicado (efeito simulado, ex: log de "notificação enviada").
- Deve ser idempotente: se o mesmo evento for entregue mais de uma vez pela fila (comportamento normal de filas at-least-once), o efeito não deve se repetir — reaproveitar a lógica do Processador de Webhooks Idempotente.

### Simulação de falha
- Deve ser possível demonstrar que, se o relay for interrompido depois de um pedido ser criado mas antes do evento ser publicado, o evento não se perde — ele continua `pending` na tabela e será publicado no próximo ciclo do relay assim que o processo voltar.

### `GET /evaluation`
Endpoint de autoavaliação: cria um pedido, verifica que o evento aparece como `pending` na tabela outbox, aguarda o ciclo do relay, confirma que o evento foi publicado e processado exatamente uma vez, e retorna um relatório em JSON.

### Testes mínimos
- Criar um pedido grava o pedido e o evento na mesma transação.
- O relay publica corretamente eventos pendentes e os marca como `sent`.
- O consumer não reprocessa um evento já processado (idempotência).

---

## Exemplos de Resposta

### Pedido criado — `201 Created`

```http
HTTP/1.1 201 Created
Content-Type: application/json
```
```json
{
  "order": {
    "id": "ord_7c19a2",
    "customer": "cus_4471",
    "amount": 1000,
    "status": "confirmed"
  },
  "outboxEvent": {
    "id": "evt_3f0912",
    "type": "order.confirmed",
    "status": "pending"
  }
}
```

### Consulta de status do pedido após o relay publicar — `200 OK`

```json
{
  "order": {
    "id": "ord_7c19a2",
    "customer": "cus_4471",
    "amount": 1000,
    "status": "confirmed"
  },
  "outboxEvent": {
    "id": "evt_3f0912",
    "type": "order.confirmed",
    "status": "sent",
    "sentAt": "2026-07-13T14:32:05.000Z"
  }
}
```

### Relatório do `GET /evaluation`

```json
{
  "timestamp": "2026-07-13T14:35:00.000Z",
  "results": [
    {
      "test": "Pedido e evento são gravados na mesma transação",
      "passed": true,
      "responseTimeMs": 6
    },
    {
      "test": "Relay publica evento pendente e marca como sent",
      "passed": true,
      "responseTimeMs": 2010
    },
    {
      "test": "Consumer processa o evento e aplica o efeito colateral",
      "passed": true
    },
    {
      "test": "Reentrega do mesmo evento não duplica o efeito colateral",
      "passed": true
    }
  ],
  "summary": {
    "total": 4,
    "passed": 4,
    "failed": 0
  }
}
```

---

## ⭐ Diferenciais (bônus — fazer se sobrar tempo/confiança)

- **CDC em vez de polling**: usar `LISTEN/NOTIFY` do PostgreSQL (ou uma abordagem estilo Debezium) para publicar eventos assim que são gravados, em vez de esperar o próximo ciclo de polling.
- **Dead-letter**: eventos que falham repetidamente ao serem publicados ou processados vão para uma fila separada de investigação, após N tentativas.
- **Métrica de lag do outbox**: medir e expor o tempo entre a criação do evento (`pending`) e sua publicação (`sent`), útil para detectar atraso no relay.
- **Comparar BullMQ x RabbitMQ**: implementar a publicação com as duas opções e documentar as diferenças de garantias e complexidade operacional.
- **Múltiplos tipos de evento de negócio**: suportar mais de uma ação (ex: `order.confirmed`, `order.cancelled`), cada uma gerando seu próprio evento na outbox.

## Critérios de avaliação (autoavaliação)

**Núcleo:**
- [ ] Pedido e evento são gravados atomicamente na mesma transação.
- [ ] Relay publica eventos pendentes e atualiza o status corretamente.
- [ ] Consumer processa o evento e aplica o efeito esperado.
- [ ] Consumer é idempotente diante de reentrega do mesmo evento.
- [ ] Endpoint `/evaluation` roda e retorna relatório coerente.

**Diferenciais (opcional):**
- [ ] Publicação via CDC/LISTEN-NOTIFY em vez de polling.
- [ ] Mecanismo de dead-letter implementado.
- [ ] Métrica de lag do outbox exposta.
- [ ] Comparativo BullMQ x RabbitMQ documentado.
- [ ] Suporte a múltiplos tipos de evento de negócio.

---

## Regras

- Desafio sem código gerado por IA (apenas consulta).

## Stack

Node.js, PostgreSQL, BullMQ (Redis)
