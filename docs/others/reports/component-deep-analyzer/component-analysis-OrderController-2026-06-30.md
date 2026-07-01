# Component Deep Analysis Report: OrderController

**Project:** order-management-api (Order Management System REST API)
**Component analyzed:** `OrderController`
**Primary file:** `src/modules/orders/order.controller.ts`
**Analysis date:** 2026-06-30
**Analysis mode:** Read-only (no files modified)

---

## 1. Executive Summary

`OrderController` is the HTTP entry point of the Orders module, an Express-based REST resource responsible for the full lifecycle of customer orders in the Order Management System: listing, retrieving, creating, transitioning status, and deleting orders. The controller itself (`src/modules/orders/order.controller.ts`, 56 lines) is intentionally thin — it contains no business logic. Its sole responsibilities are:

1. Extracting request data (query params, route params, body, authenticated user) from the Express `Request` object.
2. Performing a single authorization guard (`req.user` presence check) on mutating endpoints (`create`, `changeStatus`).
3. Delegating all domain work to `OrderService`.
4. Mapping successful service results to HTTP status codes (`200`, `201`, `204`) and JSON responses.
5. Forwarding any thrown error to Express's `next(err)` for centralized handling by `errorMiddleware`.

The real complexity of the "Orders" capability lives one layer down, in `OrderService` (`src/modules/orders/order.service.ts`), which owns transactional order creation, status-transition validation, stock debit/replenishment, order numbering, and referential integrity checks against `Customer` and `Product`. `OrderRepository` (`src/modules/orders/order.repository.ts`) is a thin Prisma data-access layer used only for list/read/delete operations — notably, both `create` and `changeStatus` in `OrderService` bypass the repository and use `this.prisma` directly to run multi-statement Prisma transactions (`$transaction`), which is an architectural inconsistency worth noting (see Technical Debt section).

Key findings:
- The controller exposes 5 REST endpoints, all mounted under `/api/v1/orders` and all protected by a router-level `authenticate` middleware (JWT bearer token).
- Authorization in `create` and `changeStatus` is redundant with the router-level `authenticate` middleware — by the time these handlers execute, `authenticate` middleware would already have called `next(new UnauthorizedError())` for missing/invalid tokens, meaning `req.user` should always be populated. The extra `if (!req.user) throw new UnauthorizedError()` checks are defensive/dead-code-like given the current router wiring, unless `OrderController` is reused on a route lacking `authenticate`.
- There is no role-based authorization (`requireRole`) applied to any order endpoint — both `ADMIN` and `OPERATOR` roles can create, change status, and delete orders.
- Order creation is fully transactional (Prisma `$transaction`) and enforces referential integrity (customer must exist, all products must exist and be active), item de-duplication/aggregation, discount validation, and server-computed pricing (client cannot supply prices).
- Status transitions are governed by an explicit finite-state machine (`order.status.ts`) with side-effecting inventory operations (stock debit on `PENDING→PAID`, stock replenishment on `PAID/PROCESSING→CANCELLED`).
- Deletion is restricted to orders in `PENDING` or `CANCELLED` status, preventing accidental removal of orders that have progressed through fulfillment.
- Test coverage for the module is solid at the integration/HTTP level (`tests/orders.test.ts`, 8 tests covering create, transitions, stock effects, listing/filtering, and delete guard) but there are no isolated unit tests for `OrderController`, `OrderService`, `OrderRepository`, or `order.status.ts` — all coverage is via black-box HTTP tests through `supertest`.

---

## 2. Data Flow Analysis

### 2.1 `POST /api/v1/orders` (create)

```
1. Request enters Express app (src/app.ts) → /api/v1 → /orders router (order.routes.ts)
2. authenticate middleware (src/middlewares/auth.middleware.ts) verifies JWT bearer token,
   populates req.user = { id, email, role } or calls next(UnauthorizedError)
3. validate({ body: createOrderSchema }) middleware (validate.middleware.ts) parses/coerces
   req.body via Zod; on failure calls next(ValidationError) with per-field details
4. OrderController.create handler:
     - Defensive check: if (!req.user) throw UnauthorizedError
     - Calls OrderService.create(req.body, req.user.id)
5. OrderService.create:
     a. Rejects empty items array (ValidationError) [redundant with Zod .min(1)]
     b. Aggregates duplicate productId line items into summed quantities (aggregateItems)
     c. Opens a Prisma transaction ($transaction):
        i.   Loads Customer by id; 404 NotFoundError('Customer') if missing
        ii.  Loads all referenced Products by id; 404 NotFoundError('Product') if any missing
        iii. Rejects if any product is inactive → 422 UnprocessableEntityError('INACTIVE_PRODUCT')
        iv.  Computes unitPriceCents/totalCents per item from server-side Product.priceCents
             (client-submitted price data is never trusted)
        v.   Computes subtotalCents = sum(item totals)
        vi.  Validates discountCents <= subtotalCents → 400 ValidationError otherwise
        vii. Computes totalCents = subtotalCents - discountCents
        viii.Reserves a sequential order number via OrderNumberSequence upsert (reserveOrderNumber)
        ix.  Creates Order row with nested OrderItem[] and initial OrderStatusHistory
             (fromStatus: null, toStatus: PENDING, reason: 'order created')
        x.   Returns order with items/history/customer relations included
6. OrderController serializes result as HTTP 201 with the full OrderWithRelations JSON body
7. On any thrown error at any step, next(err) routes to errorMiddleware (error.middleware.ts)
   which maps AppError subclasses to their statusCode/errorCode, ZodError to 400, and
   Prisma known errors (P2002/P2025) to 409/404, else 500 with logged stack trace
```

### 2.2 `GET /api/v1/orders` (list)

```
1. authenticate middleware validates JWT
2. validate({ query: listOrdersQuerySchema }) coerces page/pageSize/status/customerId/from/to
3. OrderController.list casts req.query to ListOrdersQuery and calls OrderService.list(query)
4. OrderService.list computes skip/take from page/pageSize, delegates to OrderRepository.list
5. OrderRepository.list builds a Prisma where clause (status, customerId, createdAt range)
   and runs a $transaction of [findMany, count] for consistent pagination totals
6. OrderService wraps results via paginated() (shared/http/response.ts) into
   { data: Order[], pagination: { page, pageSize, total, totalPages } }
7. OrderController responds 200 with the paginated envelope
```

### 2.3 `GET /api/v1/orders/:id` (getById)

```
1. authenticate → validate({ params: orderIdParamSchema }) (UUID format check)
2. OrderController.getById calls OrderService.getById(req.params.id)
3. OrderService.getById → OrderRepository.findByIdWithRelations (Prisma findUnique with
   items/product/history/customer includes); throws 404 NotFoundError('Order') if null
4. OrderController responds 200 with the full order representation
```

### 2.4 `PATCH /api/v1/orders/:id/status` (changeStatus)

