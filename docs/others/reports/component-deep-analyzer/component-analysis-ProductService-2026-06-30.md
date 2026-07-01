# Component Deep Analysis Report: ProductService

**Project:** order-management-api
**Component analyzed:** `ProductService` (src/modules/products/product.service.ts)
**Analysis date:** 2026-06-30

---

## 1. Executive Summary

`ProductService` (src/modules/products/product.service.ts) is the application-layer service that implements the business logic for product catalog management in the Order Management System REST API. It sits between `ProductController` (HTTP boundary) and `ProductRepository` (Prisma-backed persistence boundary), and exposes five operations: `list`, `getById`, `create`, `update`, and `delete`.

The component is a small, single-responsibility service class with a constructor-injected dependency on `ProductRepository`. It has no direct dependency on the HTTP framework (Express) or on the ORM (Prisma) beyond the `Product` type import, which keeps it reasonably decoupled from transport and persistence concerns. Its core responsibilities are:

- Enforcing SKU uniqueness on product creation and update (a domain invariant not expressible purely through Zod schema validation).
- Enforcing existence checks before update and delete operations (fail-fast with domain-specific `NotFoundError`).
- Translating repository pagination primitives (`skip`/`take`) from page-based query parameters.
- Applying partial-update semantics (only fields explicitly present in the input are forwarded to the repository).

Key findings:

1. **No dedicated automated test coverage exists for `ProductService` or the `/products` REST endpoints.** The only place products are exercised in tests is indirectly, via a `createTestProduct` factory used inside `tests/orders.test.ts` to set up fixtures for Order module tests. This is a significant coverage gap (see Section 11 and Section 10).
2. The component correctly implements the "check-then-act" pattern for uniqueness and existence guards, but these checks are not race-condition-safe (see Technical Debt, Section 10) — concurrent requests could both pass the existence/uniqueness check before either write completes, relying on the database's unique constraint (and the global error middleware's `P2002` handling) as the actual safety net.
3. The component is a clean example of the Controller-Service-Repository layering pattern used consistently across the codebase (auth, users, customers, orders modules follow the same shape).
4. `ProductService` has an important cross-module consumer: `OrderService` (src/modules/orders/order.service.ts) reads and writes `Product` rows directly via `tx.product.*` Prisma calls inside database transactions, **bypassing `ProductService` and `ProductRepository` entirely**. This means product stock mutation business rules (stock decrement/increment on order placement/cancellation) live outside the analyzed component boundary, in the Orders module, creating a duplicated/parallel data-access path to the same table.
5. Authorization is coarse-grained: all `/products` routes require only `authenticate` (valid JWT), with no `requireRole` restriction, meaning both `ADMIN` and `OPERATOR` roles have full CRUD access to the product catalog, including create/update/delete.

---

## 2. Data Flow Analysis

### 2.1 List products (`GET /api/v1/products`)

```
1. Request enters product.routes.ts -> authenticate middleware (JWT check)
2. validate({ query: listProductsQuerySchema }) middleware coerces/validates query params
   (page, pageSize, search, active) via Zod (product.schemas.ts)
3. ProductController.list (product.controller.ts:8-16) casts req.query to ListProductsQuery
4. ProductService.list (product.service.ts:14-22):
   a. Computes skip = (page - 1) * pageSize
   b. Delegates to ProductRepository.list({ search, active, skip, take: pageSize })
5. ProductRepository.list (product.repository.ts:25-37):
   a. buildWhere() constructs Prisma OR filter (name/sku contains) + active equality filter
   b. Executes prisma.$transaction([findMany, count]) for atomic paginated read
6. ProductService wraps result in paginated() helper (shared/http/response.ts) ->
   { data: Product[], pagination: { page, pageSize, total, totalPages } }
7. ProductController returns 200 with JSON body
```

### 2.2 Get product by ID (`GET /api/v1/products/:id`)

```
1. Request enters product.routes.ts -> authenticate
2. validate({ params: productIdParamSchema }) ensures :id is a UUID
3. ProductController.getById passes req.params.id to ProductService.getById
4. ProductService.getById (product.service.ts:24-28):
   a. ProductRepository.findById(id) -> Prisma findUnique
   b. If null -> throws NotFoundError('Product') (mapped to 404 by error.middleware.ts)
5. Controller returns 200 with the Product JSON
```

### 2.3 Create product (`POST /api/v1/products`)

```
1. Request enters product.routes.ts -> authenticate
2. validate({ body: createProductSchema }) validates/defaults sku, name, description,
   priceCents, stockQuantity (default 0), active (default true)
3. ProductController.create passes req.body (typed CreateProductInput) to ProductService.create
4. ProductService.create (product.service.ts:30-43):
   a. ProductRepository.findBySku(input.sku) - uniqueness pre-check
   b. If existing product found -> throws ConflictError('SKU already in use', 'SKU_ALREADY_USED') (409)
   c. Else -> ProductRepository.create(...) persists via Prisma, normalizing description
      (undefined -> null)
5. Controller returns 201 with the created Product
6. (Fallback safety net) If a race condition slips past step 4a, Prisma's unique constraint
   on `sku` triggers P2002, caught by error.middleware.ts and mapped to 409 CONFLICT
```

### 2.4 Update product (`PATCH /api/v1/products/:id`)

