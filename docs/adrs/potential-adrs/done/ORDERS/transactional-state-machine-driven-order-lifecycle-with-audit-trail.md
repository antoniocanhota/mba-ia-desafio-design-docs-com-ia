# Potential ADR: Transactional, State-Machine-Driven Order Lifecycle with Embedded Audit Trail

**Module**: ORDERS
**Category**: Architecture / Data Integrity
**Priority**: Must Document (Score: 135/150)
**Date Identified**: 2026-07-01

---

## Existing ADR Context

ℹ️ **RELATED DECISIONS** (no `docs/adrs/generated/` directory exists yet, so this is drawn from other **potential** ADRs already identified in this repository, not formal ADRs):

- **Prisma ORM as the Data Access Layer** (DATA, must-document) — this decision is the ORDERS-specific *application* of that ORM choice to the system's most business-critical write paths (order creation, status transitions). The DATA-module ADR file already explicitly forward-references this analysis: *"the atomic `OrderNumberSequence` pattern... is worth capturing as an implementation detail if a formal ADR is written for either this decision or the ORDERS module's transactional consistency strategy."* This ADR is that follow-up.
- **MySQL as the Primary Relational Database** (DATA, must-document) — the `prisma.$transaction` guarantees described here rely on MySQL's InnoDB transactional/locking semantics (row-level locks on `order_number_sequence`, isolation level for concurrent `stockQuantity` decrements).
- **Stateless JWT Authentication with Custom RBAC** (AUTH, must-document) — `changeStatus`/`create` are invoked with the authenticated `userId` (`req.user.id`) recorded as `createdById`/`changedById` on every order and history row, tying this audit trail to the AUTH module's identity model.

**Timeline Context**: All three related decisions were introduced in the same single "init repository" commit (2026-06-24) as this one — no supersession or evolution pattern to report; these are all foundational, co-designed decisions from the project's initial commit.

**Consolidation Check**: This is *not* a duplicate or a granular subset of the Prisma ORM ADR — it is the specific transactional-consistency and state-machine strategy layered on top of "we use Prisma," analogous to how the DATA-module ADRs cover *which* database/ORM was chosen while this ADR covers *how* the ORM is used to guarantee correctness for the system's core business process.

---

## What Was Identified

`OrderService` (`order.service.ts`, 256 lines — the largest and most complex file in the codebase) implements order creation and order status transitions as a single cohesive architectural pattern with three tightly-coupled parts, all executed inside `prisma.$transaction`:

1. **Explicit finite-state-machine for status transitions** (`order.status.ts`): a declarative transition table (`PENDING → PAID/CANCELLED`, `PAID → PROCESSING/CANCELLED`, `PROCESSING → SHIPPED/CANCELLED`, `SHIPPED → DELIVERED`, with `DELIVERED`/`CANCELLED` terminal) is decoupled from the service into its own module, exposing `canTransition()`, `allowedTransitions()`, `isTerminal()`, plus two derived predicates, `shouldDebitStock()` and `shouldReplenishStock()`, that map specific transitions to inventory side effects. Every mutation to `order.status` is required to pass through `canTransition()` before being persisted — there is no code path that writes a status value without validating it against this table first.
2. **Multi-table transactional atomicity**: both `create()` and `changeStatus()` wrap all reads and writes — `Customer`/`Product` validation, `Order`/`OrderItem` creation or `Order.status` update, `Product.stockQuantity` increment/decrement, `OrderStatusHistory` insertion, and (on creation) `OrderNumberSequence` reservation — inside one `prisma.$transaction(async (tx) => {...})` callback. This guarantees that stock is never debited without a corresponding status change being committed, and no status change is ever recorded without a corresponding audit-history row, even under failure or concurrent access.
3. **Embedded, immutable audit trail** (`OrderStatusHistory`): every order creation and every status transition writes an append-only history row (`fromStatus`, `toStatus`, `changedById`, `reason`, `changedAt`) in the *same* transaction as the state change itself — never as a fire-and-forget side effect, never batched or asynchronous. There is no `update`/`delete` path for `OrderStatusHistory` anywhere in the module; it is write-once by construction (enforced by the repository/service surface, not a DB constraint).