```
1. authenticate → validate({ params: orderIdParamSchema, body: updateOrderStatusSchema })
2. OrderController.changeStatus: defensive req.user check, calls
   OrderService.changeStatus(id, body, userId)
3. OrderService.changeStatus opens a Prisma transaction:
     a. Loads order with items; 404 NotFoundError('Order') if missing
     b. from = order.status, to = input.toStatus
     c. If from === to → 409 ConflictError('INVALID_STATUS_TRANSITION') (no-op transition)
     d. If !canTransition(from, to) (order.status.ts state machine) → 409
        InvalidStatusTransitionError
     e. If shouldDebitStock(from, to) [PENDING→PAID] → debitStock():
          - Reloads referenced Products, checks stockQuantity >= requested quantity per item
          - If any insufficient → 422 InsufficientStockError with per-SKU shortfall details
            (transaction rolls back, no partial decrement)
          - Otherwise decrements Product.stockQuantity per item
     f. If shouldReplenishStock(from, to) [PAID/PROCESSING→CANCELLED] → replenishStock():
          - Increments Product.stockQuantity per item (no bound/limit check)
     g. Updates Order.status = to
     h. Inserts OrderStatusHistory row (fromStatus, toStatus, changedById, reason)
     i. Re-fetches and returns the order with items/history/customer relations
4. OrderController responds 200 with the updated order
```

### 2.5 `DELETE /api/v1/orders/:id` (delete)

```
1. authenticate → validate({ params: orderIdParamSchema })
2. OrderController.delete calls OrderService.delete(id) (no user id passed/used)
3. OrderService.delete:
     a. OrderRepository.findById; 404 NotFoundError('Order') if missing
     b. Guard: status must be PENDING or CANCELLED, else 409 ConflictError
        ('INVALID_ORDER_STATE_FOR_DELETE')
     c. OrderRepository.deleteById (hard delete; OrderItem/OrderStatusHistory cascade
        via Prisma onDelete: Cascade at the DB/schema level)
4. OrderController responds 204 No Content
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authentication | All order endpoints require a valid JWT bearer token | order.routes.ts:14, auth.middleware.ts:27 |
| Authorization | `create` and `changeStatus` additionally require `req.user` to be set (defensive, redundant with router-level auth) | order.controller.ts:30, order.controller.ts:40 |
| Validation | Order must contain at least one item (schema-level) | order.schemas.ts:13 |
| Validation | Order must contain at least one item (service-level, redundant) | order.service.ts:51-53 |
| Validation | `discountCents` must be a non-negative integer, defaults to 0 | order.schemas.ts:14 |
| Validation | `customerId` and each `productId` must be valid UUIDs | order.schemas.ts:7-12 |
| Validation | `notes` capped at 1000 characters | order.schemas.ts:15 |
| Validation | Status-change `reason` capped at 500 characters | order.schemas.ts:20 |
| Validation | `pageSize` for listing capped between 1 and 100, default 20 | order.schemas.ts:25 |
| Business Logic | Duplicate `productId` line items in a single order are aggregated (summed quantities) | order.service.ts:194-202 |
| Business Logic | Customer referenced by `customerId` must exist | order.service.ts:59-60 |
| Business Logic | All referenced products must exist | order.service.ts:62-65 |
| Business Logic | All referenced products must be active (`active = true`) | order.service.ts:66-73 |
| Business Logic | Unit price and line totals are computed server-side from `Product.priceCents`, never trusted from client input | order.service.ts:77-83 |
| Business Logic | `discountCents` cannot exceed the computed subtotal | order.service.ts:87-91 |
| Business Logic | Total = subtotal - discount | order.service.ts:92 |
| Business Logic | New orders always start in `PENDING` status with an initial history entry | order.service.ts:99, 106-113 |
| Business Logic | Order numbers are sequential, formatted `ORD-######`, reserved via a dedicated sequence table | order.service.ts:245-254 |
| Business Logic | Order creation, product/customer checks, pricing, numbering, and initial history write all occur inside a single DB transaction | order.service.ts:58-123 |
| Business Logic | Status transitions follow a fixed finite-state machine; only specific from→to pairs are allowed | order.status.ts:3-14 |
| Validation | Transitioning a status to its current value is rejected as a conflict | order.service.ts:140-146 |
| Business Logic | Stock is debited only on `PENDING → PAID` transition | order.status.ts:24-31 |
| Business Logic | Stock debit fails the whole transition (422) if any item's requested quantity exceeds available stock | order.service.ts:211-224 |
| Business Logic | Stock is replenished on `CANCELLED` transitions from `PAID` or `PROCESSING` | order.status.ts:33-37 |
| Business Logic | Every status change is recorded in `OrderStatusHistory` with actor, timestamp, and optional reason | order.service.ts:158-167 |
| Business Logic | Orders can only be deleted while in `PENDING` or `CANCELLED` status | order.service.ts:184-190 |
| Business Logic | Order listing supports filtering by status, customerId, and createdAt date range, with pagination | order.repository.ts:21-45, order.schemas.ts:23-30 |

### Detailed breakdown of the business rules

---

### Business Rule: Server-Authoritative Order Pricing

**Overview**:
When an order is created, the client supplies only `customerId`, a list of `{ productId, quantity }` items, an optional `discountCents`, and optional `notes`. No pricing data is accepted from the client. All unit prices, line totals, subtotal, and grand total are computed exclusively from the current `Product.priceCents` values read inside the same database transaction as order creation.

**Detailed description**:
This rule is implemented in `OrderService.create` (`order.service.ts:50-124`). After validating that at least one item is present and aggregating duplicate product lines, the service loads the referenced `Product` records inside a Prisma transaction (`tx.product.findMany`) and builds `itemsToCreate` by looking up each aggregated item's product and computing `unitPriceCents = product.priceCents` and `totalCents = unitPriceCents * quantity`. Because this lookup happens transactionally at creation time, the price captured on the `OrderItem` reflects current catalog pricing at the moment of order placement, not a client-supplied or stale value, and is immutable afterward (there is no code path that updates `OrderItem.unitPriceCents` post-creation).

This protects the system against a class of vulnerabilities common in e-commerce APIs where a client could otherwise submit an arbitrary price for a cart line item, or where a race between price changes and order submission could create inconsistent totals. It also guarantees an audit-safe historical record: because `unitPriceCents`/`totalCents` are persisted on `OrderItem` (see `prisma/schema.prisma:99-114`) rather than recalculated on every read, an order's total remains correct even if the product's price is changed after the order is placed.

The subtotal (`subtotalCents`) is derived by summing all computed line totals (`order.service.ts:86`), and `discountCents` (validated separately, see the Discount Validation rule below) is subtracted to produce `totalCents` (`order.service.ts:92`). This two-step computation (line pricing, then aggregate subtotal/discount/total) is entirely server-side and cannot be influenced by client-submitted totals, since the `CreateOrderInput` schema (`order.schemas.ts:11-16`) has no fields for unit price, subtotal, or total — only `discountCents` is client-influenced, and it is itself bounded (see below).

**Rule workflow**:
```
1. Client submits items: [{ productId, quantity }, ...] with no price fields
2. Duplicate productId entries are aggregated (summed quantity) before pricing
3. Inside a DB transaction, referenced Products are loaded fresh from the database
4. For each aggregated item: unitPriceCents = current Product.priceCents
                              totalCents    = unitPriceCents * quantity
5. subtotalCents = sum(all item totalCents)
6. totalCents (order) = subtotalCents - discountCents (validated <= subtotal)
7. OrderItem rows persist unitPriceCents/totalCents as an immutable price snapshot
```

---

### Business Rule: Referential Integrity on Order Creation (Customer and Product Existence/Activity)

