# Potential ADR: Prisma ORM as the Data Access Layer

**Module**: DATA
**Category**: Technology (ORM / Data Access Layer)
**Priority**: Must Document (Score: 145)
**Date Identified**: 2026-07-01

---

## What Was Identified

Prisma ORM 5.22 (`prisma`, `@prisma/client`) is the sole data-access mechanism for the entire application. The schema-first model is defined in `prisma/schema.prisma`, with a single, shared `PrismaClient` instance instantiated in `src/config/database.ts` and injected into every feature module's repository layer (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS). All queries, including multi-table transactions (e.g., the order-creation flow that atomically inserts an `Order`, its `OrderItem`s, and increments the `OrderNumberSequence` counter), are expressed through Prisma's generated, type-safe client API rather than raw SQL or a query builder.

The choice extends beyond simple CRUD access: Prisma's migration engine (`prisma/migrations/`) with a shadow database (`SHADOW_DATABASE_URL`) drives schema evolution, and `prisma/seed.ts` uses the same client for reproducible local seed data. This structural choice dictates how every module reads and writes data, how transactions are composed (`tx.orderNumberSequence.upsert` inside a Prisma `$transaction`), and how the TypeScript type system stays in sync with the database schema (generated Prisma Client types flow directly into service/repository signatures).

As with the MySQL decision, the repository has a single "init repository" commit (2026-06-24), so there is no multi-commit evolution history — but the pattern is applied consistently and exclusively across all 5 feature modules with no bypass (no raw SQL queries or alternative data-access code paths were found).

## Why This Might Deserve an ADR

- **Impact**: Every repository/service in every feature module (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS) is built directly on the Prisma Client API — it is the exclusive data-access mechanism, with no abstraction layer decoupling business logic from Prisma-specific APIs.
- **Trade-offs**: Prisma was chosen over alternatives (raw `mysql2`/`knex` query builder, TypeORM, Drizzle) — trading some query flexibility and raw-SQL performance control for developer ergonomics, generated types, and an integrated migration workflow.
- **Complexity**: Introduces a code-generation step (`prisma generate`) as a build dependency, a distinct migration/shadow-database workflow, and ties the entire codebase's type safety to Prisma's generated client.
- **Team Knowledge**: Every engineer touching persistence must understand Prisma's schema DSL, migration commands (`prisma migrate dev`/`deploy`), transaction API (`prisma.$transaction`), and generated-client conventions — this is essential knowledge for 80%+ of feature work.
- **Future Implications**: Because there is no repository abstraction shielding business logic from Prisma, migrating to a different ORM or query layer later would require touching every module's data-access code — a significant, multi-week-to-months effort.

## Evidence Found in Codebase

### Key Files
- [`prisma/schema.prisma`](../../../../../prisma/schema.prisma)
  - Full schema-first model definition (7 models, 2 enums) driving both the database schema and generated TypeScript types
- [`src/config/database.ts`](../../../../../src/config/database.ts)
  - Single shared `PrismaClient` instance (`createPrismaClient`) exported for use across all modules
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) - Lines ~240-255
  - Uses `prisma.$transaction` with `tx.orderNumberSequence.upsert` to atomically generate sequential order numbers and persist orders — showcasing reliance on Prisma's transaction API for business-critical consistency guarantees
- [`prisma/migrations/20260519182739_init/migration.sql`](../../../../../prisma/migrations/20260519182739_init/migration.sql)
  - Prisma-generated SQL migration, the only migration so far, defining the full initial schema
- [`prisma/seed.ts`](../../../../../prisma/seed.ts)
  - Uses `PrismaClient` directly to seed customers/products/users for local development

### Code Evidence
```typescript
// src/config/database.ts
export function createPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error'],
  });
}
export const prisma: PrismaClient = createPrismaClient();
```

```typescript
// src/modules/orders/order.service.ts (order number generation inside a transaction)
const seq = await tx.orderNumberSequence.upsert({
  where: { id: 1 },
  create: { id: 1, nextValue: 2 },
  update: { nextValue: { increment: 1 } },
  select: { nextValue: true },
});
const current = seq.nextValue - 1;
```

### Impact Analysis
- Introduced: 2026-06-24 (single "init repository" commit)
- Modified: 0 subsequent commits since init (schema, migration, and client wrapper unchanged)
- Affects: all 5 feature modules' repository layers, the seed script, and the migration/shadow-database workflow
- Git history not available beyond the initial commit; no refactor/evolution themes to report yet

### Alternatives (if observable)
No explicit alternatives (e.g., TypeORM, raw SQL, Knex, Drizzle) are referenced in comments, config, or commit messages — Prisma appears to have been the starting choice with no recorded comparison.

## Questions to Address in ADR (if created)

- What problem was being solved, and why was an ORM chosen over raw SQL/query builders?
- Why Prisma specifically, versus TypeORM, Drizzle, or Knex?
- How is the lack of a repository abstraction layer (direct Prisma Client usage in services) expected to affect testability and future portability?
- What is the migration strategy/discipline for schema changes in team environments (shadow database, migration review process)?
- How are N+1 query risks and performance-sensitive queries (e.g., order listing with joins) monitored and optimized?

## Related Potential ADRs
- MySQL as the Primary Relational Database (same module — Prisma's `mysql` provider and shadow-database migration workflow are directly coupled to this choice)

## Additional Notes
The atomic `OrderNumberSequence` pattern (a single-row counter table updated via `upsert`/`increment` inside a Prisma transaction) is a notable concurrency-safety design embedded in this ORM usage. It is not scored as a separate ADR here (narrow scope: one table, one service method) but is worth capturing as an implementation detail if a formal ADR is written for either this decision or the ORDERS module's transactional consistency strategy.