```
1. Request enters product.routes.ts -> authenticate
2. validate({ params, body: updateProductSchema }) - body is a partial schema, all fields optional
3. ProductController.update passes id + req.body to ProductService.update
4. ProductService.update (product.service.ts:45-61):
   a. Calls this.getById(id) - throws NotFoundError if product does not exist (existence guard)
   b. If input.sku provided: ProductRepository.findBySku(input.sku); if a different product
      (sameSku.id !== id) already owns that SKU -> throws ConflictError('SKU_ALREADY_USED')
   c. Builds a sparse update object using conditional spreads, forwarding only fields that
      are !== undefined in the input (true partial-update / PATCH semantics)
   d. ProductRepository.update(id, data) persists via Prisma
5. Controller returns 200 with the updated Product
```

### 2.5 Delete product (`DELETE /api/v1/products/:id`)

```
1. Request enters product.routes.ts -> authenticate
2. validate({ params: productIdParamSchema })
3. ProductController.delete passes id to ProductService.delete
4. ProductService.delete (product.service.ts:63-66):
   a. Calls this.getById(id) - throws NotFoundError if product does not exist
   b. ProductRepository.delete(id) - hard delete via Prisma
5. Controller returns 204 No Content
6. (Not handled explicitly) If the product is referenced by existing OrderItem rows,
   the delete would fail at the database foreign-key level; Prisma would raise an error that
   is not explicitly mapped by name in error.middleware.ts (only P2002/P2025 are special-cased),
   so it would fall through to a generic 500 Internal Server Error unless the FK violation
   maps to a Prisma code not handled here
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation | SKU must be 2-64 chars | product.schemas.ts:6 |
| Validation | Name must be 2-200 chars | product.schemas.ts:7 |
| Validation | Description optional, max 2000 chars | product.schemas.ts:8 |
| Validation | Price (priceCents) must be a non-negative integer | product.schemas.ts:9 |
| Validation | Stock quantity must be a non-negative integer, defaults to 0 | product.schemas.ts:10 |
| Validation | `active` defaults to `true` on creation | product.schemas.ts:11 |
| Validation | Product ID path param must be a valid UUID | product.schemas.ts:3 |
| Validation | Pagination `page` defaults to 1, min 1 | product.schemas.ts:17 |
| Validation | Pagination `pageSize` defaults to 20, range 1-100 | product.schemas.ts:18 |
| Validation | `active` query filter is a strict string-literal `'true'/'false'`, transformed to boolean | product.schemas.ts:20-23 |
| Business Logic | SKU must be unique across all products on creation | product.service.ts:31-34 |
| Business Logic | SKU must remain unique across products on update (excluding self) | product.service.ts:47-52 |
| Business Logic | Product must exist to be fetched by ID | product.service.ts:24-28 |
| Business Logic | Product must exist to be updated | product.service.ts:46 |
| Business Logic | Product must exist to be deleted | product.service.ts:63-64 |
| Business Logic | Update applies partial/sparse field replacement (PATCH semantics) | product.service.ts:53-60 |
| Business Logic | List supports free-text search across `name` OR `sku` (substring match) | product.repository.ts:16-20 |
| Business Logic | List supports filtering by `active` status | product.repository.ts:15 |
| Business Logic | List pagination is computed from 1-based page number | product.service.ts:18 |
| Business Logic | Description input `undefined` is normalized to `null` on create | product.service.ts:38 |
| Authorization | All product endpoints require a valid authenticated user (any role) | product.routes.ts:14 |
| Cross-module | Order placement decrements product stockQuantity outside ProductService | order.service.ts:206-231 (external to this component) |
| Cross-module | Order cancellation restores product stockQuantity outside ProductService | order.service.ts:235-240 (external to this component) |
| Cross-module | Only active products may be added to an order | order.service.ts:66-71 (external to this component) |

### Detailed breakdown of the business rules

---

### Business Rule: SKU Uniqueness on Creation

**Overview**:
When a new product is created, its SKU (Stock Keeping Unit) must not already be in use by any other product in the catalog.

**Detailed description**:
The SKU is the canonical human/business-facing identifier for a product, distinct from the internal UUID primary key. Because SKUs are typically used by external systems, warehouse operations, or barcodes, the system must guarantee there is never more than one product sharing the same SKU. `ProductService.create` (product.service.ts:30-34) enforces this at the application layer by first calling `ProductRepository.findBySku(input.sku)` and checking whether a product with that SKU already exists before attempting to persist the new record. If a match is found, the service throws a `ConflictError('SKU already in use', 'SKU_ALREADY_USED')`, which the global error middleware maps to an HTTP 409 response with error code `SKU_ALREADY_USED`.

This application-level check exists in addition to (not instead of) the database-level uniqueness constraint defined in the Prisma schema (`sku String @unique @db.VarChar(64)`, prisma/schema.prisma:58). The application-level pre-check provides a friendlier, domain-specific error message and error code before the request reaches the database, improving the developer/API-consumer experience. However, because the check-then-write sequence is not wrapped in a database transaction or advisory lock, it is inherently subject to a time-of-check to time-of-use (TOCTOU) race condition: two concurrent create requests with the same SKU could both pass the `findBySku` check before either `create` call commits. In that scenario, the second write is not rejected by ProductService's own logic but rather by the database (Prisma's `P2002` unique-constraint violation), which is caught generically by `error.middleware.ts` and mapped to a slightly different response shape (`code: 'CONFLICT'`, generic message referencing the DB constraint target) rather than the `SKU_ALREADY_USED` code the service would have produced. This means API consumers may observe two different error payload shapes for what is conceptually the same failure, depending on request timing.

The rule directly protects data integrity for downstream consumers of the product catalog (notably the Orders module, which resolves order line items by `productId`, not `sku` — but external integrations or import/export processes may rely on SKU uniqueness as an implicit contract). Because SKU input is only length-validated (2-64 characters) at the schema layer and carries no format/pattern constraint, any string within that length range is accepted as a candidate SKU, placing the entire uniqueness burden on this business rule.

**Rule workflow**:
```
1. POST /api/v1/products with body containing `sku`
2. Zod schema validates sku length (2-64 chars) - schema-level, not uniqueness-aware
3. ProductService.create calls products.findBySku(sku)
4. IF a product with that sku exists:
   -> throw ConflictError('SKU already in use', 'SKU_ALREADY_USED') -> HTTP 409
