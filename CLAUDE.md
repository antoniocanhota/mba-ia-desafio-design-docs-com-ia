# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is an MBA challenge deliverable ("Da Reunião ao Documento: Design Docs Gerados por IA"). The repo contains:

1. A working **Order Management System (OMS)** REST API (`src/`, `prisma/`, `tests/`) — this is the *existing* application, provided as context.
2. `TRANSCRICAO.md` — the literal transcript of a technical meeting where a team decided to build an **Order Webhook Notification System** (outbox pattern, retries/DLQ, HMAC auth) on top of the OMS.
3. A **documentation package** to be produced in `docs/` (PRD, RFC, FDD, ADRs, Tracker) plus a process `README.md`, describing that not-yet-built webhook feature.

**The deliverable for this repo is purely documentational.** `src/`, `prisma/`, `tests/`, and config files must not be modified — they exist only as reference material to ground the documentation (e.g. citing real file paths, classes, and patterns). Do not "fix" or refactor application code found here unless a user explicitly asks for app-level work outside the challenge scope.

Full requirements and acceptance criteria for the documentation package are in `README.original.md` (the original assignment brief). Read it before generating or editing anything in `docs/`.

## Working on the documentation package

- Every claim in `docs/*.md` must be traceable to either `TRANSCRICAO.md` (with a `[hh:mm] Nome` timestamp) or an actual file/symbol in the codebase — no invented requirements or decisions.
- Document altitude matters and must not overlap:
  - `docs/PRD.md` — product/business level (why + what)
  - `docs/RFC.md` — architecture proposal for review (how we intend to solve it + open questions), concise
  - `docs/adrs/ADR-NNN-*.md` — one closed architectural decision per file, MADR-style (Status, Contexto, Decisão, Alternativas Consideradas, Consequências)
  - `docs/FDD.md` — implementation-level spec (flows, contracts, error matrix, integration with existing code)
  - `docs/TRACKER.md` — cross-cutting traceability table mapping every documented item back to `TRANSCRICAO.md` or source code
- ADRs live in `docs/adrs/` named `ADR-NNN-titulo-em-kebab-case.md`.
- The FDD's error codes must use the `WEBHOOK_*` prefix, and its "Integração com o sistema existente" section must cite at least 4 real paths from `src/`.
- Suggested production order: ADRs → RFC → FDD → PRD → Tracker → README (see `README.original.md`'s "Ordem de execução sugerida" for the full rationale).
- `docs/adrs/mapping.md`, `docs/adrs/potential-adrs/`, and `docs/adrs/generated/` are scratch/working artifacts from an ADR-generation agent workflow (see Skills below), not part of the final deliverable set unless promoted into a numbered `ADR-NNN-*.md`.
- **All deliverable documentation must be written in Brazilian Portuguese (pt-BR)**: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, every `docs/adrs/ADR-NNN-*.md`, and the root `README.md`. Section headings, prose, and table contents should all be in Portuguese, matching the language of `TRANSCRICAO.md` and `README.original.md`. Keep code identifiers, file paths, error codes (e.g. `WEBHOOK_*`), and technical terms without an established Portuguese equivalent (e.g. "outbox", "webhook", "backoff") as-is.

### Available skills for doc generation

This project has custom skills/agents (under `.claude/`) for architecture analysis and ADR generation: `project-analizer` (architectural/component/dependency reports), `adrs-management` (codebase mapping → potential ADR identification → formal ADR generation → cross-linking), and `diagrams-generator` (C4/Mermaid diagrams from an FDD). Use these instead of writing analysis from scratch when producing the design docs.

## Commands (for the reference application in `src/`)

```bash
npm run dev          # start API with tsx watch, loads .env
npm run build         # tsc build to dist/ (tsconfig.build.json)
npm start             # run built server

npm run db:migrate    # prisma migrate dev
npm run db:reset      # prisma migrate reset --force
npm run db:seed       # tsx prisma/seed.ts

npm test              # vitest run (all tests)
npx vitest run tests/orders.test.ts   # single test file
npm run test:watch

npm run lint
npm run format
docker compose up -d  # start MySQL (see docker-compose.yml)
```

Requires `.env` (see `.env.example`) with `DATABASE_URL`, `SHADOW_DATABASE_URL`, `JWT_SECRET`. Tests run against a real MySQL via Prisma (`tests/setup.ts` truncates all tables before each test; `fileParallelism: false` / `singleFork: true` in `vitest.config.ts` because tests share one DB).

## Architecture of the reference application

Layered, per-module structure under `src/modules/{auth,users,customers,products,orders}/`, each with `*.controller.ts` → `*.service.ts` → `*.repository.ts` → `*.schemas.ts` (Zod) → `*.routes.ts`. Dependencies are wired manually (no DI container) in `buildControllers()` in `src/app.ts`, then mounted per-module in `src/routes/index.ts` under `/api/v1`.

Key shared pieces likely to be referenced by webhook documentation:

- **Order state machine** (`src/modules/orders/order.status.ts`): explicit transition table (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, with `CANCELLED` branches). `canTransition`, `shouldDebitStock`, `shouldReplenishStock` gate side effects.
- **`OrderService.changeStatus`** (`src/modules/orders/order.service.ts`): the transactional heart of the app — validates the transition, debits/replenishes stock, updates `Order.status`, and inserts an `OrderStatusHistory` row, all inside one `prisma.$transaction`. This is the natural hook point for emitting webhook events (e.g. an outbox insert in the same transaction).
- **Error classes** (`src/shared/errors/`): `AppError` base with `statusCode` + `errorCode` + optional `details`; concrete errors in `http-errors.ts` (`BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, plus domain-specific `InvalidStatusTransitionError`, `InsufficientStockError`). Webhook error codes should follow this same class/`errorCode` pattern.
- **Centralized error middleware** (`src/middlewares/error.middleware.ts`): maps `AppError` → JSON `{ error: { code, message, details? } }`, plus special-cases `ZodError` and known Prisma error codes (`P2002` unique constraint → 409, `P2025` not found → 404), and falls back to a logged 500.
- **Auth** (`src/middlewares/auth.middleware.ts`): JWT bearer auth (`authenticate`) populates `req.user = { id, email, role }`; `requireRole(...roles)` gates `ADMIN`/`OPERATOR`. Webhook secret/signature auth would be a distinct scheme from this user-facing JWT auth.
- **Logger**: Pino, via `src/shared/logger/index.ts` and `request-logger.middleware.ts` (assigns `req.id` for correlation, used by the error middleware).
- **Data model** (`prisma/schema.prisma`, MySQL): `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` (append-only audit trail of status changes), `OrderNumberSequence` (a single-row counter table used transactionally in `OrderService.reserveOrderNumber` to generate sequential `ORD-000001`-style numbers — a useful precedent for an outbox/sequence pattern).

There is currently no outbox, event, queue, or webhook mechanism anywhere in `src/` — that absence is the entire premise of the feature being documented.