A closely related mechanism lives in the same transaction: `reserveOrderNumber()` uses a single-row `OrderNumberSequence` table (`id=1`, `nextValue`) updated via `upsert(... update: { nextValue: { increment: 1 } })` to generate sequential, human-readable order numbers (`ORD-000001`). Because this upsert runs inside the same `$transaction` as the rest of order creation, it serializes all concurrent order-creation transactions against a single row lock — a deliberate trade-off (gapless-ish sequential numbering vs. write throughput) discussed further below.

This pattern was introduced fully formed in the project's single "init repository" commit (2026-06-24); there is no subsequent commit history on `order.service.ts`, `order.status.ts`, or the relevant sections of `prisma/schema.prisma` to indicate iteration, meaning it should be read as the team's initial, deliberate architectural baseline for order-lifecycle integrity rather than something that evolved reactively (e.g., in response to a production data-corruption incident).

## Why This Might Deserve an ADR

- **Impact**: This is the correctness backbone of the system's core business process (an Order Management System). It governs the two highest-stakes write endpoints in the API (`POST /orders`, `PATCH /orders/:id/status`), touches four Prisma models (`Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`) plus `Product` (stock mutation, in the PRODUCTS module) and `Customer` (validation, in the CUSTOMERS module) — making it the most cross-module-consequential logic in ORDERS.
- **Trade-offs**:
  - Wrapping everything in one long-lived `$transaction` guarantees atomicity but holds row locks (notably on `order_number_sequence` and the touched `Product` rows) for the duration of the whole operation — a scalability ceiling if order volume grows, since every `create()` call serializes on the single sequence row.
  - The FSM is intentionally minimal (a static table, no external workflow-engine dependency, no per-transition hooks/plugins) — simple and easy to audit today, but would need to be redesigned if the business needs parallel/branching workflows, per-tenant custom flows, or asynchronous side effects (e.g., calling an external shipping provider) decoupled from the DB transaction.
  - The audit trail is a first-class, transactionally-consistent table rather than a generic append-only event log or external audit service — cheap to query today (`history: { orderBy: { changedAt: 'asc' } }` on every order fetch) but not designed for high-volume event sourcing or cross-service audit aggregation.
  - `OrderNumberSequence`'s upsert-based reservation is simple and correctness-safe under MySQL's row locking, but is a known, explicitly-flagged contention point: it forces order creation to be effectively single-writer at the database level. Alternatives not used here — a DB `AUTO_INCREMENT` column, a client-generated ULID/Snowflake ID, or a sharded/pre-allocated sequence pool — would trade away strict sequential/gapless numbering for higher write concurrency.
- **Complexity**: High relative to the rest of the codebase — `OrderService` is the largest file (256 lines) and the only service using multi-model `$transaction` composition with nested conditional side effects (`shouldDebitStock`/`shouldReplenishStock`) inside the transaction body.
- **Team Knowledge**: Any engineer adding a new order status, a new side effect tied to a transition (e.g., sending a notification, triggering a refund), or debugging stock/order-count discrepancies must understand this transaction boundary and the FSM guard — getting it wrong (e.g., adding a stock mutation outside the transaction, or a new state without updating `transitions`) would silently reintroduce the exact data-corruption risk this pattern exists to prevent.
- **Future Implications**: If the system needs to scale order-creation throughput (the sequence-table serialization point), integrate asynchronous external systems into the lifecycle (payment gateways, shipping carriers, notifications), or move toward event-driven/microservice decomposition, this synchronous single-transaction design will be the first thing that needs to be revisited — understanding why it was built this way (simplicity and strict consistency over throughput, for a young system with unknown order volume) will matter for that migration.
- **Temporal Context**: Introduced as a complete, coherent pattern in the project's one initial commit; zero iteration since. This means today is the cheapest point at which to document or reconsider it — before more features couple against the current transaction boundaries or the `ORD-NNNNNN` order-number format.

## Evidence Found in Codebase

