# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ Critical rule: no AI-generated code — consultation only

`desafio-outbox-pattern.md` states explicitly: **"Desafio sem código gerado por IA (apenas consulta)"**. The user (Lucas) is an intern building this as a self-study portfolio challenge — the point is for _him_ to develop the reasoning to reach the solution, not to receive it.

**Allowed:** answering conceptual questions, explaining how a piece of code _should_ be written and why (including discussing specific snippets he's written or is considering), code review/feedback on code he wrote, explaining errors/library behavior, breaking a problem into smaller guided steps when he's stuck, keeping `context/` up to date.

**Not allowed:** writing or editing implementation files (production code, tests, migrations, infra config) on his behalf, or producing ready-to-paste code blocks as the solution. If he explicitly asks for an implementation exception, confirm that's intentional before proceeding.

**Difficulty calibration:** he's an intern — keep hints and explanations at a healthy difficulty (guided questions and incremental hints over direct answers), not artificially brutal, but don't oversimplify the core distributed-systems concepts he's here to learn. Full working agreement: `context/rules.md`.

## Session continuity

The `context/` folder persists project state across chat sessions (each new chat otherwise starts cold). At the start of a session, check `context/progress.md` (current status) and `context/decisions.md` (technical decisions + rationale already made). Update `context/progress.md` as the challenge advances and log meaningful decisions in `context/decisions.md` as they're made during consultations.

## Repository state

This repo currently contains only the challenge specification (`desafio-outbox-pattern.md`) — no source code, `package.json`, or project scaffolding exists yet. There are no build/lint/test commands to run because no project has been initialized.

## What this challenge is

A self-proposed backend portfolio challenge implementing the **Outbox Pattern** with an event queue, to solve the dual-write consistency problem: when an operation must both persist data and publish an event, doing these as separate writes creates a window where the process can crash between them, causing the event to be lost or published without the data being durably saved. The outbox pattern solves this by writing the event to an `outbox` table in the _same database transaction_ as the business data, then relaying it to the queue via a separate, at-least-once process.

This project pairs with a companion challenge, `desafio-idempotent-webhook-processor.md` (referenced but not present in this repo): the outbox relay guarantees at-least-once delivery, and the queue consumer here should reuse idempotent-processing logic to handle possible redeliveries.

**Stack:** Node.js, PostgreSQL, BullMQ (Redis).

## Required scope (`Núcleo obrigatório`)

- **`POST /orders`** — in a single DB transaction: insert into `orders`, insert a corresponding `order.confirmed` event into `outbox` with status `pending`. Either both writes succeed or both roll back.
- **Outbox relay (worker)** — separate process, polls `outbox` for `pending` rows (e.g. every 2s), publishes each to BullMQ, and only flips status to `sent` _after_ the enqueue is confirmed.
- **Queue consumer** — processes the published event (simulated side effect, e.g. a "notification sent" log) and must be idempotent: redelivery of the same event must not repeat the side effect.
- **Failure simulation** — must be demonstrable that killing the relay after an order is created but before the event is published does not lose the event; it stays `pending` and is picked up on the next relay cycle after restart.
- **`GET /evaluation`** — self-check endpoint: creates an order, verifies the event is `pending` in the outbox, waits for a relay cycle, confirms the event was published and processed exactly once, and returns a JSON report (see example shapes in `desafio-outbox-pattern.md`).
- **Minimum tests** — order creation writes order + event atomically; the relay correctly publishes pending events and marks them `sent`; the consumer does not reprocess an already-processed event.

## Optional bonus scope (`Diferenciais`)

CDC via PostgreSQL `LISTEN/NOTIFY` instead of polling; dead-letter queue after N failed attempts; outbox lag metric (time between `pending` and `sent`); BullMQ vs RabbitMQ comparison; support for multiple business event types (e.g. `order.confirmed`, `order.cancelled`).

Self-evaluation checklist for both required and bonus scope is in `desafio-outbox-pattern.md` under "Critérios de avaliação".