**Overview**:
An order cannot be created unless its referenced `customerId` corresponds to an existing `Customer`, and every referenced `productId` corresponds to an existing, currently-active `Product`.

**Detailed description**:
`OrderService.create` performs two sequential existence checks inside the creation transaction. First, it loads the customer via `tx.customer.findUnique({ where: { id: input.customerId } })` and throws `NotFoundError('Customer')` (HTTP 404) if no matching row exists (`order.service.ts:59-60`). Second, it loads all products referenced by the (deduplicated) item list via `tx.product.findMany({ where: { id: { in: productIds } } })` and compares the returned count to the requested count; if fewer products were found than requested, at least one `productId` does not exist, and the service throws `NotFoundError('Product')` (`order.service.ts:62-65`). This count-comparison approach means the specific missing product ID(s) are not surfaced to the client — the error is generic ("Product not found") rather than identifying which SKU/ID was invalid, which is a minor usability gap compared to the more detailed error reporting used for inactive products and insufficient stock.

Beyond mere existence, the service enforces a stricter business constraint: every referenced product must be currently active (`Product.active === true`). Inactive products (e.g., discontinued, temporarily delisted, or catalog items pending review) are filtered out of the fetched product set (`inactive = products.filter(p => !p.active)`), and if any are found, the service throws `UnprocessableEntityError('One or more products are inactive', 'INACTIVE_PRODUCT', { skus: [...] })` (HTTP 422) — notably, this error path *does* include the specific SKUs of the offending products, giving the client actionable detail (`order.service.ts:66-73`).

Both checks run before any pricing, stock, or persistence logic executes, and both are wrapped in the same `$transaction` as the rest of order creation, so a failure at this stage causes the entire transaction to roll back — no partial order, item, or history record is left behind. This "fail fast, fail atomically" approach protects data integrity: it is impossible to have an `Order` row pointing to a non-existent customer, or an `OrderItem` referencing an inactive/nonexistent product, given the current code path (assuming no direct DB writes bypass this service).

**Rule workflow**:
```
1. Begin creation transaction
2. Load Customer by input.customerId
   -> if not found: throw NotFoundError('Customer') [404], transaction rolls back
3. Load Products by aggregated productIds (findMany, id IN [...])
   -> if products.length != productIds.length: throw NotFoundError('Product') [404], rollback
4. Filter loaded products where active === false
   -> if any inactive: throw UnprocessableEntityError('INACTIVE_PRODUCT') [422]
      with details.skus = [inactive product SKUs], rollback
5. Only if both customer and all products pass: proceed to pricing and persistence
```

---

### Business Rule: Discount Cannot Exceed Subtotal

**Overview**:
The `discountCents` value supplied at order creation must not be greater than the computed `subtotalCents` of the order's line items; otherwise the order would have a negative total.