### Key Files
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) - Lines 50-124 (`create`), 126-179 (`changeStatus`), 204-243 (`debitStock`/`replenishStock`), 245-254 (`reserveOrderNumber`) — the full transactional lifecycle implementation
- [`src/modules/orders/order.status.ts`](../../../../../src/modules/orders/order.status.ts) - Lines 3-37: the declarative FSM transition table and derived stock-effect predicates
- [`prisma/schema.prisma`](../../../../../prisma/schema.prisma) - Lines 74-138: `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence` models and their indexes/relations
- [`src/modules/orders/order.repository.ts`](../../../../../src/modules/orders/order.repository.ts) - Lines 47-64: read-side confirms `OrderStatusHistory` is only ever read (`orderBy: { changedAt: 'asc' }`), never mutated outside `order.service.ts`

### Code Evidence
```typescript
// src/modules/orders/order.status.ts:3-14
const transitions: Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>> = {
  [OrderStatus.PENDING]: [OrderStatus.PAID, OrderStatus.CANCELLED],
  [OrderStatus.PAID]: [OrderStatus.PROCESSING, OrderStatus.CANCELLED],
  [OrderStatus.PROCESSING]: [OrderStatus.SHIPPED, OrderStatus.CANCELLED],
  [OrderStatus.SHIPPED]: [OrderStatus.DELIVERED],
  [OrderStatus.DELIVERED]: [],
  [OrderStatus.CANCELLED]: [],
};

export function canTransition(from: OrderStatus, to: OrderStatus): boolean {
  return transitions[from].includes(to);
}
```

```typescript
// src/modules/orders/order.service.ts:131-167 (changeStatus, abridged)
return this.prisma.$transaction(async (tx) => {
  const order = await tx.order.findUnique({ where: { id }, include: { items: true } });
  if (!order) throw new NotFoundError('Order');
  const from = order.status;
  const to = input.toStatus;
  if (!canTransition(from, to)) throw new InvalidStatusTransitionError(from, to);

  if (shouldDebitStock(from, to)) await this.debitStock(tx, order.items);
  if (shouldReplenishStock(from, to)) await this.replenishStock(tx, order.items);

  await tx.order.update({ where: { id }, data: { status: to } });
  await tx.orderStatusHistory.create({
    data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null },
  });
  // ...
});
```

```typescript
// src/modules/orders/order.service.ts:245-254
private async reserveOrderNumber(tx: TxClient): Promise<string> {
  const seq = await tx.orderNumberSequence.upsert({
    where: { id: 1 },
    create: { id: 1, nextValue: 2 },
    update: { nextValue: { increment: 1 } },
    select: { nextValue: true },
  });
  const current = seq.nextValue - 1;
  return `ORD-${String(current).padStart(6, '0')}`;
}
```