5. ELSE:
   -> ProductRepository.create() persists the new row
   -> IF a race condition causes a duplicate to slip through, Prisma raises P2002
      -> error.middleware.ts maps it to HTTP 409 with a generic CONFLICT code
```

---

### Business Rule: SKU Uniqueness on Update (Excluding Self)

**Overview**:
When updating an existing product's SKU, the new SKU value must not collide with the SKU of any *other* product, but must be allowed to remain unchanged (i.e., a product can "update" its SKU to its own current value without triggering a conflict).

**Detailed description**:
`ProductService.update` (product.service.ts:47-52) only performs the uniqueness check when the update payload explicitly includes an `sku` field (`if (input.sku)`), consistent with the partial-update (PATCH) semantics of the endpoint. When present, it calls `ProductRepository.findBySku(input.sku)` to look up any product currently holding that SKU, and if one is found, it checks `sameSku.id !== id` to determine whether the conflicting record is a *different* product than the one being updated. If it is a different product, the service throws the same `ConflictError('SKU already in use', 'SKU_ALREADY_USED')` used during creation, keeping error semantics consistent between create and update flows.

This self-exclusion logic is essential: without it, any update request that included the product's own current SKU value (e.g., a client that always sends the full object back, even unchanged fields) would incorrectly be rejected as a conflict, since `findBySku` would find the record being updated itself. The `sameSku.id !== id` guard specifically distinguishes "this SKU belongs to me already" from "this SKU belongs to someone else," ensuring idempotent-style updates are not erroneously blocked.

Like the creation-time check, this rule is a check-then-act (TOCTOU) pattern without transactional or locking protection, meaning it shares the same race-condition characteristics described in the creation rule: concurrent updates targeting the same new SKU from two different products could both pass this check, deferring the actual conflict resolution to the database's unique constraint and the generic Prisma-error-handling path in `error.middleware.ts`. Additionally, because `input.sku` is only checked with a truthy condition (`if (input.sku)`), an update payload that explicitly sets `sku` to an empty string would bypass this check entirely if it somehow passed schema validation — though in practice the Zod schema's `min(2)` constraint (inherited via `.partial()` from `createProductSchema`) prevents an empty string from reaching this point when the field is present.

**Rule workflow**:
```
1. PATCH /api/v1/products/:id with body possibly containing `sku`
2. ProductService.update first calls this.getById(id) to confirm the target product exists
3. IF input.sku is truthy:
   a. ProductRepository.findBySku(input.sku)
   b. IF a matching product exists AND its id !== the product being updated:
      -> throw ConflictError('SKU already in use', 'SKU_ALREADY_USED') -> HTTP 409
   c. ELSE (no match, or match is the same product) -> proceed
4. IF input.sku is absent/falsy -> skip the uniqueness check entirely
5. Sparse update object is built and persisted via ProductRepository.update
```

---

### Business Rule: Product Existence Guard (Get, Update, Delete)

**Overview**:
Any operation that targets a specific product by ID (`getById`, `update`, `delete`) must first confirm the product exists; otherwise a domain-specific 404 error is raised before any further processing or mutation occurs.

**Detailed description**:
`ProductService.getById` (product.service.ts:24-28) is the foundational existence check used directly for read-by-ID requests, and is also reused internally by both `update` (product.service.ts:46) and `delete` (product.service.ts:63-64) as a pre-flight guard. This reuse (`await this.getById(id)`) is a deliberate code-reuse decision: rather than duplicating the "look up and throw NotFoundError" logic in three places, the service composes `update` and `delete` on top of `getById`, ensuring consistent error behavior (same error message format `"Product not found"`, same error code `NOT_FOUND`, same HTTP 404 status) across all three ID-targeted operations.

Functionally, this guard prevents Prisma-level errors from leaking through as opaque 500s or inconsistent error shapes when a client references a non-existent product ID. Without this check, `ProductRepository.update` and `ProductRepository.delete` would call Prisma's `update`/`delete` methods directly against a `where: { id }` clause with no matching row, which Prisma resolves as a `P2025` "Record not found" error — a case that *is* explicitly handled in `error.middleware.ts` (mapped to a generic 404), but with a less specific message (`"Resource not found"` rather than `"Product not found"`) and no domain context. By pre-checking existence in the service layer, `ProductService` ensures a more informative and consistent error contract for API consumers, independent of whatever the underlying ORM's error surface happens to be.

A side effect of this design is a minor performance/efficiency cost: `update` and `delete` each perform two database round-trips (one `findUnique` via `getById`, one `update`/`delete`) instead of one, and results of the initial `getById` fetch inside `update` are discarded rather than reused as a base for merging update fields. There is also a narrow TOCTOU window between the existence check and the subsequent write/delete, though this is much less consequential than the SKU race condition since the operation targets a specific already-known ID rather than a value being newly claimed.

**Rule workflow**:
```
1. Request targets a specific product by :id (GET /:id, PATCH /:id, DELETE /:id)
2. Schema validation confirms :id is a syntactically valid UUID (does not confirm existence)
3. ProductService.getById(id) [called directly, or internally by update/delete]:
   a. ProductRepository.findById(id) -> Prisma findUnique
   b. IF null -> throw NotFoundError('Product') -> HTTP 404, code NOT_FOUND,
      message "Product not found"
   c. ELSE -> return the found Product (used directly for GET; discarded for
      update/delete, which only use it as a guard)
