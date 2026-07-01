# Component Deep Analysis Report: OrderService

## 1. Executive Summary

`OrderService` (`src/modules/orders/order.service.ts`) is the transactional core of the Order Management System REST API. It orchestrates the full lifecycle of an `Order` aggregate: creation (with line-item pricing, discount validation, and order-number generation), status-machine transitions (with conditional stock debit/replenishment), listing, retrieval, and deletion.

Unlike the `ProductService` and `CustomerService` siblings — which delegate all persistence to a dedicated repository (`ProductRepository`, `CustomerRepository`) — `OrderService` is architecturally an outlier. It holds a direct reference to the raw `PrismaClient` (`order.service.ts:29`) in addition to `OrderRepository`, and uses that raw client to open `$transaction` blocks that read/write `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, and `OrderNumberSequence` directly via `tx.*` calls. `OrderRepository` is only used for the simpler, non-transactional operations (`list`, `getById`, `delete`'s existence check). This makes `OrderService` a de-facto Unit-of-Work coordinator across three other entities' tables (`Product.stockQuantity`, `OrderNumberSequence`, and indirectly `Customer`), bypassing `ProductRepository` and `CustomerRepository` entirely for these operations.

Key findings: (1) all business-critical write logic — pricing, order numbering, stock debit/replenishment, and state transitions — lives inside two `$transaction` closures (`create` at order.service.ts:58-123, `changeStatus` at order.service.ts:131-178); (2) the order-number sequence is guaranteed unique/monotonic only by relying on the surrounding Prisma transaction's isolation level, with no explicit row locking, unique-retry, or optimistic-concurrency mechanism visible in code; (3) stock debit validates availability and decrements atomically inside the transaction, but stock replenishment on cancellation performs no corresponding validation; (4) the component has REST-level integration test coverage for the happy paths and several error paths, but lacks coverage for concurrency scenarios, multi-item stock partial-failure ordering, and several status-transition edge cases (e.g., PROCESSING → CANCELLED replenishment, SHIPPED → DELIVERED, terminal-state re-transition attempts other than PENDING → SHIPPED).

## 2. Data Flow Analysis

### 2.1 Order Creation (`POST /api/v1/orders`)

```
1. Request enters Express router: order.routes.ts:18 (POST /) after authenticate middleware (order.routes.ts:14)
2. Body validated against createOrderSchema (Zod) via validate middleware — order.routes.ts:18, order.schemas.ts:11-16
3. OrderController.create (order.controller.ts:28-36) checks req.user is present (else UnauthorizedError), then calls OrderService.create(req.body, req.user.id)
4. OrderService.create (order.service.ts:50-124):
   a. Validates items.length > 0 (order.service.ts:51-53) — redundant with Zod .min(1) but re-checked in-service
   b. Aggregates duplicate productId line items into summed quantities (aggregateItems, order.service.ts:194-202)
   c. Opens this.prisma.$transaction(async (tx) => {...}) (order.service.ts:58)
   d. Inside tx: loads Customer by id (tx.customer.findUnique) — 404 if missing (order.service.ts:59-60)
   e. Inside tx: loads all Products by aggregated productIds (tx.product.findMany) — 404 if count mismatch (order.service.ts:62-65)
   f. Inside tx: rejects if any matched product is inactive (422 UNPROCESSABLE_ENTITY, order.service.ts:66-73)
   g. Inside tx: computes itemsToCreate with server-side unitPriceCents/totalCents from Product.priceCents (order.service.ts:75-84) — client-submitted prices are never trusted
   h. Inside tx: computes subtotalCents (order.service.ts:86), validates discountCents <= subtotal (400 ValidationError, order.service.ts:87-91), computes totalCents (order.service.ts:92)
   i. Inside tx: calls reserveOrderNumber(tx) which upserts OrderNumberSequence(id=1) and derives "ORD-000123" format (order.service.ts:93, 245-254)
   j. Inside tx: creates Order + nested OrderItem[] + nested OrderStatusHistory[1] (status PENDING) in a single tx.order.create with include (order.service.ts:95-120)
5. Transaction commits; OrderWithRelations returned to controller
6. OrderController serializes response with 201 status (order.controller.ts:32)
```

### 2.2 Order Status Change (`PATCH /api/v1/orders/:id/status`)

```
1. Request enters order.routes.ts:19-23 (PATCH /:id/status) after authenticate
2. params validated by orderIdParamSchema, body by updateOrderStatusSchema (order.schemas.ts:4, 18-21)
3. OrderController.changeStatus (order.controller.ts:38-46) checks req.user, calls OrderService.changeStatus(id, body, userId)
4. OrderService.changeStatus (order.service.ts:126-179):
   a. Opens this.prisma.$transaction(async (tx) => {...}) (order.service.ts:131)
   b. Inside tx: loads Order with items (tx.order.findUnique) — 404 if missing (order.service.ts:132-136)
   c. Inside tx: compares from/to status — 409 ConflictError if from === to (idempotent no-op rejected) (order.service.ts:140-146)
   d. Inside tx: calls canTransition(from, to) from order.status.ts state machine — 409 InvalidStatusTransitionError if not allowed (order.service.ts:147-149)
   e. Inside tx: if shouldDebitStock(from,to) [PENDING->PAID only] calls debitStock(tx, order.items) (order.service.ts:151-153)
   f. Inside tx: if shouldReplenishStock(from,to) [->CANCELLED from PAID/PROCESSING] calls replenishStock(tx, order.items) (order.service.ts:154-156)
   g. Inside tx: updates Order.status (order.service.ts:158)
   h. Inside tx: creates OrderStatusHistory row (order.service.ts:159-167)
   i. Inside tx: re-reads Order with full relations for response (order.service.ts:169-176)