**Detailed description**:
`discountCents` is accepted from the client via `CreateOrderInput` and is schema-validated as a non-negative integer with a default of `0` (`order.schemas.ts:14`, using Zod's `.nonnegative().default(0)`), which prevents negative discounts at the transport boundary but does not by itself bound the discount relative to the order's value. The actual business constraint — that the discount cannot exceed the subtotal — is enforced in `OrderService.create` only after the subtotal has been computed from server-priced line items (`order.service.ts:86-91`): `if (input.discountCents > subtotalCents) throw new ValidationError('Discount cannot exceed subtotal', [{ path: 'discountCents', message: 'Discount exceeds subtotal' }])`.

This ordering is deliberate and necessary: the subtotal is not known until after products are loaded and priced, so this check cannot happen at the Zod schema layer (which validates shape/format before any DB access) and must instead live in the service layer, inside the transaction, after `itemsToCreate`/`subtotalCents` are computed. Because it throws a `ValidationError` (HTTP 400, same error class family as schema validation failures), from the client's perspective this behaves consistently with other input-validation errors even though it is enforced deeper in the stack — the response shape (`{ error: { code: 'VALIDATION_ERROR', message, details: [{ path, message }] } }`) matches the Zod-driven validation error format produced by `errorMiddleware` (`error.middleware.ts:26-35`) closely enough for uniform client-side handling, though it is constructed manually here rather than via Zod.

A consequence of this rule is that discounts equal to the subtotal (zero-cost orders) are explicitly allowed (`>` not `>=`), meaning `totalCents` can be `0` but never negative. There is no equivalent validation preventing a discount that results in an unreasonably low but still non-negative total (e.g., no minimum order value enforcement), and no percentage-based discount cap — the only constraint is the absolute subtotal ceiling.

**Rule workflow**:
```
1. Zod schema ensures discountCents is a non-negative integer (default 0) — shape-level only
2. Server computes subtotalCents from priced line items (see Pricing rule)
3. If discountCents > subtotalCents: throw ValidationError('Discount cannot exceed subtotal')
   -> HTTP 400, transaction rolls back, no order created
4. Else: totalCents = subtotalCents - discountCents (>= 0 guaranteed)
```

---

### Business Rule: Order Status Finite-State Machine

**Overview**:
Orders progress through a fixed set of statuses (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), and only specific from→to transitions are permitted, as defined centrally in `order.status.ts`.

**Detailed description**:
The allowed transition graph is declared as a static lookup table (`transitions`) in `order.status.ts:3-10`:
- `PENDING` → `PAID` or `CANCELLED`
- `PAID` → `PROCESSING` or `CANCELLED`
- `PROCESSING` → `SHIPPED` or `CANCELLED`
- `SHIPPED` → `DELIVERED` only
- `DELIVERED` → (none — terminal state)
- `CANCELLED` → (none — terminal state)

This encodes a linear fulfillment pipeline (PENDING→PAID→PROCESSING→SHIPPED→DELIVERED) with a cancellation escape hatch available at every non-terminal, non-shipped stage, but notably **not** available once an order reaches `SHIPPED` — an order that has shipped can only be marked `DELIVERED`, it cannot be cancelled through this API. `OrderService.changeStatus` (`order.service.ts:126-179`) enforces this by calling `canTransition(from, to)` from `order.status.ts:12-14`, which performs an array-includes lookup against the transitions table; any disallowed pair throws `InvalidStatusTransitionError` (a subclass of `ConflictError`, HTTP 409) carrying the attempted `from`/`to` in its error details (`http-errors.ts`).

A distinct edge case is handled before the state-machine check: if `from === to` (client attempts to "transition" an order to the status it already has), the service short-circuits with a differently-worded `ConflictError` using the same `INVALID_STATUS_TRANSITION` error code (`order.service.ts:140-146`) rather than delegating to `canTransition`, since a self-loop is not present in the transitions table and would otherwise produce a generic message; this gives a clearer message ("Order is already in X status") for that specific case. Two side-effecting inventory operations are coupled to specific transitions via helper predicates `shouldDebitStock` and `shouldReplenishStock` (`order.status.ts:29-37`), executed only when the transition passes the `canTransition` check — meaning inventory is never touched for a rejected transition attempt.

Every successful transition — regardless of whether it triggers stock effects — results in an `OrderStatusHistory` row being appended (`order.service.ts:159-167`), creating a complete, append-only audit trail of every status change, the acting user (`changedById`), timestamp (`changedAt`, DB-defaulted), and an optional free-text `reason`. The order's `status` column itself is a simple denormalized "current state" pointer (`order.service.ts:158`), while `OrderStatusHistory` is the source of truth for the transition timeline, both updated atomically in the same transaction as the stock side-effects.

**Rule workflow**:
```
1. Load order (with items) by id inside transaction; 404 if not found
2. from = order.status; to = input.toStatus
3. If from === to: throw ConflictError('Order is already in {to} status') [409]
4. If NOT transitions[from].includes(to): throw InvalidStatusTransitionError(from, to) [409]
5. If (from, to) == (PENDING, PAID): run stock debit check/decrement (see Stock rule)
6. If to == CANCELLED and from in (PAID, PROCESSING): run stock replenishment (see Stock rule)
7. Update Order.status = to
8. Insert OrderStatusHistory { fromStatus: from, toStatus: to, changedById, reason }
9. Re-fetch and return order with fresh items/history/customer relations
```

---

### Business Rule: Stock Debit on PENDING → PAID Transition (with Atomic Insufficient-Stock Guard)

**Overview**:
When an order transitions from `PENDING` to `PAID`, the system must atomically verify that sufficient stock exists for every ordered product and, only if all items have sufficient stock, decrement stock quantities accordingly; if any item is short, the entire transition (and the stock check) fails without partial mutation.

**Detailed description**:
This rule is the single point at which committing to payment translates into inventory commitment. `shouldDebitStock` (`order.status.ts:29-31`) narrowly matches only the exact `PENDING → PAID` pair (defined via the `STOCK_DEBIT_TRANSITION` constant), meaning stock is debited exactly once per order, at the moment of payment confirmation — not at order creation (where stock is checked implicitly only via product existence, not availability) and not on any other transition. When triggered, `OrderService.debitStock` (`order.service.ts:204-231`) reloads the current `Product` rows for all items on the order (fresh read inside the same transaction, avoiding stale stock data) and, for each item, compares `product.stockQuantity` against the requested `item.quantity`.

Critically, the check-then-decrement sequence is split into two loops: the first loop (`order.service.ts:212-221`) builds a complete `unavailable` list of every item that fails the stock check (not just the first one found), capturing `sku`, `requested`, and `available` quantities for each. Only after this full validation pass, if `unavailable.length > 0`, does the method throw `InsufficientStockError(unavailable)` (`order.service.ts:222-224`), a 422 `UnprocessableEntityError` subtype whose `details.unavailable` array gives the client complete, per-SKU shortfall information in a single response rather than requiring iterative retry-and-discover. Because this throw happens before any `product.update` calls execute, and because the whole operation runs inside the outer Prisma `$transaction` from `changeStatus`, no stock is decremented for any item if even one item is short — this is an all-or-nothing operation at the order level, preventing a scenario where an order is partially fulfilled from a stock perspective while its status still says `PAID`.

If the availability check passes, the second loop (`order.service.ts:225-230`) performs `tx.product.update` with `stockQuantity: { decrement: item.quantity }` for each item — a relative (delta) update rather than a read-then-write absolute set, which is safer under concurrent access at the Prisma/DB level for the decrement itself, though the preceding availability *check* is a separate read that is not fully protected against a concurrent transaction also debiting the same product between the check and the decrement (see Technical Debt for the associated race-condition risk, since MySQL's default isolation level and Prisma's transaction handling here do not use explicit row locking or optimistic concurrency tokens on `Product.stockQuantity`).

**Rule workflow**:
```
1. Triggered only when changeStatus transition is exactly PENDING -> PAID
2. Reload all Products referenced by the order's items (fresh read in-transaction)
3. For each item: if product missing OR product.stockQuantity < item.quantity,
   record { sku, requested: item.quantity, available: product.stockQuantity ?? 0 }
4. If any items recorded as unavailable:
     throw InsufficientStockError(unavailable) [422 UNPROCESSABLE_ENTITY /
     INSUFFICIENT_STOCK], entire changeStatus transaction rolls back, status
     remains PENDING, no stock mutated
5. Else, for each item: Product.stockQuantity -= item.quantity (atomic decrement op)
6. Proceed to persist Order.status = PAID and OrderStatusHistory entry
```

---

### Business Rule: Stock Replenishment on Cancellation from PAID or PROCESSING

**Overview**:
When an order that has already had stock debited (i.e., it was `PAID` or is `PROCESSING`) is cancelled, the previously-debited stock quantities are returned to inventory.

**Detailed description**:
`shouldReplenishStock` (`order.status.ts:33-37`) matches transitions where `to === CANCELLED` and `from` is either `PAID` or `PROCESSING` — precisely the two states reachable only after the stock-debit transition has already occurred (since debit only happens on `PENDING→PAID`, and `PROCESSING` is only reachable from `PAID`). This design correctly excludes `PENDING → CANCELLED`, since a `PENDING` order never had its stock debited in the first place, so replenishing it would incorrectly inflate inventory. It also correctly excludes `SHIPPED`/`DELIVERED` states, since (per the state machine) cancellation is not a legal transition from those states at all — an order that has shipped cannot be cancelled through this rule, and by extension its debited stock can never be replenished via this code path once it ships (physical return/RMA logic, if any, is outside this component's scope).

`OrderService.replenishStock` (`order.service.ts:233-243`) is a simple loop that, for every item on the order, issues `tx.product.update` with `stockQuantity: { increment: item.quantity }`. Notably, unlike `debitStock`, this method performs **no availability or bounds check** before incrementing — there is no verification that the resulting `stockQuantity` is sane, no check against a maximum stock ceiling, and no idempotency guard against double-replenishment (though the state machine's terminal nature of `CANCELLED` — no outgoing transitions — combined with the "cannot transition to same status" guard in `changeStatus`, prevents the same order from being cancelled twice, which is the primary safeguard against double-replenishment in practice).

This rule and the debit rule together are designed to keep `Product.stockQuantity` consistent with "stock that is not currently reserved by an active paid/processing order," but the consistency depends entirely on the debit and replenish operations being correctly paired for every order that reaches `PAID`. Since `SHIPPED`/`DELIVERED` orders never replenish (by design — the goods have left the warehouse), and `CANCELLED` is terminal, each unit of stock debited at `PAID` either eventually ships (permanently consumed) or is replenished via cancellation from `PAID`/`PROCESSING` (returned) — there is no code path that leaves stock permanently and incorrectly debited without either shipping or replenishment, assuming no direct DB manipulation outside this service.

**Rule workflow**:
```
1. Triggered when changeStatus transition has to == CANCELLED and
   from in { PAID, PROCESSING }
2. For each item on the order: Product.stockQuantity += item.quantity (atomic increment)
3. No availability/ceiling check performed
4. Proceed to persist Order.status = CANCELLED and OrderStatusHistory entry
```

---

### Business Rule: Order Deletion Restricted to PENDING or CANCELLED Status

**Overview**:
An order can only be hard-deleted via `DELETE /api/v1/orders/:id` while its status is `PENDING` or `CANCELLED`; orders that are `PAID`, `PROCESSING`, `SHIPPED`, or `DELIVERED` cannot be deleted.

**Detailed description**:
`OrderService.delete` (`order.service.ts:181-192`) first loads the order via `OrderRepository.findById` and throws `NotFoundError('Order')` (404) if it does not exist. It then checks `order.status !== OrderStatus.PENDING && order.status !== OrderStatus.CANCELLED`; if the order is in any other status, it throws `ConflictError('Order can only be deleted while in PENDING or CANCELLED status', 'INVALID_ORDER_STATE_FOR_DELETE', { status: order.status })` (409), including the current status in the error details for client diagnostics. Only if the guard passes does it call `OrderRepository.deleteById`, which issues a hard Prisma `delete`.

This rule protects against destroying financially or logistically significant records: once an order has been paid (`PAID`) — meaning stock has been debited and, presumably, a real payment has been captured by an external or adjacent system — deleting the order row would destroy the audit trail and financial record while stock/payment side effects have already occurred, with no compensating reversal. The rule deliberately allows deletion of `CANCELLED` orders (which is somewhat unusual, since cancellation itself is normally the terminal "soft-delete" equivalent) — this permits cleanup of cancelled orders (e.g., duplicate/erroneous cancelled test orders) without requiring a separate purge mechanism, though it also means a cancelled order's audit trail (its `OrderStatusHistory`) is fully destroyed on deletion rather than retained, since `OrderItem`/`OrderStatusHistory` cascade-delete with the parent `Order` (`onDelete: Cascade` in `prisma/schema.prisma:108,125`).

Unlike creation and status-change, `delete` in `OrderController` and `OrderService.delete` does **not** pass or use the acting `userId` at all — no audit record is created for the deletion event itself (there is no "who deleted this order" trail), which is an asymmetry relative to the meticulous history-tracking applied to status transitions and creation.

**Rule workflow**:
```
1. Load order by id (repository.findById, no relations)
   -> if not found: throw NotFoundError('Order') [404]
2. If order.status not in { PENDING, CANCELLED }:
     throw ConflictError('INVALID_ORDER_STATE_FOR_DELETE') [409] with status in details
3. Else: hard-delete Order row (Prisma delete)
   -> OrderItem and OrderStatusHistory rows cascade-delete at the DB level
4. Controller responds 204 No Content (no body)
```

---

### Business Rule: Sequential Human-Readable Order Numbering

**Overview**:
Every created order is assigned a unique, sequential, zero-padded order number in the format `ORD-######`, generated via a dedicated single-row sequence table rather than derived from the order's UUID primary key.

**Detailed description**:
`OrderService.reserveOrderNumber` (`order.service.ts:245-254`) uses an upsert against `OrderNumberSequence` (a single-row table with fixed `id: 1`, see `prisma/schema.prisma:133-138`): on first use it creates the row with `nextValue: 2` and treats the "current" value as `1`; on subsequent calls it increments `nextValue` by 1 and computes `current = seq.nextValue - 1` as the number to assign, then formats it as `ORD-${String(current).padStart(6, '0')}` (e.g., `ORD-000001`, `ORD-000042`). Because this upsert happens inside the same Prisma `$transaction` as the rest of order creation (it is called with the transactional `tx` client, `order.service.ts:93`), the reservation and the actual `Order.create` are atomic — if any later step in creation fails and the transaction rolls back, the sequence increment is also rolled back, preventing permanently "burned" order numbers from failed attempts (unlike, e.g., a Postgres `SERIAL`/sequence object, which does not roll back on transaction abort — this custom table-based approach is more conservative and gap-free by design, at some cost to concurrency, since concurrent order creations will serialize on this single row).

The `Order.orderNumber` column is declared `@unique` in the Prisma schema (`prisma/schema.prisma:76`), providing a database-level backstop against duplicate order numbers even if application logic were to misbehave; a violation would surface through `errorMiddleware`'s handling of Prisma's `P2002` unique-constraint error code as a generic 409 Conflict (`error.middleware.ts:37-47`), not as a domain-specific order-numbering error.

This human-readable numbering scheme is distinct from and in addition to the order's `id` (a UUID primary key used in all URL paths, e.g. `GET /orders/:id`), serving presentation/business-communication purposes (e.g., customer service references, invoices) where a UUID would be unwieldy.

**Rule workflow**:
```
1. Inside the order-creation transaction, upsert OrderNumberSequence(id=1):
     - if absent: create with nextValue = 2, "current" assigned = 1
     - if present: increment nextValue by 1, "current" assigned = (new nextValue - 1)
2. Format: orderNumber = "ORD-" + current padded to 6 digits with leading zeros
3. Persist orderNumber on the new Order row (unique constraint enforced at DB level)
4. If the transaction later fails/rolls back, the sequence increment rolls back too
```

---

## 4. Component Structure

```
src/modules/orders/
├── order.controller.ts     # HTTP handlers: list, getById, create, changeStatus, delete
│                            # (thin — auth guard + delegation to OrderService + res mapping)
├── order.routes.ts         # Express Router: mounts authenticate + validate middleware
│                            # per route, wires handlers to HTTP verbs/paths
├── order.schemas.ts        # Zod schemas: createOrderSchema, updateOrderStatusSchema,
│                            # listOrdersQuerySchema, orderIdParamSchema + inferred types
├── order.service.ts        # Core business logic: create (transactional, pricing,
│                            # referential integrity), changeStatus (state machine +
│                            # stock effects), delete (state guard), list, getById,
│                            # private helpers (aggregateItems, debitStock,
│                            # replenishStock, reserveOrderNumber)
├── order.repository.ts     # Prisma data-access: list (paginated + filtered),
│                            # findByIdWithRelations, findById, deleteById
└── order.status.ts         # Pure functions: order status transition table,
                             # canTransition, allowedTransitions, isTerminal,
                             # shouldDebitStock, shouldReplenishStock

Related shared/cross-cutting files consulted for this analysis:
src/
├── app.ts                          # buildControllers(): wires OrderRepository ->
│                                    # OrderService -> OrderController; buildApp():
│                                    # mounts middleware, /api/v1 router, error handler
├── routes/index.ts                 # buildApiRouter(): mounts buildOrderRouter at /orders
├── middlewares/
│   ├── auth.middleware.ts          # authenticate (JWT verify, sets req.user),
│   │                                # requireRole (NOT used on order routes)
│   ├── validate.middleware.ts      # validate({body,query,params}) — Zod-driven
│   │                                # request validation, maps ZodError -> ValidationError
│   └── error.middleware.ts         # Central error handler: AppError -> statusCode/JSON,
│                                    # ZodError -> 400, Prisma P2002/P2025 -> 409/404,
│                                    # else -> 500 + structured log
├── shared/
│   ├── errors/
│   │   ├── app-error.ts            # Base AppError class (message, statusCode,
│   │   │                            # errorCode, details)
│   │   ├── http-errors.ts          # BadRequestError, ValidationError, UnauthorizedError,
│   │   │                            # ForbiddenError, NotFoundError, ConflictError,
│   │   │                            # UnprocessableEntityError,
│   │   │                            # InvalidStatusTransitionError (extends Conflict),
│   │   │                            # InsufficientStockError (extends Unprocessable)
│   │   └── index.ts                # Barrel export
│   ├── http/response.ts            # PaginatedResponse<T>, buildPagination, paginated()
│   └── logger/                     # Structured logger used by error.middleware
└── config/
    ├── database.ts                 # Prisma client instance (used by tests, app.ts)
    └── env.ts                      # Env/config loader (JWT_SECRET, etc.)

prisma/schema.prisma                # Order, OrderItem, OrderStatusHistory,
                                     # OrderNumberSequence, Customer, Product, User
                                     # models and OrderStatus enum
```

---

## 5. Dependency Analysis

```
Internal Dependencies (compile-time imports):

order.controller.ts
  -> order.service.ts (type OrderService, constructor-injected)
  -> order.schemas.ts (type ListOrdersQuery)
  -> shared/errors/index.ts (UnauthorizedError)

order.service.ts
  -> order.repository.ts (type OrderRepository, OrderWithRelations; constructor-injected)
  -> order.schemas.ts (types: CreateOrderInput, ListOrdersQuery, UpdateOrderStatusInput)
  -> order.status.ts (canTransition, shouldDebitStock, shouldReplenishStock)
  -> shared/errors/index.ts (ConflictError, InsufficientStockError,
     InvalidStatusTransitionError, NotFoundError, UnprocessableEntityError, ValidationError)
  -> shared/http/response.ts (paginated, PaginatedResponse)
  -> @prisma/client (Order, OrderStatus, Prisma, PrismaClient types) [external]

order.repository.ts
  -> @prisma/client (Order, OrderItem, OrderStatus, OrderStatusHistory, Prisma,
     PrismaClient types) [external]

order.routes.ts
  -> order.controller.ts (type OrderController)
  -> order.schemas.ts (createOrderSchema, listOrdersQuerySchema, orderIdParamSchema,
     updateOrderStatusSchema)
  -> middlewares/auth.middleware.ts (authenticate)
  -> middlewares/validate.middleware.ts (validate)
  -> express (Router) [external]

order.schemas.ts
  -> zod [external]
  -> @prisma/client (OrderStatus enum) [external]

order.status.ts
  -> @prisma/client (OrderStatus enum) [external]

Composition root (src/app.ts):
  buildControllers(prisma) instantiates:
    OrderRepository(prisma) -> OrderService(orderRepository, prisma) -> OrderController(orderService)
  buildApp() wires:
    requestLogger -> /api/v1 (buildApiRouter -> buildOrderRouter) -> notFoundHandler -> errorMiddleware

Downstream consumer:
  src/routes/index.ts -> buildOrderRouter(controllers.orders) -> mounted at '/orders'
    under buildApiRouter, itself mounted at '/api/v1' in app.ts

External Dependencies:
- express (^4.x, exact version not verified in this analysis) - HTTP routing/middleware framework
- @prisma/client / Prisma ORM - Database ORM, transaction management, generated types
  (Order, OrderStatus, OrderItem, OrderStatusHistory, Product, Customer, PrismaClient)
- zod - Runtime schema validation and static type inference for request payloads
- jsonwebtoken (via auth.middleware.ts, indirectly relevant to all order routes) - JWT verification
- MySQL (via Prisma datasource provider="mysql", prisma/schema.prisma:6-9) - Relational
  data persistence for Order, OrderItem, OrderStatusHistory, OrderNumberSequence, Customer,
  Product, User tables
- vitest + supertest (test-only) - Integration/HTTP test execution for tests/orders.test.ts
```

---

## 6. Afferent and Efferent Coupling

Analysis unit: classes/modules within and immediately around the Orders component (TypeScript classes and cohesive function modules).

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-----------|------------------------|-------------------------|----------|
| OrderController | 1 (routes/index.ts via app.ts wiring, order.routes.ts) | 2 (OrderService, shared/errors) | Low |
| OrderService | 1 (OrderController) | 6 (OrderRepository, order.schemas types, order.status functions, shared/errors (6 error classes), shared/http/response, @prisma/client) | High |
| OrderRepository | 1 (OrderService) | 1 (@prisma/client) | Medium |
| order.status.ts (transition functions) | 1 (OrderService) | 1 (@prisma/client OrderStatus enum) | High |
| order.schemas.ts (Zod schemas) | 3 (order.routes.ts, OrderService via inferred types, OrderController via ListOrdersQuery type) | 2 (zod, @prisma/client) | Medium |
| order.routes.ts | 1 (routes/index.ts) | 4 (OrderController, order.schemas.ts, auth.middleware.ts, validate.middleware.ts) | Low |
| shared/errors (AppError hierarchy) | 5+ (OrderService, OrderController, validate.middleware, auth.middleware, error.middleware, and equivalent modules in users/customers/products/auth) | 0 | High (shared kernel) |

Notes on interpretation:
- `OrderService` has the highest efferent coupling (Ce = 6) within the module, consistent with it being the sole holder of business logic — it reaches into the repository, validation-derived types, the status state machine, six distinct error classes, the pagination helper, and Prisma types/client directly (for `$transaction`).
- `order.status.ts` is marked "High" criticality despite low fan-in/fan-out counts because it is the single source of truth for a state machine that gates financially significant operations (stock debit/replenishment); a defect here has outsized business impact relative to its small size (38 lines).
- `shared/errors` is a shared-kernel dependency with very high afferent coupling across the whole codebase (not just Orders) — any breaking change to `AppError`/`http-errors.ts` ripples through every module's controllers, services, and middleware.
- `OrderController` itself has low coupling in both directions by design — it is a thin adapter, which is a positive cohesion/coupling characteristic for a controller layer.
- `OrderRepository`'s efferent coupling is limited to `@prisma/client`, but note that `OrderService.create` and `OrderService.changeStatus` bypass `OrderRepository` entirely and call `this.prisma` directly for transactional writes — meaning the *true* efferent coupling of `OrderService` to `@prisma/client` is higher than what is mediated through `OrderRepository`, an architectural inconsistency discussed in Technical Debt.

---

## 7. Endpoints

All endpoints are mounted at `/api/v1/orders` (see `src/app.ts:67` and `src/routes/index.ts:28`) and require a valid `Authorization: Bearer <JWT>` header (enforced by the `authenticate` middleware applied to the whole router in `order.routes.ts:14`). No route applies `requireRole`, so both `ADMIN` and `OPERATOR` roles have identical access to all order operations.

| Endpoint | Method | Description | Auth Required | Request Validation | Success Status |
|----------|--------|--------------|----------------|---------------------|-----------------|
| /api/v1/orders | GET | List orders with optional filters (status, customerId, from/to date range) and pagination (page, pageSize) | Yes (JWT) | `listOrdersQuerySchema` (query) | 200 |
| /api/v1/orders/:id | GET | Retrieve a single order by UUID with items, history, and customer relations | Yes (JWT) | `orderIdParamSchema` (params) | 200 |
| /api/v1/orders | POST | Create a new order (customerId, items[], discountCents, notes); server computes pricing and initial PENDING status | Yes (JWT) + `req.user` check | `createOrderSchema` (body) | 201 |
| /api/v1/orders/:id/status | PATCH | Change an order's status (toStatus, optional reason) per the allowed state-machine transitions; triggers stock debit/replenishment as applicable | Yes (JWT) + `req.user` check | `orderIdParamSchema` (params) + `updateOrderStatusSchema` (body) | 200 |
| /api/v1/orders/:id | DELETE | Delete an order; only allowed while status is PENDING or CANCELLED | Yes (JWT) | `orderIdParamSchema` (params) | 204 |

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|-----------------|
| MySQL (via Prisma) | Database | Persistence of Order, OrderItem, OrderStatusHistory, OrderNumberSequence, and cross-referenced Customer/Product/User rows | Prisma Client (SQL under the hood) | Relational rows / Prisma-typed objects | Prisma known errors (P2002 unique violation, P2025 record-not-found) mapped centrally in `error.middleware.ts:37-54` to 409/404; all other Prisma/unexpected errors fall through to a generic 500 with structured logging |
| Customer module (internal, via shared Prisma schema) | Internal data dependency | Order creation validates `customerId` exists (`tx.customer.findUnique`) | In-process Prisma call (same transaction) | Prisma `Customer` model | `NotFoundError('Customer')` (404) if missing |
| Product module (internal, via shared Prisma schema) | Internal data dependency | Order creation validates products exist/are active and computes pricing; status changes debit/replenish `Product.stockQuantity` | In-process Prisma call (same transaction) | Prisma `Product` model | `NotFoundError('Product')` (404) if missing; `UnprocessableEntityError('INACTIVE_PRODUCT')` (422) if inactive; `InsufficientStockError` (422) if stock too low |
| Auth module (JWT) | Internal middleware dependency | Authenticates every order request and supplies `req.user.id` used as `createdById`/`changedById` audit fields | HTTP header (`Authorization: Bearer <token>`), in-process JWT verification | JWT (jsonwebtoken) | `UnauthorizedError` (401) on missing/invalid/expired token, thrown by `authenticate` middleware before reaching the controller |
| Validation layer (Zod) | Internal middleware dependency | Validates/coerces all inbound query/params/body for order routes before controller execution | In-process function calls | Zod schema objects | `ValidationError` (400) with per-field `details` array, thrown by `validate` middleware on `ZodError` |
| Structured logger | Internal cross-cutting dependency | Logs unhandled errors reaching `error.middleware.ts` with request context (requestId, method, path) | In-process function calls | Structured log object | N/A (logging is a side effect, not itself error-handled) |

No external third-party HTTP/API integrations (e.g., payment gateways, shipping providers, notification services) were found within the traced dependency graph of `OrderController` — the `PAID` status transition records a "payment confirmed" state via `reason` text supplied by the caller, but there is no evidence of an actual payment-gateway integration invoked by this component; payment confirmation appears to be an out-of-band process communicated to this API via the `PATCH /:id/status` endpoint.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|-----------------|----------|---------|
| Layered Architecture (Controller-Service-Repository) | OrderController -> OrderService -> OrderRepository | order.controller.ts, order.service.ts, order.repository.ts | Separation of HTTP concerns, business logic, and data access |
| Dependency Injection (constructor injection, manual/composition-root) | `OrderController(orders: OrderService)`, `OrderService(orders: OrderRepository, prisma: PrismaClient)` wired in `buildControllers` | order.controller.ts:7, order.service.ts:27-30, app.ts:42-44 | Testability and decoupling from concrete instantiation; no DI framework used, plain constructor wiring |
| Repository Pattern | `OrderRepository` encapsulates Prisma queries for list/find/delete | order.repository.ts | Data access abstraction, though partially bypassed (see Technical Debt) |
| Finite State Machine | `transitions` lookup table + `canTransition`/`allowedTransitions`/`isTerminal` | order.status.ts:3-22 | Centralizes and makes explicit/testable the legal order-status transition graph |
| Strategy-like Predicate Functions | `shouldDebitStock`, `shouldReplenishStock` | order.status.ts:29-37 | Decouples "which transition triggers which side effect" from the orchestration logic in `OrderService.changeStatus` |
| Unit of Work / Transaction Script | Prisma `$transaction` wrapping multi-step reads/writes in `create` and `changeStatus` | order.service.ts:58-123, 131-178 | Atomicity across multi-entity mutations (order + items + history + product stock) |
| Custom Exception Hierarchy | `AppError` base class with typed subclasses (`NotFoundError`, `ConflictError`, `ValidationError`, `InvalidStatusTransitionError` extends `ConflictError`, `InsufficientStockError` extends `UnprocessableEntityError`) | shared/errors/app-error.ts, shared/errors/http-errors.ts | Consistent HTTP status/error-code mapping via centralized `errorMiddleware`, domain-specific errors carry structured `details` |
| Centralized Error-Handling Middleware | `errorMiddleware` as the terminal Express error handler | middlewares/error.middleware.ts | Single place mapping `AppError`/`ZodError`/Prisma errors to consistent JSON error envelopes |
| Middleware Chain / Pipeline | `authenticate` -> `validate(...)` -> controller handler per route | order.routes.ts:14-24 | Cross-cutting concerns (auth, validation) applied declaratively before business logic |
| DTO / Schema-Derived Types | Zod schemas with `z.infer<>` producing `CreateOrderInput`, `UpdateOrderStatusInput`, `ListOrdersQuery` | order.schemas.ts:32-34 | Single source of truth for both runtime validation and compile-time types, avoiding drift between validation and type definitions |
| Idempotent Aggregation | `aggregateItems` merges duplicate product line items pre-pricing | order.service.ts:194-202 | Ensures pricing/stock logic operates on a canonical, deduplicated item list |
| Sequence Table Pattern | `OrderNumberSequence` single-row upsert-based counter | order.service.ts:245-254, schema.prisma:133-138 | Transactional, gap-free (on rollback) sequential business-identifier generation, portable across DB vendors compared to native DB sequences |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|-----------------|-------|--------|
| High | OrderService.debitStock | Stock availability check (read) and decrement (write) are not protected by explicit row-level locking or optimistic concurrency (e.g., `SELECT ... FOR UPDATE` or a version column); under concurrent `PENDING -> PAID` transitions for orders sharing a product, a race condition could allow both transactions to pass the availability check before either decrements, resulting in oversold stock (relies entirely on default MySQL/Prisma transaction isolation behavior, which was not verified in this analysis) | Potential inventory oversell under concurrent load; data-integrity risk |
| High | OrderService (create, changeStatus) vs OrderRepository | `OrderService` bypasses `OrderRepository` entirely for `create` and `changeStatus`, using `this.prisma.$transaction(...)` directly with raw `tx.order.*`/`tx.product.*`/`tx.customer.*` calls, while `list`/`getById`/`delete` go through `OrderRepository`. This breaks the Repository pattern's abstraction boundary and means data-access logic is split across two layers inconsistently | Reduced testability (service unit tests would need to mock Prisma's transaction API directly rather than a repository interface), harder to swap persistence layer, inconsistent architecture across the same class |
| Medium | OrderController.create / OrderController.changeStatus | The `if (!req.user) throw new UnauthorizedError()` checks are logically unreachable given the current router wiring (`authenticate` middleware runs first and always populates `req.user` or short-circuits with 401 before the controller is reached) — this is dead/defensive code that adds no real protection under the current composition but could mask a routing misconfiguration if `OrderController` is ever mounted without `authenticate` | Minor: false sense of security if router wiring changes; harmless but non-obvious redundancy for maintainers |
| Medium | OrderService.create | The empty-items validation (`if (input.items.length === 0) throw ValidationError`) at `order.service.ts:51-53` is unreachable in practice because `createOrderSchema.items` already enforces `.min(1, ...)` at the Zod/middleware layer (`order.schemas.ts:13`), which runs before the controller/service are invoked | Dead code; minor maintenance overhead, slight risk of divergent error messages if schema changes without updating this line |
| Medium | OrderService.debitStock error reporting | When a product referenced by an order item is not found during the stock-debit re-check (`!product` branch, `order.service.ts:214-219`), the `sku` field falls back to `item.productId` (a UUID) rather than a real SKU, and `available` falls back to `0` — since this data path re-queries the same set of products already validated to exist at order-creation time, hitting this branch would imply a product was deleted after order creation, an edge case not covered by any visible test | Ambiguous/misleading error payload (`sku` field containing a UUID) surfaced to API clients in an already-rare edge case |
| Medium | OrderService.replenishStock | No upper-bound or sanity check on stock replenishment (`stockQuantity: { increment: item.quantity } }`); relies entirely on the state machine's terminal `CANCELLED` state and the "same-status" transition guard to prevent double-replenishment, with no explicit idempotency key or check that the order was actually previously debited | If the state-machine invariants are ever violated (e.g., future code change allows re-entering PAID from CANCELLED, or a direct DB manipulation), stock could be double-replenished with no defense-in-depth check |
| Medium | OrderService.create (Product not-found error) | `NotFoundError('Product')` on count mismatch (`order.service.ts:62-65`) does not identify which specific `productId`(s) were not found, unlike the inactive-product and insufficient-stock errors which include specific SKU details | Reduced API usability/debuggability for clients submitting orders with typo'd or stale product IDs |
| Low | OrderService.delete | No audit trail (no `OrderStatusHistory` entry, no "deleted by" record) is created when an order is deleted, unlike creation and status changes which are meticulously audited; the acting user's ID is not even passed into `OrderService.delete` from the controller | Loss of accountability/audit trail for order deletions, inconsistent with the audit rigor applied elsewhere in the module |
| Low | order.repository.ts / order.service.ts | `Order` type from `list()` (via `OrderRepository.list`) omits `items`/`history`/`customer` relations (plain `Order[]`), while `getById`/`create`/`changeStatus` return `OrderWithRelations`; API consumers must be aware the list endpoint's items have a different shape than detail/create/status-change responses, which is not explicitly documented in code comments | Potential client-side confusion/bugs if consumers assume uniform response shape across list vs. detail endpoints |
| Low | order.controller.ts | No request/response logging, tracing span, or explicit input sanitization occurs at the controller level itself — all cross-cutting concerns are pushed to middleware, which is architecturally fine, but means the controller has zero defensive coding beyond the single `req.user` check | Not a functional defect; noted for completeness — controller correctly relies on the middleware pipeline |
| Low | order.schemas.ts | `notes` and `reason` fields are validated only for max length (`.max(1000)`, `.max(500)`); no sanitization against injection/XSS-style content is performed at this layer (though Prisma parameterizes queries, mitigating SQL injection; downstream rendering contexts, if any, were not in scope for this analysis) | Potential stored-content risk if these free-text fields are ever rendered unsanitized in a UI outside this API's scope |

---

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage (qualitative) | Test Quality |
|-----------|------------|---------------------|--------------------------|----------------|
| OrderController | 0 | 8 (indirect, via HTTP through `tests/orders.test.ts`) | All 5 endpoints exercised indirectly (GET list, GET by id implicitly via created order id, POST, PATCH status, DELETE) | No isolated controller unit tests exist (e.g., no mocked-`OrderService` tests asserting status codes/error passthrough in isolation); reliance on full-stack HTTP tests means controller-only regressions (e.g., wrong status code on an untested branch) could be masked by service-level test overlap |
| OrderService | 0 | 8 (indirect, via `tests/orders.test.ts`) | Create (pricing, totals, history), status transition happy path (PENDING->PAID with stock decrement), invalid transition (409), insufficient stock (422), cancellation replenishment (PAID->CANCELLED), delete guard (409 for non-PENDING/CANCELLED) are all covered at the HTTP/integration level | Good assertions on response bodies and side effects (e.g., verifies `Product.stockQuantity` via direct Prisma read after the HTTP call); missing coverage: `getById` 404 case, `list` pagination beyond a single filter scenario, discount-exceeds-subtotal validation (400), inactive-product rejection (422 INACTIVE_PRODUCT), duplicate-item aggregation behavior, order-number sequence correctness beyond regex format check, SHIPPED->DELIVERED and PROCESSING->SHIPPED transitions, delete happy-path (successful deletion of a PENDING/CANCELLED order), unauthorized/missing-token cases specific to order routes |
| OrderRepository | 0 | 0 direct (only indirectly exercised as a dependency of the tested service flows for `list`, and implicitly via `create`'s HTTP test assertions on returned data) | `findById`/`deleteById` paths (used by `OrderService.delete`) are exercised only in the "refuses to delete" negative test; the happy-path delete (successful 204) is not tested | No dedicated repository-level tests (e.g., verifying `buildWhere` filter combinations for `from`/`to` date ranges, or `customerId` filter) exist |
| order.status.ts (state machine) | 0 | 0 direct (exercised only through the subset of transitions covered by `tests/orders.test.ts`: PENDING->PAID, PENDING->SHIPPED (rejected), PAID->CANCELLED) | Partial — `PROCESSING->SHIPPED`, `SHIPPED->DELIVERED`, `PROCESSING->CANCELLED`, and terminal-state rejection (e.g., attempting a transition from `DELIVERED` or `CANCELLED`) are not covered by any test found in this analysis | No pure unit tests for `canTransition`/`allowedTransitions`/`isTerminal`/`shouldDebitStock`/`shouldReplenishStock`, despite these being simple, highly-testable pure functions well-suited to fast unit tests independent of the database |
| order.schemas.ts (Zod validation) | 0 | Indirect only (schema violations are not explicitly tested in `tests/orders.test.ts`; the only near-validation test asserts a 404 for a well-formed but non-existent product UUID, not a schema-level 400) | Low direct coverage — no test asserts 400 responses for malformed `createOrderSchema`/`updateOrderStatusSchema`/`listOrdersQuerySchema` payloads (e.g., missing customerId, negative discountCents, invalid UUID format, pageSize > 100, invalid `toStatus` enum value) | Gap: validation-layer behavior for the Orders module is entirely unverified by tests in this repository as located |

**Test file locations referenced:**
- `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/tests/orders.test.ts` — primary integration test suite for the Orders component (8 `it` blocks under a single `describe('Orders')`)
- `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/tests/helpers/factories.ts` — shared test fixtures/helpers (`getTestApp`, `bootstrapAuthenticatedUser`, `createTestCustomer`, `createTestProduct`) used by `orders.test.ts` and, per its own `createTestUser`/`loginAndGetToken` helpers, shared with `tests/auth.test.ts`
- `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/tests/setup.ts` — global test setup (references "order" per a preliminary grep match; likely DB reset/seed logic shared across suites, not Orders-specific business logic)
- No other test files in the repository (outside `tests/`) were found referencing order-related code during this analysis.

Overall test strategy for this component is exclusively black-box HTTP integration testing via `supertest` against a fully wired `Express` app (`buildApp`) backed by a real Prisma/MySQL connection (per `tests/helpers/factories.ts:9-14` and `tests/orders.test.ts:9`), with no mocking of `OrderRepository`, `OrderService`, or Prisma at any level. This yields high confidence in end-to-end correctness for the covered scenarios but no fast, isolated unit-test safety net for the pure-logic modules (`order.status.ts`) or for controller/service behavior in isolation from the database.

---

*End of Component Deep Analysis Report — OrderController*