4. For update/delete, execution proceeds to the respective mutation only if step 3
   did not throw
```

---

### Business Rule: Partial Update (PATCH) Semantics

**Overview**:
Update requests only modify the fields explicitly present (not `undefined`) in the request body; omitted fields retain their existing stored values.

**Detailed description**:
`ProductService.update` (product.service.ts:53-60) constructs the data object passed to `ProductRepository.update` using a series of conditional spread expressions: `...(input.field !== undefined ? { field: input.field } : {})`, repeated for `sku`, `name`, `description`, `priceCents`, `stockQuantity`, and `active`. This pattern ensures that the resulting Prisma update payload contains only the keys that the client actually supplied in the PATCH body, rather than overwriting every field with `undefined` (which, depending on Prisma's update semantics, could otherwise unset fields or trigger validation errors on required columns).

This behavior is enabled at the schema layer by `updateProductSchema = createProductSchema.partial()` (product.schemas.ts:14), which makes every field optional for update requests while still applying the same per-field validation rules (length limits, non-negative integer constraints, etc.) *when a field is provided*. The combination of a `.partial()` Zod schema and the conditional-spread building logic in the service is what gives the `PATCH /api/v1/products/:id` endpoint proper partial-update semantics, as opposed to `PUT`-style full-replacement semantics. This is an important distinction for API consumers: a client wishing to update only `stockQuantity` can send `{ "stockQuantity": 42 }` without needing to resend the product's `name`, `sku`, `priceCents`, etc.

One notable nuance is the handling of `description`: unlike `create` (where `input.description ?? null` normalizes an absent description to explicit `null`), the `update` path uses the `!== undefined` check uniformly for all fields including `description`. This means a client can explicitly send `"description": null` in a PATCH body to clear the description, and the field will be included in the update (since `null !== undefined`), correctly setting it to null in the database. If `description` is omitted from the PATCH body entirely, it is excluded from the spread and the existing value is preserved. This asymmetry between create-time normalization (`?? null`) and update-time pass-through (`!== undefined` check only) is intentional and consistent with typical PATCH semantics, but is a subtle implementation detail that could confuse a developer unfamiliar with the distinction between "field absent" and "field explicitly null."

**Rule workflow**:
```
1. PATCH /api/v1/products/:id with a body containing zero or more of:
   sku, name, description, priceCents, stockQuantity, active
2. Zod partial schema validates only the fields present (each still subject to its
   individual constraints: length, non-negativity, etc.)
3. ProductService.update builds a sparse update object:
   FOR EACH field in [sku, name, description, priceCents, stockQuantity, active]:
     IF input.field !== undefined -> include { field: input.field } in the update payload
     ELSE -> omit the field entirely (existing DB value is preserved)
4. ProductRepository.update(id, sparseData) persists only the included fields via Prisma
```

---

### Business Rule: Product List Filtering and Search

**Overview**:
The product listing endpoint supports optional filtering by `active` status and free-text search matching against either the product `name` or `sku` fields, combined with page-based pagination.

**Detailed description**:
`ProductService.list` (product.service.ts:14-22) accepts a validated `ListProductsQuery` (page, pageSize, search, active) and translates the 1-based `page`/`pageSize` pair into Prisma's `skip`/`take` offset-pagination primitives via `skip = (page - 1) * pageSize`. It delegates the actual filtering logic to `ProductRepository.list`, which in turn calls the private `buildWhere` helper (product.repository.ts:13-23) to construct a Prisma `WhereInput`. If an `active` boolean is present in the filters, an equality condition (`where.active = filters.active`) is applied; if a `search` string is present, an `OR` condition is added matching `name` contains search OR `sku` contains search (case sensitivity is governed by the underlying database collation, since no explicit `mode: 'insensitive'` option is set in the Prisma query).

The `active` filter's query-string handling is worth noting at the schema layer: `listProductsQuerySchema` (product.schemas.ts:20-23) accepts only the literal strings `'true'` or `'false'` (not arbitrary truthy/falsy values) and transforms them into a boolean via `.transform((v) => v === 'true')`. Any other string value for the `active` query parameter (e.g., `?active=1` or `?active=yes`) would fail Zod validation and result in a 400 `VALIDATION_ERROR` response rather than being silently coerced. This is a deliberate strictness choice that avoids ambiguous boolean parsing from query strings.

Pagination metadata for the response is computed by the shared `paginated`/`buildPagination` helpers (shared/http/response.ts), which calculate `totalPages` as `Math.ceil(total / pageSize)`, with a defensive guard returning 0 when `pageSize` is 0 (avoiding a division-by-zero / Infinity result), though in practice `pageSize` cannot be 0 given the schema's `min(1)` constraint. The repository executes the `findMany` and `count` queries together inside a single `prisma.$transaction([...])` call (product.repository.ts:27-35), ensuring the returned `items` and `total` are consistent with each other as of the same transactional snapshot, which matters for correctness under concurrent writes (e.g., avoiding a `total` count that reflects a product inserted after the `findMany` page was already fetched).

**Rule workflow**:
```
1. GET /api/v1/products?page=&pageSize=&search=&active=
2. Zod query schema validates/coerces: page (default 1, min 1), pageSize (default 20,
   1-100), search (optional, trimmed, min 1 char), active ('true'/'false' -> boolean,
   optional)