5. Transaction commits; OrderWithRelations returned
6. Controller responds 200 with updated order (order.controller.ts:42)
```

### 2.3 List / Get / Delete

```
list: OrderController.list -> OrderService.list -> OrderRepository.list (order.service.ts:32-42, order.repository.ts:33-45) uses prisma.$transaction([findMany, count]) purely for read consistency, then wraps in paginated() (shared/http/response.ts)
getById: OrderController.getById -> OrderService.getById -> OrderRepository.findByIdWithRelations; 404 if null (order.service.ts:44-48)
delete: OrderController.delete -> OrderService.delete (order.service.ts:181-192) -> OrderRepository.findById (non-transactional) -> status guard (PENDING or CANCELLED only) -> OrderRepository.deleteById (single non-transactional prisma.order.delete call, order.repository.ts:66-68). No stock reversal is performed on delete.
```

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation | Order must contain at least one item (service-level, redundant with Zod) | order.service.ts:51-53 |
| Validation | Duplicate productId line items are aggregated (summed) before pricing | order.service.ts:55, 194-202 |
| Validation | Customer referenced by customerId must exist | order.service.ts:59-60 |
| Validation | All referenced products must exist (count match check) | order.service.ts:62-65 |
| Validation | All referenced products must be active | order.service.ts:66-73 |
| Business Logic | Unit price and line total are computed server-side from Product.priceCents, never trusting client input | order.service.ts:77-84 |
| Business Logic | Subtotal = sum of line totals | order.service.ts:86 |
| Validation | discountCents must not exceed subtotalCents | order.service.ts:87-91 |
| Business Logic | totalCents = subtotalCents - discountCents | order.service.ts:92 |
| Business Logic | Order-number sequence generation via upsert on singleton OrderNumberSequence(id=1) | order.service.ts:93, 245-254 |
| Business Logic | New orders always start in PENDING status with an initial history entry | order.service.ts:99, 106-113 |
| Business Logic | Order status transitions constrained by explicit state machine | order.status.ts:3-14, order.service.ts:147-149 |
| Validation | Self-transition (from === to) rejected as conflict | order.service.ts:140-146 |
| Business Logic | Stock is debited only on PENDING -> PAID transition | order.status.ts:24-31, order.service.ts:151-153 |
| Business Logic | Stock is replenished only on transition to CANCELLED from PAID or PROCESSING | order.status.ts:33-37, order.service.ts:154-156 |
| Validation | Stock debit fails (422 INSUFFICIENT_STOCK) if any product's stockQuantity < requested quantity | order.service.ts:211-224 |
| Business Logic | Stock debit decrements Product.stockQuantity per item via atomic Prisma decrement | order.service.ts:225-230 |
| Business Logic | Stock replenishment increments Product.stockQuantity per item via atomic Prisma increment, with no upper-bound check | order.service.ts:233-243 |
| Business Logic | Every status change is recorded in OrderStatusHistory (from, to, changedBy, reason, timestamp) | order.service.ts:158-167 |
| Validation | Order deletion allowed only when status is PENDING or CANCELLED | order.service.ts:181-192 |
| Business Logic | Order deletion performs no stock reversal even if items were debited (e.g., deleting a CANCELLED order that passed through PAID) | order.service.ts:181-192, order.repository.ts:66-68 |

### Detailed breakdown of the business rules

---

### Business Rule: Order Item Validation and Aggregation

**Overview**:
Before any pricing or persistence occurs, `OrderService.create` enforces that an order contains at least one item and normalizes duplicate product references into a single aggregated line by summing quantities.

**Detailed description**:
The Zod schema `createOrderSchema` already enforces `items: z.array(...).min(1, ...)` (order.schemas.ts:13) at the HTTP boundary, but `OrderService.create` re-checks `input.items.length === 0` and throws a `ValidationError` (order.service.ts:51-53). This is a defense-in-depth pattern: the service does not trust that it is only ever invoked through the HTTP layer with Zod validation applied (for example, if it were called from a script, seed job, or another internal caller, the guard still holds). The check is effectively unreachable through the public REST API because the router validation middleware would reject an empty array first, but it protects the service's public contract.

The `aggregateItems` private method (order.service.ts:194-202) uses a `Map<string, number>` keyed by `productId` to merge quantities for repeated product references in a single request payload — e.g., a client submitting `productId: A, quantity: 2` and `productId: A, quantity: 3` in the same order would be collapsed into a single line item of quantity 5 rather than creating two separate `OrderItem` rows or double-processing stock/price calculations. This directly affects downstream logic: the deduplicated `productIds` list drives the `tx.product.findMany` existence check (order.service.ts:62), so a client cannot artificially inflate the required-product-count validation by repeating IDs, and it prevents the stock debit logic from checking/decrementing the same product's stock twice in unrelated loop iterations (which could otherwise create a race between two decrements referencing stale in-memory quantities within the same request).

The effect on the system is that `Order.items` (the persisted `OrderItem` rows) always have unique `productId` values per order — there is no explicit database constraint enforcing this (no `@@unique([orderId, productId])` in the Prisma schema), so this uniqueness is a business-logic-only guarantee, not a data-layer guarantee. If the aggregation logic were bypassed (e.g., a future direct repository call), duplicate `OrderItem` rows for the same product within one order could exist.

**Rule workflow**:
```
input.items.length === 0? -> throw ValidationError('Order must contain at least one item')
else -> aggregateItems(input.items) collapses same-productId entries by summing quantity
     -> aggregatedItems used for all subsequent product lookups, pricing, and (later) stock operations
```

---

### Business Rule: Customer Existence Validation

**Overview**:
An order cannot be created for a customer that does not exist in the system; this is enforced inside the creation transaction.

**Detailed description**:
Within the `$transaction` block, the very first operation is `tx.customer.findUnique({ where: { id: input.customerId } })` (order.service.ts:59), and if `null` is returned, a `NotFoundError('Customer')` is thrown (order.service.ts:60), which maps to HTTP 404 with error code `NOT_FOUND` (http-errors.ts:27-31). Because this check happens inside the transaction rather than before it opens, the customer lookup benefits from the transaction's read consistency with the rest of the operation, though it also means a customer-existence failure still incurs the overhead of opening (and then rolling back) a database transaction rather than failing fast before any transaction is started.

This rule matters for referential integrity from the business perspective: although the Prisma schema enforces a foreign key (`customer Customer @relation(fields: [customerId], references: [id])`, schema.prisma:87), which would itself reject an insert with a non-existent `customerId` at the database level, the service-level check exists to produce a clean, typed `404 NOT_FOUND` API response with a domain-specific message rather than surfacing a raw Prisma foreign-key-constraint-violation error to the client. This is a common pattern of "pre-flight" validation duplicating what the database would enforce anyway, trading a small amount of redundant work for better API ergonomics and error shaping.

Notably, `OrderService` does not use `CustomerRepository` for this lookup — it calls `tx.customer.findUnique` directly on the transactional Prisma client (order.service.ts:59), bypassing the repository abstraction that `CustomerService` uses elsewhere in the codebase (customer.repository.ts:38-40). This is because `CustomerRepository` is constructed against the top-level `PrismaClient`, not a `Prisma.TransactionClient`, so it cannot participate in `OrderService`'s transaction — the bypass is architecturally forced by the repository's design, not merely a convenience shortcut.

**Rule workflow**:
```
tx.customer.findUnique({ id: input.customerId })
  -> null? throw NotFoundError('Customer') [404 NOT_FOUND]
  -> found -> proceed to product validation
```

---

### Business Rule: Product Existence and Active-Status Validation

**Overview**:
All products referenced in an order must exist and must be marked `active`; inactive or nonexistent products block order creation entirely (all-or-nothing, no partial order creation).

**Detailed description**:
After aggregating line items, `OrderService.create` queries `tx.product.findMany({ where: { id: { in: productIds } } })` (order.service.ts:62) and compares the length of the result set to the length of the requested `productIds` array (order.service.ts:63-65). If any requested product ID does not resolve to a row, the counts mismatch and a `NotFoundError('Product')` is thrown — importantly, this generic error does not identify which specific product ID(s) were not found, unlike the more detailed `InsufficientStockError` used later in the stock-debit path which includes a `details.unavailable` array with SKU-level information.

Beyond existence, every matched product is checked for `active === true`; any inactive product causes the whole request to fail with `UnprocessableEntityError` (422) and error code `INACTIVE_PRODUCT`, including a `details.skus` array listing the offending SKUs (order.service.ts:66-73). This models a common e-commerce/inventory rule: an "active" flag on `Product` (schema.prisma:63, `active Boolean @default(true)`) is likely used to represent discontinued, hidden, or temporarily unsellable catalog items, and this rule prevents orders from being placed against such items even if they still technically exist and have stock. Because this validation, the customer check, and the stock debit check all happen inside the same all-or-nothing database transaction, a single invalid line item in a multi-item order rolls back and rejects the entire order — there is no partial order creation or "split order" behavior.

The check operates purely on `Product.active`, independent of `Product.stockQuantity` — a product can be active with zero stock and still pass this validation stage (stock availability is validated later, only during the PENDING -> PAID transition, not at order-creation time). This is a significant business-logic characteristic: **an order can be created for a product with zero or insufficient stock**, and the stock check is deferred entirely to the payment/stock-debit phase.

**Rule workflow**:
```
tx.product.findMany({ id in productIds })
  -> products.length !== productIds.length? throw NotFoundError('Product') [404]
  -> filter products where active === false
  -> inactive.length > 0? throw UnprocessableEntityError('INACTIVE_PRODUCT', { skus }) [422]
  -> all products exist and active -> proceed to pricing