### Impact Analysis
- Introduced: 2026-06-24 ("init repository") — the FSM, transaction strategy, audit trail, and sequence-table pattern all appeared fully formed together in the project's initial commit.
- Modified: 0 follow-up commits touching `order.service.ts`, `order.status.ts`, or `prisma/schema.prisma` since init — no iteration/evolution signal (consistent with the whole repository's single-commit history: 8 total commits in the repo, only 1 of which touches these files).
- Affects: The 256-line `order.service.ts` (largest file in the codebase), `order.status.ts`, `order.repository.ts` (read-only consumer), 4 Prisma models directly + 1 externally (`Product`), and both order-mutating HTTP endpoints exposed via `order.controller.ts`/`order.routes.ts`.
- Recent themes: N/A — no commit history beyond the initial creation to infer intent keywords from; the project is approximately one week old as of this analysis (2026-07-01).

### Alternatives (if observable)
No alternatives are explicitly discussed in code comments or commit messages. The presence of a dedicated `order.status.ts` module (rather than inline `if`/`switch` checks in the service) and a dedicated `OrderNumberSequence` table (rather than a MySQL `AUTO_INCREMENT` column or a UUID-only identifier) are implicit design choices with no recorded rationale in the codebase — exactly the kind of "why" a formal ADR would need to capture.

## Questions to Address in ADR (if created)

- Why was a hand-rolled declarative state table chosen over encoding transition rules directly in the database (e.g., a check constraint or trigger) or via a dedicated workflow/state-machine library?
- Why is stock debit/replenishment tied to specific status transitions inside the same transaction as the audit-log write, rather than being decoupled into an event-driven or eventually-consistent process?
- What is the expected order-creation throughput, and at what volume does the `OrderNumberSequence` single-row upsert become a bottleneck? Is strict sequential/gapless numbering (`ORD-000001`) a hard business requirement (e.g., for invoicing/compliance), or could a non-sequential identifier be adopted if it becomes a scaling constraint?
- Is `OrderStatusHistory` intended to remain purely transactional/relational, or will it need to evolve into a more general event-sourcing/audit-log mechanism (e.g., for compliance reporting or replaying order state)?
- What is the plan for extending the FSM if new statuses or parallel/branching workflows (e.g., partial shipments, returns/refunds) are introduced?

## Related Potential ADRs
- Prisma ORM as the Data Access Layer (DATA) — `potential-adrs/must-document/DATA/prisma-orm-for-data-access.md` — this ADR is the ORDERS-specific application of that ORM to guarantee cross-table consistency; the DATA-module ADR explicitly flags the `OrderNumberSequence` pattern as belonging here.
- MySQL as the Primary Relational Database (DATA) — `potential-adrs/must-document/DATA/mysql-as-primary-relational-database.md` — the transactional/locking guarantees this decision relies on are provided by MySQL/InnoDB.
- Stateless JWT Authentication with Custom RBAC (AUTH) — `potential-adrs/must-document/AUTH/stateless-jwt-authentication-with-custom-rbac.md` — supplies the `userId` recorded as `createdById`/`changedById` in every order and history row created by this pattern.

## Additional Notes
- **Consolidation rationale (Red Flag 5)**: The finite-state-machine (`order.status.ts`), the multi-table `$transaction` boundary, the `OrderStatusHistory` audit trail, and the `OrderNumberSequence` reservation are documented as **one** cohesive decision rather than four separate ADRs, because they are implemented together, invoked together on every order mutation, and exist to serve a single purpose: guaranteeing that an order's persisted state (status, stock, audit trail, order number) is always consistent, even under concurrent access or partial failure. This mirrors how the AUTH module's RBAC and password-hashing mechanisms were folded into one "authentication strategy" ADR rather than split apart.
- **`OrderNumberSequence` concurrency scored, not spun out**: The sequence-table/upsert pattern was evaluated as a potential standalone "consider" tier ADR (mapping.md explicitly flags it as "a potential contention point under concurrency"). Scored independently, it reaches roughly 35-45/150 (Scope+Impact ~10 — single table, single method; Cost to Change ~15-20 — a migration to e.g. `AUTO_INCREMENT` or a distributed ID scheme is a real but bounded, weeks-scale effort with data-migration/format-compatibility risk; Team Knowledge ~15 — relevant mainly to whoever does capacity planning on order creation) — below the 75-point threshold on its own. Rather than discarding it, it is documented here as a first-class trade-off/question within this larger transactional-consistency ADR, since it lives inside the exact same transaction boundary this ADR covers.
- **Integer-cents monetary representation discarded from ORDERS scope (Red Flag 5 / cross-cutting)**: `Order.subtotalCents`/`discountCents`/`totalCents` and `OrderItem.unitPriceCents`/`totalCents` all follow the integer-cents convention, but that convention *originates* in the PRODUCTS module (`Product.priceCents`, per `mapping.md`: "money represented as integer cents... to avoid floating-point errors") and ORDERS merely propagates/snapshots it (`unitPriceCents = product.priceCents` at order-item creation). This is not scored as a separate ORDERS ADR — it is cross-cutting with PRODUCTS and, if documented, belongs either to a PRODUCTS-module analysis or a dedicated cross-cutting "monetary representation" ADR that references both modules.
- No existing ADRs were found in `docs/adrs/generated/` (directory does not exist yet), so no duplicate/related-*formal*-ADR context applies; the "Existing ADR Context" section above instead cross-references other **potential** ADRs already identified in this repository.