3. ProductService.list computes skip = (page - 1) * pageSize, take = pageSize
4. ProductRepository.buildWhere constructs:
   - active filter (equality) IF active is defined
   - OR[name contains search, sku contains search] IF search is defined
5. ProductRepository.list executes findMany(where, orderBy createdAt desc, skip, take)
   and count(where) inside a single Prisma transaction
6. ProductService wraps { items, total } via paginated() into
   { data, pagination: { page, pageSize, total, totalPages } }
```

---

### Business Rule: Authenticated Access Only (No Role Restriction)

**Overview**:
All `/products` endpoints require a valid, authenticated JWT bearer token, but do not restrict access based on user role — both `ADMIN` and `OPERATOR` roles have identical permissions across all product operations, including create, update, and delete.

**Detailed description**:
`product.routes.ts:14` applies `router.use(authenticate)` to the entire product router, meaning every route (`list`, `getById`, `create`, `update`, `delete`) requires a valid JWT in the `Authorization: Bearer <token>` header, verified against `env.JWT_SECRET` (auth.middleware.ts). The `authenticate` middleware populates `req.user` with `{ id, email, role }` extracted from the JWT payload, but the products router never calls `requireRole(...)` on any of its routes — unlike patterns available elsewhere in the codebase (`requireRole` is exported from `auth.middleware.ts` specifically to support role-gating on top of `authenticate`).

This means that any authenticated user, regardless of whether their role is `ADMIN` or `OPERATOR`, can create, modify, or delete products in the catalog. For a business domain where product catalog changes (pricing, SKU management, deactivation) are often considered administrative/privileged actions, this is a notable authorization design choice — it may be intentional (e.g., both roles are trusted internal staff with equal catalog responsibilities) or it may be an oversight relative to stricter patterns applied elsewhere in the system. Without additional context or an architecture-level authorization matrix, this analysis treats it as an observed fact with an "implicit, unconfirmed" business rule confidence level: it is unclear from the code alone whether uniform access is the intended design or a gap.

The practical effect is that the component's authorization surface is entirely binary (authenticated vs. not), with all fine-grained authorization logic (if any exists elsewhere in the system) not present within the ProductService/product module boundary. This is consistent with the "coarse-grained gate at the router, business logic in the service" separation of concerns pattern also seen in other modules, but the products module's specific choice to omit `requireRole` should be verified against actual business requirements if role-based catalog restrictions are expected.

**Rule workflow**:
```
1. Any request to /api/v1/products/* passes through router.use(authenticate)
2. authenticate middleware:
   a. Requires "Authorization: Bearer <token>" header -> else 401 UnauthorizedError
   b. Verifies JWT against env.JWT_SECRET -> else 401 UnauthorizedError
   c. Populates req.user = { id, email, role } from JWT payload
3. NO requireRole(...) check is applied for ANY product route
4. Request proceeds to the corresponding ProductController method regardless of
   req.user.role value (ADMIN or OPERATOR both pass)
```

---

## 4. Component Structure

```
src/modules/products/
├── product.routes.ts       # Express router wiring: applies `authenticate` middleware
│                            #   globally, applies `validate()` per-route with Zod schemas,
│                            #   maps HTTP verbs/paths to ProductController methods
├── product.controller.ts   # HTTP boundary: extracts req.params/req.query/req.body,
│                            #   delegates to ProductService, maps results to
│                            #   res.status(...).json(...)/.send(), forwards errors via next(err)
├── product.service.ts      # ANALYZED COMPONENT - business logic: SKU uniqueness,
│                            #   existence guards, pagination translation, partial-update
│                            #   composition
├── product.repository.ts   # Persistence boundary: wraps PrismaClient product model
│                            #   access (list/findById/findBySku/findManyByIds/create/
│                            #   update/delete), builds Prisma WhereInput filters
└── product.schemas.ts      # Zod schemas + inferred TypeScript types for request
                             #   validation: productIdParamSchema, createProductSchema,
                             #   updateProductSchema (partial), listProductsQuerySchema
```

Directly related files outside the module directory that form the component's immediate boundary:

```
src/
├── app.ts                              # Composition root: instantiates ProductRepository,
│                                        #   ProductService, ProductController and wires
│                                        #   dependencies (manual DI, no framework)
├── routes/index.ts                     # Mounts buildProductRouter(controllers.products)
│                                        #   at /api/v1/products
├── middlewares/
│   ├── auth.middleware.ts              # `authenticate` (JWT check) used by product.routes.ts;
│   │                                   #   also exports unused-by-this-module `requireRole`
│   └── validate.middleware.ts          # Generic Zod-based request validator used per-route
├── shared/
│   ├── errors/
│   │   ├── app-error.ts                # Base AppError class (statusCode, errorCode, details)
│   │   ├── http-errors.ts              # NotFoundError, ConflictError (used by ProductService)
│   │   └── index.ts                    # Re-exports
│   └── http/response.ts                # paginated()/buildPagination() helpers used by
│                                        #   ProductService.list
prisma/schema.prisma                    # Product model definition (Prisma schema),
                                         #   referenced indirectly via @prisma/client types
```

Note: `src/modules/orders/order.service.ts` also reads/writes the `Product` table directly via `PrismaClient`/transaction (`tx.product.*`), completely bypassing `ProductRepository`/`ProductService`. This file is outside the products module directory and is not a direct code dependency of `ProductService`, but it is a significant data-boundary interaction worth noting (see Section 5 and Section 10).

---

## 5. Dependency Analysis

```
Internal Dependencies (compile-time imports):

product.routes.ts
  -> auth.middleware.ts (authenticate)
  -> validate.middleware.ts (validate)
  -> product.schemas.ts (createProductSchema, updateProductSchema, productIdParamSchema,
     listProductsQuerySchema)
  -> product.controller.ts (ProductController, type-only)

product.controller.ts
  -> product.service.ts (ProductService, type-only)
  -> product.schemas.ts (ListProductsQuery, type-only)

product.service.ts  [ANALYZED COMPONENT]
  -> product.repository.ts (ProductRepository, type-only)
  -> shared/errors/index.ts (ConflictError, NotFoundError)
  -> product.schemas.ts (CreateProductInput, ListProductsQuery, UpdateProductInput, type-only)
  -> shared/http/response.ts (paginated, PaginatedResponse)
  -> @prisma/client (Product, type-only)

product.repository.ts
  -> @prisma/client (Prisma, PrismaClient, Product, type-only + runtime via injected instance)

app.ts (composition root)
  -> instantiates: ProductRepository(prisma) -> ProductService(productRepository) ->
     ProductController(productService)

routes/index.ts
  -> product.routes.ts (buildProductRouter)


External Dependencies:

- express (^4.x, inferred from RequestHandler/Router types) - HTTP routing/handling
  (used by controller/routes, NOT by ProductService directly)
- zod - Schema validation and type inference (product.schemas.ts)
- @prisma/client / Prisma ORM - Database access and generated types (Product, Prisma
  namespace) - used directly by ProductRepository, referenced type-only by ProductService
- PostgreSQL/MySQL-compatible relational database (inferred from prisma/schema.prisma
  @db.Char, @db.VarChar, @db.Text column type annotations) - underlying persistence store
- jsonwebtoken - JWT verification in auth.middleware.ts (indirect dependency via routing
  layer, not used by ProductService itself)


Cross-Module (Data-layer) Coupling Outside Direct Imports:

- src/modules/orders/order.service.ts reads and mutates the `products` table directly
  through PrismaClient/transaction calls (tx.product.findMany, tx.product.update),
  independent of ProductRepository/ProductService. This is a data-level dependency
  (shared table) rather than a code-level import dependency, but it means stock-related
  business rules for products are implemented outside the ProductService component.
```

---

## 6. Afferent and Efferent Coupling

Coupling is measured at the class level (TypeScript classes/exported functions), consistent with the object-oriented structure of this Node.js/TypeScript codebase.

| Component | Afferent Coupling (incoming) | Efferent Coupling (outgoing) | Critical |
|-----------|-------------------------------|-------------------------------|----------|
| ProductService | 2 (ProductController, app.ts composition root) | 3 (ProductRepository, shared/errors, shared/http/response) | High |
| ProductRepository | 2 (ProductService, app.ts composition root) | 1 (PrismaClient / @prisma/client) | High |
| ProductController | 2 (product.routes.ts, app.ts composition root) | 1 (ProductService) | Medium |
| product.schemas.ts (schemas/types) | 4 (product.routes.ts, product.controller.ts, product.service.ts, tests/helpers factories indirectly via Prisma types) | 1 (zod) | Medium |
| buildProductRouter (product.routes.ts) | 1 (routes/index.ts) | 3 (auth.middleware, validate.middleware, product.schemas.ts) | Low |

Notes:
- "Critical" reflects the blast radius if the component's public contract changes: `ProductService` and `ProductRepository` are rated High because they sit on the sole path through which product data reaches the HTTP layer, and `ProductService`'s method signatures are consumed directly by `ProductController` with no adapter/anti-corruption layer in between.
- Afferent coupling counts are low in absolute terms because this is a small, self-contained module; the more significant coupling risk is the data-level (table-sharing) coupling from `OrderService` bypassing this component entirely (see Section 5), which is not captured by import-based afferent/efferent counts but is architecturally significant.

---

## 7. Endpoints

All endpoints are mounted under `/api/v1/products` (see src/routes/index.ts, src/app.ts) and require authentication (no role restriction — see Business Rule "Authenticated Access Only").

| Endpoint | Method | Description | Auth | Request Validation |
|----------|--------|-------------|------|---------------------|
| /api/v1/products | GET | List products with pagination, optional `search` and `active` filters | Bearer JWT (any role) | query: listProductsQuerySchema |
| /api/v1/products/:id | GET | Get a single product by UUID | Bearer JWT (any role) | params: productIdParamSchema |
| /api/v1/products | POST | Create a new product (enforces unique SKU) | Bearer JWT (any role) | body: createProductSchema |
| /api/v1/products/:id | PATCH | Partially update a product (enforces unique SKU if changed) | Bearer JWT (any role) | params: productIdParamSchema, body: updateProductSchema (partial) |
| /api/v1/products/:id | DELETE | Delete a product | Bearer JWT (any role) | params: productIdParamSchema |

Response shapes:
- `GET /products` -> 200, `{ data: Product[], pagination: { page, pageSize, total, totalPages } }`
- `GET /products/:id` -> 200, `Product` object, or 404 `{ error: { code: 'NOT_FOUND', message: 'Product not found' } }`
- `POST /products` -> 201, created `Product`, or 409 `{ error: { code: 'SKU_ALREADY_USED', message: 'SKU already in use' } }`
- `PATCH /products/:id` -> 200, updated `Product`, or 404/409 as above
- `DELETE /products/:id` -> 204 No Content, or 404 as above

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|-----------------|
| PostgreSQL/relational DB (via Prisma) | Internal Datastore | Persistence of product catalog (CRUD, search, pagination) | Prisma Client (SQL under the hood) | Relational rows mapped to typed `Product` objects | Prisma errors (P2002 unique constraint, P2025 not found) caught centrally in error.middleware.ts; ProductService adds a pre-emptive application-level check for SKU uniqueness and existence to produce friendlier domain errors before hitting the DB constraint |
| Orders module (data-level, not code-level) | Internal Service (implicit) | OrderService reads Product rows to validate order items, price snapshotting, and stock adjustment | Direct PrismaClient transaction calls (tx.product.*), bypassing ProductRepository/ProductService | Prisma-typed Product rows | Handled entirely within order.service.ts (InsufficientStockError, NotFoundError, UnprocessableEntityError); no interaction with ProductService's own error paths |
| Auth subsystem (JWT) | Internal Middleware | Authenticates all requests to the products router | HTTP header (Authorization: Bearer) | JWT (jsonwebtoken) | UnauthorizedError (401) on missing/invalid/expired token, handled by error.middleware.ts |
| Zod validation layer | Internal Middleware | Validates/coerces request body, query, and params for every product route | In-process function calls | JS objects | ZodError caught by validate.middleware.ts, converted to ValidationError (400) with per-field details |

ProductService itself has no direct external (third-party/network) integrations; all of its integration points are mediated through `ProductRepository` (database) and consumed by `ProductController` (HTTP).

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered Architecture (Controller-Service-Repository) | ProductController -> ProductService -> ProductRepository | product.controller.ts, product.service.ts, product.repository.ts | Separation of HTTP concerns, business logic, and persistence concerns |
| Repository Pattern | ProductRepository wraps all Prisma access for the Product entity | product.repository.ts | Abstracts persistence details and query construction (buildWhere) from the service layer |
| Dependency Injection (manual constructor injection) | ProductService(products: ProductRepository), ProductController(products: ProductService) | product.service.ts:12, product.controller.ts:6; wired in app.ts:38-40 | Testability (repository can be mocked/stubbed) and decoupling from concrete instantiation |
| DTO / Schema-derived typing | Zod schemas define both runtime validation and compile-time types via z.infer | product.schemas.ts:26-28 | Single source of truth for both validation rules and TypeScript input types (CreateProductInput, UpdateProductInput, ListProductsQuery) |
| Guard Clause / Fail-Fast | Existence and uniqueness checks throw immediately before proceeding | product.service.ts:26, 33-34, 46, 50-51, 64 | Prevents invalid states from reaching the persistence layer; produces predictable domain errors early |
| Centralized Error Handling | AppError hierarchy + errorMiddleware translates domain and infra errors uniformly | shared/errors/*, src/middlewares/error.middleware.ts | Consistent HTTP error response shape across all modules, including Product |
| Sparse/Partial Update Composition | Conditional spread operators build only-changed-fields update payload | product.service.ts:53-59 | Implements true PATCH semantics without overwriting unspecified fields |
| Method Reuse via Internal Composition | update/delete call this.getById(id) internally | product.service.ts:46, 64 | DRY existence-check logic, consistent NotFoundError behavior across operations |
| Middleware Pipeline Pattern | authenticate -> validate -> controller handler chain | product.routes.ts | Standard Express cross-cutting concern composition (auth, then validation, then business logic) |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|-----------------|-------|--------|
| High | Test Coverage | No dedicated unit or integration tests exist for ProductService or the /products endpoints anywhere in the repository (only indirect use via createTestProduct factory in tests/orders.test.ts) | Regressions in SKU uniqueness, existence guards, partial-update logic, or pagination could go undetected; no safety net for refactors |
| Medium | ProductService.create / update | SKU uniqueness check-then-act pattern (findBySku then create/update) is not race-condition-safe; relies on the DB unique constraint and generic error-middleware P2002 handling as a fallback, producing an inconsistent error response shape (SKU_ALREADY_USED vs generic CONFLICT with DB-target message) depending on timing | Under concurrent writes, API consumers may receive different error codes/messages for the same logical conflict, complicating client-side error handling |
| Medium | Cross-module coupling | OrderService (outside this component) mutates the same `products` table directly via raw Prisma transaction calls, bypassing ProductRepository/ProductService entirely; stock adjustment business rules for products live in the Orders module rather than the Products module | Business logic for a single entity (Product) is split across two modules; changes to stock semantics must be coordinated in a file outside the ProductService boundary, increasing risk of inconsistent invariant enforcement (e.g., ProductService.update could set stockQuantity in a way that conflicts with concurrent order-driven stock mutations, with no shared locking/transaction strategy between the two code paths) |
| Medium | Authorization | No requireRole restriction on any /products route; both ADMIN and OPERATOR can create/update/delete products, despite requireRole being available and presumably used elsewhere for privileged actions | Potential over-permissioning if catalog management is intended to be an admin-only capability; unclear whether this is by design |
| Low | ProductService.delete | Hard delete with no explicit handling of foreign-key conflicts (e.g., a product referenced by existing OrderItem rows); error.middleware.ts only special-cases P2002 and P2025, so an FK violation on delete (if not represented as P2025) could surface as a generic 500 rather than a domain-appropriate 409/422 | Deleting a product that is referenced by historical orders could produce an unclear 500 error instead of a clear conflict message; also raises a data-integrity question of whether products should ever be hard-deleted vs. soft-deleted via the existing `active` flag |
| Low | ProductService.update / delete | Two sequential DB round-trips (getById, then update/delete) instead of a single conditional operation; the Product fetched by the internal getById() call in update() is discarded rather than being reused to short-circuit unnecessary work | Minor performance inefficiency; not a correctness issue, but adds latency and a (low-severity) TOCTOU window between the check and the write |
| Low | product.schemas.ts | SKU field has no format/pattern constraint beyond length (2-64 chars); any string qualifies as a valid SKU | Could allow inconsistent SKU formats (whitespace-only-looking strings, special characters) into the catalog; no normalization (e.g., trimming, uppercasing) is applied before uniqueness checks, so "ABC-1 " and "ABC-1" (with trailing space) would be treated as distinct SKUs |
| Low | product.repository.ts search | Free-text search uses Prisma `contains` without an explicit case-insensitivity mode; matching behavior depends on the database's default collation, which is not verified within this codebase | Search behavior could be inconsistent across different database configurations/environments (case-sensitive vs. case-insensitive) |

---

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|---------------------|----------|---------------|
| ProductService | 0 | 0 | None (0%) — no dedicated test file exists (e.g., no products.test.ts or product.service.spec.ts anywhere in the repository) | N/A — untested |
| ProductController / /products endpoints | 0 | 0 direct | None (0%) — no HTTP-level tests target /api/v1/products routes directly | N/A — untested |
| ProductRepository | 0 | 0 direct | None (0%) — no direct tests of repository query-building (buildWhere) or CRUD wrapper methods | N/A — untested |
| product.schemas.ts (Zod schemas) | 0 | 0 | None (0%) — no schema-level validation tests | N/A — untested |
| Product entity (indirect, via Orders tests) | 0 | Indirect only, via tests/orders.test.ts | Partial/incidental — createTestProduct (tests/helpers/factories.ts:65-77) is used as a fixture-setup helper in tests/orders.test.ts to create products consumed by order-related assertions (e.g., verifying stockQuantity changes after order placement/payment/cancellation) | The indirect coverage exercises Product creation via direct Prisma calls (not via ProductService/API), and exercises stockQuantity mutation via OrderService's direct Prisma transaction calls — it does not exercise ProductService.create, update, delete, list, or getById, nor the SKU-uniqueness or existence-guard business rules documented in Section 3 |

Test infrastructure observed in the repository (for context, not specific to Products):
- Test files located at repository root: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`.
- `tests/helpers/factories.ts` provides `getTestApp()` (builds the full Express app via `buildApp`), `createTestUser`, `loginAndGetToken`, `bootstrapAuthenticatedUser`, `createTestCustomer`, and `createTestProduct` — indicating the test infrastructure (supertest-based, real Prisma-backed database) is fully capable of supporting a dedicated `products.test.ts` file analogous to the existing `orders.test.ts`, but no such file has been authored.
- No unit-test-style mocking of `ProductRepository` was found (e.g., no `product.service.spec.ts` using a stub/mock repository), meaning even isolated (non-HTTP, non-DB) unit tests of the business rules in `ProductService` (SKU uniqueness, existence guards, partial update composition, pagination math) do not exist.

**Summary**: Test coverage for the ProductService component and its entire module boundary (controller, repository, schemas, routes) is effectively zero in terms of direct, purpose-built tests. The only coverage is incidental, arising from the Orders module's tests using product fixtures and directly manipulating the products table through Prisma (not through ProductService). This represents the most significant risk identified in this analysis (see Section 10, High severity item).

---

*End of report.*