```

---

### Business Rule: Server-Side Price Calculation (Subtotal, Discount Ceiling, Total)

**Overview**:
Unit prices, line totals, subtotal, and grand total are always computed by the server from the current `Product.priceCents`, never trusting any price data submitted by the client; a discount cannot exceed the computed subtotal.

**Detailed description**:
For each aggregated item, `OrderService.create` looks up the matching `Product` (already fetched in the prior step) and derives `unitPriceCents` directly from `product.priceCents`, then computes `totalCents = unitPriceCents * it.quantity` (order.service.ts:75-84). The client-submitted order payload (`createOrderSchema`, order.schemas.ts:6-9) only contains `productId` and `quantity` — there is no price field accepted from the client at all, which structurally prevents price-tampering at the API contract level (the Zod schema doesn't even parse a price field). This is a strong, well-implemented business rule: prices are a single source of truth from the `Product` catalog at the moment of order creation, and they are snapshotted into `OrderItem.unitPriceCents`/`totalCents` so that subsequent changes to `Product.priceCents` do not retroactively alter historical order totals.

`subtotalCents` is the reduce-sum of all line totals (order.service.ts:86). The discount rule then enforces `input.discountCents <= subtotalCents`; if the client-supplied discount (which defaults to `0` per the Zod schema, order.schemas.ts:14) exceeds the subtotal, a `ValidationError` is thrown with a field-level detail (`path: 'discountCents'`) (order.service.ts:87-91). This prevents negative order totals. `totalCents` is finally `subtotalCents - input.discountCents` (order.service.ts:92) — since discount cannot exceed subtotal, `totalCents` is guaranteed non-negative by construction, though there is no explicit lower-bound assertion on `totalCents` itself; the guarantee is purely derived from the discount-ceiling check.

There is no rule limiting the discount to a percentage, no minimum order value, no per-item discount, and no currency/tax handling visible anywhere in this component — all monetary values are integer cents (`priceCents`, `discountCents`, `totalCents`, `subtotalCents`), which avoids floating-point rounding issues but also means the component does not model any tax computation, shipping cost, or multi-currency support.

**Rule workflow**:
```
for each aggregated item:
  unitPriceCents = product.priceCents (server-side, never client-supplied)
  totalCents = unitPriceCents * quantity
subtotalCents = sum(all itemsToCreate.totalCents)
discountCents > subtotalCents? throw ValidationError('Discount cannot exceed subtotal') [400]
totalCents = subtotalCents - discountCents
```

---

### Business Rule: Order-Number Sequencing

**Overview**:
Every order receives a unique, human-readable, sequential order number in the format `ORD-######`, generated via an atomic upsert against a singleton `OrderNumberSequence` row inside the same transaction as order creation.

**Detailed description**:
`reserveOrderNumber` (order.service.ts:245-254) performs `tx.orderNumberSequence.upsert({ where: { id: 1 }, create: { id: 1, nextValue: 2 }, update: { nextValue: { increment: 1 } }, select: { nextValue: true } })`. On the very first order ever created, the row with `id: 1` does not exist, so the `create` branch fires, setting `nextValue` to `2` directly (skipping `1`), and the method then computes `current = seq.nextValue - 1` which evaluates to `1` — so the very first order number issued is `ORD-000001`. For every subsequent call, the `update` branch increments `nextValue` by 1 atomically via Prisma's `{ increment: 1 }` operator (which translates to a `SQL UPDATE ... SET nextValue = nextValue + 1`, an atomic row-level operation in MySQL/InnoDB), and `current` is again `nextValue - 1`, i.e., the value before this call's increment. The formatted order number is `ORD-` followed by `current` zero-padded to 6 digits (`String(current).padStart(6, '0')`).

Because this upsert executes inside the same Prisma `$transaction` as the rest of order creation (order.service.ts:58, called at line 93), the sequence increment and the eventual `Order` insert (which stores the returned `orderNumber` string, order.service.ts:97) are atomic with respect to each other — if any later step in the transaction fails (e.g., a downstream check I have not found, or a database error during `tx.order.create`), the entire transaction rolls back, including the sequence increment, preventing "gaps" caused by failed order creation attempts (assuming Prisma's default transaction isolation, typically `READ COMMITTED` for MySQL, and that the underlying database uses row-level locking on the `UPDATE` statement to serialize concurrent increments of the same row).

The concurrency safety of this mechanism depends entirely on the database engine's row-locking behavior for the `UPDATE order_number_sequence SET nextValue = nextValue + 1 WHERE id = 1` statement combined with Prisma/MySQL's transaction isolation level — there is no explicit `SELECT ... FOR UPDATE`, no application-level mutex, no optimistic-concurrency retry loop, and no unique-constraint-violation catch-and-retry pattern visible in the code. The `Order.orderNumber` column does have a `@unique` constraint in the Prisma schema (schema.prisma:76), which would surface as a database-level unique-constraint violation if two transactions somehow computed the same `nextValue`, but `OrderService.create` does not catch or retry on this specific error — such a failure would propagate as an unhandled Prisma error rather than a domain-specific, user-friendly error response. This is a notable technical-risk area under the "Technical Debt & Risks" section (concurrent high-throughput order creation could contend heavily on this single-row lock, effectively serializing all order creation transaction-by-transaction).

**Rule workflow**:
```
tx.orderNumberSequence.upsert(id=1):
  not exists -> create { id: 1, nextValue: 2 }; effective sequence value used = 1
  exists -> update { nextValue: increment(1) }; effective sequence value used = (new nextValue - 1)
orderNumber = "ORD-" + zeroPad(effective sequence value, 6)
-- executed inside the same $transaction as Order creation; rolls back together on failure
```

---

### Business Rule: Order Status State Machine

**Overview**:
Order status transitions are constrained to a fixed, directed graph defined in `order.status.ts`; only specific from -> to transitions are legal, and PENDING and CANCELLED/DELIVERED behave as distinguished start/terminal states.

**Detailed description**:
The allowed transitions are declared as a static lookup table (order.status.ts:3-10):
- `PENDING` -> `PAID`, `CANCELLED`
- `PAID` -> `PROCESSING`, `CANCELLED`
- `PROCESSING` -> `SHIPPED`, `CANCELLED`
- `SHIPPED` -> `DELIVERED`
- `DELIVERED` -> (none; terminal)
- `CANCELLED` -> (none; terminal)

`canTransition(from, to)` (order.status.ts:12-14) simply checks `to` is a member of the `from` state's allowed-list array. `isTerminal(status)` (order.status.ts:20-22) is a derived helper (checking if the allowed-list is empty) that is exported but, notably, is not called anywhere inside `order.service.ts` — it appears to be dead code or reserved for future/external use, since `OrderService.changeStatus` relies solely on `canTransition`.

In `OrderService.changeStatus`, the rule is enforced in two layers: first, an explicit self-transition guard (`from === to`) throws a `ConflictError` with code `INVALID_STATUS_TRANSITION` even though semantically this is a distinct case from an actually-illegal transition graph edge (order.service.ts:140-146); second, `canTransition(from, to)` is called and, if false, throws `InvalidStatusTransitionError` (also a 409 with the same `INVALID_STATUS_TRANSITION` code, http-errors.ts:45-53) — meaning both the self-transition case and genuinely invalid edges surface identically to API clients (same HTTP status and error code), so a client cannot distinguish "you tried a no-op" from "you tried an impossible transition" without inspecting the message text or `details.from`/`details.to`.

This state machine directly gates two other business rules — stock debit and stock replenishment — via `shouldDebitStock` and `shouldReplenishStock` predicates (order.status.ts:24-37), meaning the status graph is not just a validity gate but is directly coupled to inventory side effects. Because `SHIPPED` only transitions to `DELIVERED` (no `CANCELLED` option), the system models the business rule that **an order cannot be cancelled once it has shipped** — cancellation is only possible from `PENDING`, `PAID`, or `PROCESSING`. `DELIVERED` and `CANCELLED` are both hard terminal states with zero outgoing transitions, meaning there is no "return/refund after delivery" flow modeled in this state machine at all (despite the analysis brief's mention of "stock replenishment on cancellation/return" — only cancellation-driven replenishment exists in code; no post-delivery return flow exists).

**Rule workflow**:
```
from = order.status; to = input.toStatus
from === to? throw ConflictError('Order is already in {to} status', INVALID_STATUS_TRANSITION) [409]
canTransition(from, to) === false? throw InvalidStatusTransitionError(from, to) [409, INVALID_STATUS_TRANSITION]
else -> allowed; proceed to conditional stock side effects and persist new status + history entry
```

---

### Business Rule: Stock Debit on PENDING -> PAID Transition

**Overview**:
When (and only when) an order transitions from `PENDING` to `PAID`, the system validates sufficient stock for every line item and atomically decrements `Product.stockQuantity`; if any item lacks sufficient stock, the entire transition is rejected and no stock is touched.

**Detailed description**:
`shouldDebitStock` (order.status.ts:29-31) returns true only for the exact `(PENDING, PAID)` pair via the `STOCK_DEBIT_TRANSITION` constant (order.status.ts:24-27) — this is the *only* transition in the entire state machine that triggers a stock decrement. This models the business assumption that stock should be reserved/committed at the moment payment is confirmed, not at order-creation time (when the order is merely `PENDING`) and not at any later stage (`PROCESSING`, `SHIPPED`). This means, as noted in the pricing-rule analysis, that an order can be created and sit in `PENDING` indefinitely referencing more stock than physically exists — the system does not reserve/hold stock for pending orders, creating a potential overselling risk if many pending orders exist simultaneously against limited stock (first-to-pay-wins, not first-to-order-wins).

The `debitStock` private method (order.service.ts:204-231) re-fetches all relevant `Product` rows fresh inside the transaction (`tx.product.findMany`, order.service.ts:208-210) rather than reusing any previously loaded data, ensuring it reads the current `stockQuantity` at the time of the status change (not stale data from order-creation time, which could be arbitrarily long ago). It then performs a two-pass approach: first, a validation pass that builds an `unavailable` array of `{ sku, requested, available }` for every item where the product is missing or `stockQuantity < quantity` (order.service.ts:211-221); if that array is non-empty, it throws `InsufficientStockError(unavailable)` (422, code `INSUFFICIENT_STOCK`) with full per-SKU shortfall detail (order.service.ts:222-224) — this is the most detailed, well-structured error payload in the entire service. Only if *all* items pass validation does the second pass execute, iterating again and calling `tx.product.update` with `{ stockQuantity: { decrement: item.quantity } }` for each item (order.service.ts:225-230), using Prisma's atomic decrement operator (translating to `SET stockQuantity = stockQuantity - N`).

Because the validation pass and the decrement pass are two separate loops (not a single atomic `UPDATE ... WHERE stockQuantity >= N` per item), there is a theoretical window, under a database isolation level weaker than serializable, where a concurrent transaction could alter a product's `stockQuantity` between the validation read and the decrement write. However, since the entire `debitStock` call executes inside the outer `$transaction` from `changeStatus`, this risk is bounded by whatever isolation guarantees the Prisma/MySQL transaction provides (default MySQL `REPEATABLE READ`, which still permits certain write-skew phenomena between separately-issued statements within one transaction unless explicit locking reads, e.g., `SELECT ... FOR UPDATE`, are used — none are used here). This is flagged as a concurrency risk in the Technical Debt section. Additionally, the decrement itself has no floor-check at the database level — Prisma's `decrement` will happily push `stockQuantity` negative if two concurrent transactions both pass the validation read before either commits its decrement (classic TOCTOU / lost-update pattern), since MySQL does not enforce a `CHECK (stockQuantity >= 0)` constraint per the schema (schema.prisma:62 declares only `Int @default(0)`, no check constraint).

**Rule workflow**:
```
shouldDebitStock(from, to) === true only for (PENDING -> PAID)
  -> tx.product.findMany(items' productIds)
  -> for each item: product missing OR stockQuantity < quantity? add to unavailable[]
  -> unavailable.length > 0? throw InsufficientStockError(unavailable) [422, INSUFFICIENT_STOCK] -- whole transaction rolls back, no partial debit
  -> else: for each item -> tx.product.update({ stockQuantity: { decrement: quantity } })
```

---

### Business Rule: Stock Replenishment on Cancellation

**Overview**:
When an order transitions to `CANCELLED` from `PAID` or `PROCESSING` (i.e., states where stock was already debited), the system increments `Product.stockQuantity` back for every order line item, with no validation or upper bound.

**Detailed description**:
`shouldReplenishStock` (order.status.ts:33-37) returns true when `to === CANCELLED` and `from` is either `PAID` or `PROCESSING`. This correctly targets exactly the states reachable after the stock-debit transition (`PENDING` -> `PAID` is the only debit trigger, and `PAID` -> `PROCESSING` is a valid forward transition, so both represent "stock has already been committed" states). Cancelling from `PENDING` does *not* trigger replenishment (order.status.ts:33-37 excludes `PENDING` from the `from` check) — which is business-logic-consistent, since a `PENDING` order never had its stock debited in the first place, so there is nothing to give back; replenishing in that case would incorrectly inflate stock.

The `replenishStock` private method (order.service.ts:233-243) is considerably simpler than `debitStock`: it performs no validation whatsoever, iterating over the order's items and unconditionally calling `tx.product.update({ stockQuantity: { increment: item.quantity } })` for each (order.service.ts:237-242). There is no check that the product still exists (if a product were deleted after being ordered — though the schema has no cascade-delete rule from `Product` to `OrderItem`, and `Product` deletion would likely fail via FK constraint if referenced by any `OrderItem`, since no `onDelete` behavior is specified for `OrderItem.product` relation, defaulting to `RESTRICT` in Prisma/MySQL), no check that incrementing wouldn't restore stock the product should no longer offer (e.g., if a product's `stockQuantity` was manually corrected downward for inventory-audit reasons between the debit and the cancellation, replenishment would still blindly add back the original ordered quantity, potentially overstating true available stock), and no logging/audit trail beyond the generic `OrderStatusHistory` entry for the status change itself (which does not capture the specific stock quantities that were adjusted).

This is a symmetric but not equally-guarded counterpart to `debitStock`: while debit is a two-phase, validated, all-or-nothing operation with detailed error reporting, replenishment is a single-phase, unconditional, best-effort operation. From a business-risk perspective, this asymmetry is understandable (replenishment can't logically "fail" due to insufficient stock, since it's additive), but it does mean there is no safeguard against replenishment being triggered twice for the same order (e.g., if a bug or a race condition allowed the same order to pass through the `PAID -> CANCELLED` or `PROCESSING -> CANCELLED` transition more than once) — although the state machine's terminal-state design (`CANCELLED` has zero outgoing/incoming... actually `CANCELLED` has no *outgoing* transitions and, since `changeStatus` re-reads `order.status` fresh from the database at the top of every call within a new transaction, a second attempt to cancel an already-cancelled order would be caught by the `from === to` guard or by `canTransition` returning false for any `from = CANCELLED` request) — so double-replenishment for the *same* order is prevented by the state machine, but any data-integrity concerns from manual/external stock adjustments are not addressed.

**Rule workflow**:
```
shouldReplenishStock(from, to) === true only for (PAID -> CANCELLED) or (PROCESSING -> CANCELLED)
  -> for each order item: tx.product.update({ stockQuantity: { increment: quantity } })
  -> no existence check, no upper-bound check, no separate audit record of the stock delta
```

---

### Business Rule: Order Status History Audit Trail

**Overview**:
Every status change (including the initial creation) produces an immutable `OrderStatusHistory` record capturing the previous status, new status, the acting user, an optional free-text reason, and a timestamp.

**Detailed description**:
At creation time, the very first history entry is created as a nested write within the same `tx.order.create` call: `fromStatus: null, toStatus: PENDING, changedById: userId, reason: 'order created'` (order.service.ts:106-113) — note `fromStatus` is nullable in the schema (`fromStatus OrderStatus?`, schema.prisma:119) specifically to represent this "no prior state" case for the creation event. Every subsequent transition in `changeStatus` creates an additional `OrderStatusHistory` row via `tx.orderStatusHistory.create` with `fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null` (order.service.ts:159-167), where `reason` is an optional, client-supplied free-text field (max 500 chars per `updateOrderStatusSchema`, order.schemas.ts:20) with no further validation or enumeration (i.e., it is not a controlled vocabulary — clients can supply arbitrary free text or omit it).

This history is always created within the same transaction as the status update itself (both `create` and `changeStatus` wrap the history-row insert inside their respective `$transaction` blocks), guaranteeing that `Order.status` and its corresponding `OrderStatusHistory` trail can never diverge — there is no scenario in the code where a status update commits without a corresponding history record, or vice versa, because both writes are part of the same atomic unit of work. The history is surfaced to API clients on every read of an order via the `include: { history: { orderBy: { changedAt: 'asc' } } }` clause (order.service.ts:117, 173; order.repository.ts:56), giving full chronological visibility into an order's lifecycle in every `OrderWithRelations` response.

From a business/compliance perspective, this audit trail is one of the more robust aspects of the component: it provides traceability of who changed an order's status and why (when a reason is supplied), which supports operational accountability (e.g., disputes about why an order was cancelled). However, the history only tracks status transitions — it does not capture other order mutations (e.g., there is no update-order-items or update-discount operation in this service at all; `Order` fields other than `status` are immutable post-creation based on the code present), and it does not track the specific stock quantity deltas applied during debit/replenishment (that information is derivable only indirectly by cross-referencing `OrderItem.quantity` with the transition type, not stored explicitly per history event).

**Rule workflow**:
```
On create: tx.order.create(... nested history: { create: { fromStatus: null, toStatus: PENDING, changedById: userId, reason: 'order created' } } )
On changeStatus: tx.orderStatusHistory.create({ orderId, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null })
-- Always executed within the same transaction as the corresponding order/status write; always ordered ascending by changedAt on read
```

---

### Business Rule: Order Deletion Constraints

**Overview**:
An order may only be permanently deleted while it is in `PENDING` or `CANCELLED` status; deletion is a hard database delete with no stock reversal logic and no soft-delete/archival mechanism.

**Detailed description**:
`OrderService.delete` (order.service.ts:181-192) first loads the order via `this.orders.findById(id)` — note this uses `OrderRepository.findById` (order.repository.ts:62-64), a plain non-transactional `prisma.order.findUnique`, unlike `create` and `changeStatus` which operate inside `$transaction` blocks against the raw `PrismaClient` field. If the order does not exist, `NotFoundError('Order')` is thrown (404). The status guard then only permits deletion when `order.status === PENDING || order.status === CANCELLED` (order.service.ts:184-190); any other status (`PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`) causes a `ConflictError` (409, code `INVALID_ORDER_STATE_FOR_DELETE`) including the current status in `details` (order.service.ts:185-189).

This rule's rationale is presumably that `PENDING` orders have no side effects yet (no stock committed, no payment processed) and are therefore safe to discard entirely, while `CANCELLED` orders represent a terminal, already-resolved state that an operator might want to purge for cleanup. However, this reveals a significant gap: an order that reached `PAID` or `PROCESSING` and was then cancelled (triggering stock replenishment, per the prior rule) is now in `CANCELLED` status and is therefore eligible for deletion — deleting it removes the `Order` row (and, via `onDelete: Cascade` on `OrderItem.order` and `OrderStatusHistory.order` relations, schema.prisma:108 and 125, cascades to delete all associated `OrderItem` and `OrderStatusHistory` rows too) — permanently erasing the audit trail of what was ordered, its pricing, and its full status history, with no archival step. This is in tension with the otherwise careful audit-trail design of the status-history rule above: the history exists to provide traceability, but that traceability can be fully erased via the delete path for any `CANCELLED` order.

Critically, `OrderService.delete` performs **no stock reversal check** of its own — it relies entirely on the fact that stock replenishment already happened during the `PAID/PROCESSING -> CANCELLED` transition (if applicable) before deletion is attempted; deleting a `PENDING` order (which never had stock debited) is safe with respect to stock, and deleting a `CANCELLED` order (which, if it passed through `PAID`/`PROCESSING`, already had stock replenished during that transition) is also safe — so the design is internally consistent, but this consistency is implicit and relies on the invariant that replenishment always correctly and completely reverses every debit, which (per the stock-replenishment rule analysis above) has no independent verification.

**Rule workflow**:
```
orders.findById(id) -> null? throw NotFoundError('Order') [404]
order.status !== PENDING && order.status !== CANCELLED? throw ConflictError('...', INVALID_ORDER_STATE_FOR_DELETE, { status }) [409]
else -> orders.deleteById(id) -- hard delete, cascades to OrderItem and OrderStatusHistory rows, non-transactional single Prisma call
```

---

## 4. Component Structure

```
src/modules/orders/                        # Order bounded-context module
├── order.controller.ts                    # Express request handlers; auth/user extraction; delegates to OrderService; maps results to HTTP status codes
├── order.repository.ts                    # Data-access layer for simple (non-transactional) reads/deletes: list (with count), findByIdWithRelations, findById, deleteById
├── order.routes.ts                        # Express Router wiring: authenticate middleware + Zod validate middleware + controller method bindings per route
├── order.schemas.ts                       # Zod schemas/types: createOrderSchema, updateOrderStatusSchema, listOrdersQuerySchema, orderIdParamSchema
├── order.service.ts                       # CORE business logic: create, changeStatus, list, getById, delete + private helpers (aggregateItems, debitStock, replenishStock, reserveOrderNumber)
└── order.status.ts                        # Pure state-machine module: transitions table, canTransition, allowedTransitions, isTerminal, shouldDebitStock, shouldReplenishStock

src/modules/customers/                      # Related module (referenced by OrderService via raw tx.customer calls, NOT via CustomerRepository)
├── customer.repository.ts                 # CustomerRepository — NOT used by OrderService (bypassed)
└── customer.service.ts                    # Independent customer CRUD service (not invoked by OrderService)

src/modules/products/                       # Related module (referenced by OrderService via raw tx.product calls, NOT via ProductRepository)
├── product.repository.ts                  # ProductRepository — NOT used by OrderService (bypassed)
└── product.service.ts                     # Independent product CRUD service (not invoked by OrderService)

src/shared/errors/
├── app-error.ts                           # Base AppError class (statusCode, errorCode, details)
├── http-errors.ts                         # BadRequestError, ValidationError, UnauthorizedError, ForbiddenError, NotFoundError, ConflictError, UnprocessableEntityError, InvalidStatusTransitionError, InsufficientStockError
└── index.ts                               # Barrel export, imported by order.service.ts:9-16

src/shared/http/response.ts                 # paginated()/buildPagination() helpers used by OrderService.list (order.service.ts:22, 41)

src/app.ts                                  # DI wiring: instantiates OrderRepository, OrderService, OrderController (app.ts:42-44)
src/routes/index.ts                         # Mounts buildOrderRouter(controllers.orders) at '/orders' under the '/api/v1' prefix (routes/index.ts:28; app.ts:67)

prisma/schema.prisma                        # Data model: Order, OrderItem, OrderStatusHistory, OrderNumberSequence, Product, Customer, User, OrderStatus enum

tests/orders.test.ts                        # Integration/E2E tests exercising the full HTTP stack against a real (test) database
tests/helpers/factories.ts                  # Test factories: createTestCustomer, createTestProduct, bootstrapAuthenticatedUser, getTestApp
```

## 5. Dependency Analysis

### Internal Dependency Chains

```
order.routes.ts → order.controller.ts → order.service.ts → order.repository.ts (list/getById/delete existence check/deleteById)
                                                          → PrismaClient (direct, via $transaction) for create() and changeStatus()
                                                             → tx.customer.*      (bypasses CustomerRepository)
                                                             → tx.product.*      (bypasses ProductRepository)
                                                             → tx.order.*
                                                             → tx.orderStatusHistory.*
                                                             → tx.orderNumberSequence.*
order.service.ts → order.status.ts (canTransition, shouldDebitStock, shouldReplenishStock)
order.service.ts → shared/errors/index.ts (ConflictError, InsufficientStockError, InvalidStatusTransitionError, NotFoundError, UnprocessableEntityError, ValidationError)
order.service.ts → shared/http/response.ts (paginated, PaginatedResponse)
order.controller.ts → shared/errors/index.ts (UnauthorizedError)
order.routes.ts → middlewares/auth.middleware.ts (authenticate)
order.routes.ts → middlewares/validate.middleware.ts (validate)
order.routes.ts → order.schemas.ts (createOrderSchema, listOrdersQuerySchema, orderIdParamSchema, updateOrderStatusSchema)
app.ts → order.repository.ts, order.service.ts, order.controller.ts (constructs and wires the module's DI graph)
routes/index.ts → order.routes.ts (buildOrderRouter) → mounted at /orders under /api/v1
```

### External Dependencies (from package.json)

| Package | Version | Purpose in OrderService context |
|---------|---------|----------------------------------|
| @prisma/client | 5.22.0 | ORM client; PrismaClient, Prisma.TransactionClient, OrderStatus enum, all model types used throughout order.service.ts, order.repository.ts |
| express | 4.21.1 | RequestHandler type, Router — order.controller.ts, order.routes.ts |
| zod | 3.23.8 | Schema validation for create/update/list DTOs — order.schemas.ts |
| pino / pino-http | 9.5.0 / 10.3.0 | Application-wide request logging (not directly referenced inside order.service.ts, but active on the request path via app.ts middleware) |
| jsonwebtoken | 9.0.2 | Used by auth.middleware.ts (authenticate) which gates all order routes (order.routes.ts:14) — not directly imported by OrderService |
| vitest | 2.1.4 (dev) | Test runner for tests/orders.test.ts |
| supertest | 7.0.0 (dev) | HTTP assertions used in tests/orders.test.ts |
| prisma (CLI) | 5.22.0 (dev) | Schema/migration tooling for prisma/schema.prisma |

No direct usage of `uuid` or `bcrypt` was found inside `order.service.ts`, `order.repository.ts`, `order.controller.ts`, `order.routes.ts`, `order.schemas.ts`, or `order.status.ts` (uuids for Order/OrderItem/OrderStatusHistory primary keys are generated by Prisma's `@default(uuid())` at the schema level, schema.prisma:75, 100, 117 — not application code).

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling (who depends on it) | Efferent Coupling (what it depends on) | Critical |
|-----------|----------------------------------------|------------------------------------------|----------|
| OrderService | OrderController (order.controller.ts:2,7), app.ts (app.ts:17,43) — 2 | OrderRepository, PrismaClient, order.status.ts (3 functions), shared/errors (6 error classes), shared/http/response.ts (2 exports) — 6 distinct module dependencies | High |
| OrderRepository | OrderService (order.service.ts:7,28) — 1 | PrismaClient only — 1 | Medium |
| OrderController | order.routes.ts (order.routes.ts:10,12), app.ts (app.ts:18,44), routes/index.ts (routes/index.ts:6) — 3 | OrderService, shared/errors (UnauthorizedError) — 2 | Low |
| order.status.ts (state machine functions) | OrderService (order.service.ts:17-21, 5 call sites) — 1 module consumer | @prisma/client (OrderStatus enum) only — 1 | High |
| order.schemas.ts | order.routes.ts (4 schema imports), OrderService (2 type-only imports) — 2 | zod, @prisma/client (OrderStatus) — 2 | Medium |
| Product (via tx.product.*, bypassing ProductRepository) | OrderService.create, OrderService.debitStock, OrderService.replenishStock (3 internal call sites) — accessed directly, not via ProductRepository | N/A (Prisma model) | High |
| Customer (via tx.customer.*, bypassing CustomerRepository) | OrderService.create (1 call site) | N/A (Prisma model) | Medium |
| OrderNumberSequence (via tx.orderNumberSequence.*) | OrderService.reserveOrderNumber (1 call site) — sole consumer in entire codebase | N/A (Prisma model) | High (single point of write contention) |

Note: Afferent/efferent counts are derived from static import and call-site analysis within the codebase scope examined (src/ and tests/); "Critical" reflects a qualitative judgment based on blast radius if the component's behavior changes or fails (OrderService and the state machine it depends on are rated High because nearly all order-related business behavior — pricing, stock, sequencing, transitions — routes through them).

## 7. Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/orders | GET | List orders with pagination and optional filters (status, customerId, from, to) — order.routes.ts:16, OrderController.list |
| /api/v1/orders/:id | GET | Retrieve a single order by ID with full relations (items, history, customer) — order.routes.ts:17, OrderController.getById |
| /api/v1/orders | POST | Create a new order (PENDING status, server-computed pricing, auto-generated order number) — order.routes.ts:18, OrderController.create |
| /api/v1/orders/:id/status | PATCH | Transition an order's status (triggers conditional stock debit/replenishment) — order.routes.ts:19-23, OrderController.changeStatus |
| /api/v1/orders/:id | DELETE | Delete an order (only if PENDING or CANCELLED) — order.routes.ts:24, OrderController.delete |

All routes require authentication via the `authenticate` middleware applied at the router level (`router.use(authenticate)`, order.routes.ts:14); no route-level role/authorization checks (e.g., ADMIN vs OPERATOR) were found within the orders module itself — `req.user` presence is checked in the controller (order.controller.ts:30, 40) but not `req.user.role`.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL (via Prisma) | Database | Persistence for Order, OrderItem, OrderStatusHistory, OrderNumberSequence, and direct reads/writes of Customer and Product | Prisma Client / MySQL wire protocol | SQL (generated by Prisma) | Domain errors thrown explicitly (NotFoundError, ConflictError, etc.); no explicit catch/retry around raw Prisma exceptions (e.g., unique constraint violations) inside order.service.ts — such errors would propagate to the global errorMiddleware (app.ts:73) |
| OrderNumberSequence table | Internal DB sequence resource | Guarantees unique, monotonically increasing order numbers via upsert-based counter | Prisma Client (`tx.orderNumberSequence.upsert`) | Single-row counter table (id=1, nextValue Int) | No explicit handling of unique-constraint violation on Order.orderNumber if sequence contention were to somehow produce a duplicate; relies purely on DB transaction/row-lock semantics |
| Product.stockQuantity (direct write) | Internal DB resource (bypasses ProductRepository) | Stock debit on PENDING->PAID, stock replenishment on ->CANCELLED | Prisma Client (`tx.product.update` with decrement/increment) | Integer counter field | Debit path validates and throws InsufficientStockError before any write (all-or-nothing); replenish path has no validation/error handling |
| Customer (direct read) | Internal DB resource (bypasses CustomerRepository) | Existence validation for order creation | Prisma Client (`tx.customer.findUnique`) | N/A | NotFoundError thrown on null result |
| auth.middleware.ts (JWT) | Internal middleware | Authenticates requests and populates req.user before reaching OrderController | HTTP header (Authorization: Bearer) / JWT | JWT token | UnauthorizedError thrown by controller if req.user is absent post-middleware (defensive double-check) |
| validate.middleware.ts (Zod) | Internal middleware | Validates request body/params/query against order.schemas.ts before controller invocation | HTTP request body/query/params | JSON | Presumed to short-circuit with a 400-level response on schema validation failure (validate.middleware.ts not read in this analysis pass, referenced only via order.routes.ts:3,16-24) |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Repository Pattern | OrderRepository, ProductRepository, CustomerRepository | order.repository.ts:18-69, product.repository.ts:10-62, customer.repository.ts:9-57 | Abstracts data access for simple CRUD/list/find operations |
| Dependency Injection (constructor injection, manual/hand-wired) | OrderService constructor takes OrderRepository + PrismaClient; wiring done in app.ts | order.service.ts:27-30; app.ts:42-44 | Decouples service construction from usage; enables test doubles in principle (not exercised by current tests, which use real Prisma) |
| DTO Pattern (via Zod-inferred types) | CreateOrderInput, UpdateOrderStatusInput, ListOrdersQuery | order.schemas.ts:32-34 | Typed, validated request payload contracts shared between routing and service layers |
| Unit of Work (via Prisma $transaction) | this.prisma.$transaction(async (tx) => {...}) in create() and changeStatus() | order.service.ts:58, 131 | Ensures atomicity across multi-table writes (Order, OrderItem, OrderStatusHistory, Product, OrderNumberSequence) within a single request |
| State Machine Pattern | transitions lookup table + canTransition/shouldDebitStock/shouldReplenishStock | order.status.ts:3-37 | Centralizes and encodes valid order lifecycle transitions and their side-effect triggers, separate from the service orchestration logic |
| Custom Exception Hierarchy | AppError base class with typed subclasses (ValidationError, NotFoundError, ConflictError, UnprocessableEntityError, InvalidStatusTransitionError extends ConflictError, InsufficientStockError extends UnprocessableEntityError) | shared/errors/app-error.ts:3-16, shared/errors/http-errors.ts:1-63 | Maps domain failures to consistent HTTP status codes and machine-readable error codes for API consumers |
| Aggregation/Normalization helper | aggregateItems (Map-based dedup) | order.service.ts:194-202 | Prevents duplicate-productId line items from causing inconsistent pricing/stock logic |
| Anemic Repository Bypass (anti-pattern, noted for completeness) | Direct tx.customer.*, tx.product.*, tx.orderNumberSequence.* calls instead of CustomerRepository/ProductRepository | order.service.ts:59, 62, 208, 226, 238, 246 | Necessitated by the repositories being bound to the top-level PrismaClient rather than accepting a Prisma.TransactionClient, forcing OrderService to talk to Prisma directly whenever transactional consistency across entities is required |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | reserveOrderNumber (order.service.ts:245-254) | Order-number sequencing relies solely on implicit DB transaction/row-lock behavior of a single-row upsert-increment; no explicit `SELECT ... FOR UPDATE`, no retry-on-unique-violation, no documented isolation level requirement | Under concurrent order creation, this single row becomes a serialization bottleneck (throughput ceiling), and if the underlying isolation level is ever weakened or the database swapped, duplicate/out-of-order order numbers become possible with no defensive catch |
| High | debitStock (order.service.ts:204-231) | Two-phase check-then-write (validate all items, then loop updates) is not a single atomic conditional update (e.g., `UPDATE products SET stockQuantity = stockQuantity - N WHERE id = ? AND stockQuantity >= N`); no `SELECT ... FOR UPDATE` locking read | Classic TOCTOU race: concurrent PENDING->PAID transitions on orders sharing a product could both pass validation before either commits its decrement, allowing stockQuantity to go negative (no DB-level CHECK constraint exists per schema.prisma:62) |
| High | OrderService architecture (order.service.ts:27-30, 59, 62, 208, 226, 238, 246) | Bypasses CustomerRepository and ProductRepository entirely, coupling OrderService directly to Prisma's raw API and to the internal shape of Customer/Product/OrderNumberSequence tables | Any future change to how Customer/Product persistence is abstracted (e.g., adding caching, soft-delete filters, or swapping the repository implementation) would silently NOT apply to OrderService's direct tx.* calls, creating divergent behavior between modules |
| Medium | Order deletion (order.service.ts:181-192, order.repository.ts:66-68) | Hard delete with onDelete: Cascade on OrderItem/OrderStatusHistory (schema.prisma:108, 125); permanently destroys the audit trail for any CANCELLED order, including ones that previously debited and replenished stock | Loss of historical/audit data for compliance or dispute-resolution purposes; no soft-delete or archival mechanism |
| Medium | replenishStock (order.service.ts:233-243) | No existence validation, no upper-bound check, unconditional increment | If a product is deleted, disabled, or has had its stock manually corrected between debit and cancellation, replenishment blindly restores the originally-debited quantity, potentially overstating true available stock |
| Medium | changeStatus self-transition and invalid-transition errors (order.service.ts:140-149) | Both cases surface as the same HTTP 409 / INVALID_STATUS_TRANSITION code, differing only in message text | API consumers cannot programmatically distinguish "no-op transition attempted" from "structurally invalid transition" without string-matching the message |
| Medium | Generic NotFoundError('Product') (order.service.ts:64) | Unlike InsufficientStockError, does not report which specific product IDs were missing | Harder for API clients/operators to diagnose which line item(s) caused the failure, especially in multi-item orders |
| Medium | Route-level authorization (order.routes.ts:14; order.controller.ts:30,40) | Only checks req.user presence (authentication), no req.user.role check for potentially sensitive operations like delete or status changes to PAID/CANCELLED | Any authenticated user (regardless of role, e.g., OPERATOR vs ADMIN) can delete orders or force arbitrary valid status transitions; no separation of duties enforced at this layer |
| Medium | isTerminal() (order.status.ts:20-22) | Exported but never called anywhere in order.service.ts or elsewhere found in the codebase | Dead/unused code; may indicate an intended guard (e.g., preventing operations on terminal-state orders elsewhere) that was never wired up |
| Low | Hardcoded padding/format (order.service.ts:253) | Order number format ORD-###### (6-digit zero-padded) is a hardcoded magic format with no configuration; will silently overflow formatting expectations (though not correctness) past 999,999 orders, producing e.g. ORD-1000000 (7 digits) | Downstream systems/reports that assume a fixed-width order number format could break once volume exceeds 999,999 orders |
| Low | Duplicate validation (order.service.ts:51-53 vs order.schemas.ts:13) | items.length===0 check is redundant given Zod's .min(1) already applied by validate middleware upstream | Minor redundancy; not a functional bug, but indicates the service does not fully trust its own schema layer, which is defensible but adds maintenance surface (two places to keep in sync) |
| Low | No idempotency key on order creation (order.service.ts:50-124, order.schemas.ts:11-16) | No client-supplied idempotency token; a network retry of a POST /orders request could create duplicate orders for the same logical intent | Risk of duplicate order creation under client-side retry logic or double-submission (e.g., double-click), each consuming a distinct order number and potentially debiting stock twice once paid |

## 11. Test Coverage Analysis

| Rule / Branch | Covered by tests/orders.test.ts | Test Quality Notes |
|---------------|----------------------------------|---------------------|
| Order creation happy path (multi-item, discount, totals, order number format) | Yes — "creates an order in PENDING status and computes totals on the server" (orders.test.ts:12-40) | Strong: asserts subtotal/discount/total math, item count, initial history entry, and orderNumber regex format |
| Product not found on create | Yes — "rejects an order with non-existent product" (orders.test.ts:42-57) | Asserts 404 + NOT_FOUND code; does not assert response body detail content |
| Customer not found on create | No dedicated test found | Gap: no test submits a nonexistent customerId |
| Inactive product rejection (422 INACTIVE_PRODUCT) | No test found | Gap: createTestProduct factory defaults active: true and no test overrides to active: false for order creation |
| Discount exceeds subtotal (400 ValidationError) | No test found | Gap: no test submits discountCents > subtotal |
| Duplicate productId aggregation (aggregateItems) | No test found | Gap: no test submits the same productId twice in one order's items array |
| Empty items array rejection | Indirectly implied by Zod schema but no explicit service/API-level test asserting 400 on empty items[] | Gap: no direct test |
| PENDING -> PAID transition + stock debit | Yes — "transitions PENDING -> PAID and decrements stock" (orders.test.ts:59-87) | Strong: asserts status, history sequence ['PENDING','PAID'], and exact stockQuantity decrement value |
| Invalid transition PENDING -> SHIPPED (409) | Yes — "returns 409 on invalid status transition" (orders.test.ts:89-107) | Good: asserts status code and error code |
| Self-transition (from === to) 409 | No test found | Gap: no test transitions an order to its current status |
| Insufficient stock on PENDING -> PAID (422 INSUFFICIENT_STOCK) | Yes — "returns 422 when transitioning PENDING -> PAID without enough stock" (orders.test.ts:109-132) | Strong: asserts error code, details.unavailable[0].sku, and confirms stockQuantity unchanged (no partial debit) |
| PAID -> CANCELLED replenishment | Yes — "replenishes stock when going PAID -> CANCELLED" (orders.test.ts:134-161) | Strong: asserts stock decrement after PAID and full restoration after CANCELLED |
| PROCESSING -> CANCELLED replenishment | No test found | Gap: only PAID -> CANCELLED path is tested; PROCESSING -> CANCELLED (also covered by shouldReplenishStock) is untested |
| PAID -> PROCESSING, PROCESSING -> SHIPPED, SHIPPED -> DELIVERED transitions | No test found | Gap: only PENDING->PAID and ->CANCELLED paths are exercised; the rest of the forward lifecycle graph is untested |
| PENDING -> CANCELLED (no replenishment expected) | No test found | Gap: not explicitly asserted that stock is untouched when cancelling directly from PENDING |
| List orders with status filter and pagination | Yes — "lists orders filtered by status" (orders.test.ts:163-195) | Good: asserts pagination.total and filtered data length for two different status filters |
| List orders filtered by customerId/from/to | No test found | Gap: OrderRepository.buildWhere supports these filters but no test exercises them |
| getById (200 and 404) | No direct test found in orders.test.ts | Gap: GET /orders/:id is not exercised at all in the provided test file |
| Delete order in PENDING/CANCELLED (success path) | No explicit success-path delete test found (only the failure path is tested) | Gap: DELETE success (204) is untested; only the 409 rejection is covered |
| Delete order in invalid state (409 INVALID_ORDER_STATE_FOR_DELETE) | Yes — "refuses to delete an order that is not PENDING or CANCELLED" (orders.test.ts:197-218) | Good: asserts status code and error code |
| Order-number sequence uniqueness/monotonicity under concurrency | No test found | Gap: no concurrent/parallel order-creation test exists to validate the sequencing mechanism under load |
| Concurrent stock debit race condition | No test found | Gap: no test simulates two simultaneous PENDING->PAID transitions against the same product's limited stock |
| Authorization/role-based access (ADMIN vs OPERATOR) on order mutations | No test found in orders.test.ts | Gap: bootstrapAuthenticatedUser defaults to OPERATOR and no test varies role for order endpoints |

**Overall Test Coverage Assessment**: `tests/orders.test.ts` contains 8 integration tests, all executed as full HTTP-stack (supertest) tests against a real test database via `tests/helpers/factories.ts` — no unit tests isolate `OrderService`, `order.status.ts`, or private methods (`aggregateItems`, `debitStock`, `replenishStock`, `reserveOrderNumber`) independently of the HTTP/DB layer. The existing tests are of good quality where present (they assert precise numeric values, error codes, and side-effect state such as post-transition stock quantities), but coverage is materially incomplete for: customer-not-found, inactive-product, discount-exceeds-subtotal, duplicate-item aggregation, self-transition conflict, the PROCESSING->CANCELLED replenishment branch, the PAID->PROCESSING->SHIPPED->DELIVERED forward-progression branches, getById, successful deletion, and — most significantly given this component's flagged risk profile — any concurrency/race-condition scenario for stock debit or order-number sequencing. No test file other than `tests/orders.test.ts` references order-related functionality (confirmed via project-wide search for "order" in file names and no `.spec.ts` files exist).
