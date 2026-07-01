# Component Deep Analysis Report: ProductController

**Project:** order-management-api (Order Management System REST API)
**Component analyzed:** `ProductController`
**Primary file:** `src/modules/products/product.controller.ts`
**Analysis date:** 2026-06-30

---

## 1. Executive Summary

`ProductController` is the HTTP entry point for product catalog management in the Order Management System REST API. It is a thin Express-based controller (not NestJS, despite the naming convention — the stack is Node.js + TypeScript + Express 4 + Prisma 5 + Zod 3) that exposes five CRUD-style REST endpoints for managing `Product` entities: list (paginated/filterable), get by id, create, update (partial), and delete.

The controller itself contains no business logic. It is a pure adapter: it extracts already-validated data from the Express `Request` object (`req.query`, `req.params`, `req.body` — validated upstream by Zod schemas via the `validate` middleware), delegates to `ProductService` for all business rules and persistence orchestration, and maps the resolved value or thrown error onto an HTTP response using a uniform `try/catch { ... } catch (err) { next(err) }` pattern. All five handlers follow the exact same shape, differing only in the service method invoked, the request fields consumed, and the response status code (200 for read/update/list, 201 for create, 204 for delete).

Because the controller delegates validation to route-level middleware (`validate()` + Zod schemas) and business logic to `ProductService`, its own cyclomatic complexity and defect surface are very low. The component's real complexity lives in `ProductService` (SKU uniqueness checks, partial update semantics) and in `product.schemas.ts` (input validation rules: string lengths, non-negative numeric constraints, UUID format for ids, pagination bounds). Key findings: (1) the controller has no direct automated test coverage — Product is only exercised indirectly through Order integration tests via `createTestProduct`; (2) authorization on product routes is coarse-grained (any authenticated user, ADMIN or OPERATOR, can create/update/delete products — no `requireRole` guard is applied, unlike the pattern available in `auth.middleware.ts`); (3) the component follows a clean layered architecture (Controller → Service → Repository → Prisma) with consistent error handling and pagination conventions shared across the codebase.

---

## 2. Data Flow Analysis

### 2.1 `GET /api/v1/products` (list)

```
1. Request enters Express app (src/app.ts:59-73) — JSON body parser, request logger
2. Router dispatch: /api/v1 -> /products (src/routes/index.ts:27) -> buildProductRouter (product.routes.ts:16)
3. authenticate middleware (product.routes.ts:14 / auth.middleware.ts:26-46) — verifies Bearer JWT, populates req.user
4. validate({ query: listProductsQuerySchema }) (product.routes.ts:16) — Zod-parses/coerces page, pageSize, search, active; merges parsed values back into req.query
5. ProductController.list handler invoked (product.controller.ts:8-16)
6. Controller casts req.query to ListProductsQuery and calls ProductService.list(query) (product.service.ts:14-22)
7. ProductService builds repository filter object (search, active, skip = (page-1)*pageSize, take = pageSize)
8. ProductRepository.list (product.repository.ts:25-37) builds a Prisma "where" clause (buildWhere, lines 13-23) and runs a Prisma $transaction combining findMany + count
9. ProductService wraps items+total into a PaginatedResponse via paginated()/buildPagination() (shared/http/response.ts)
10. Controller sends res.status(200).json(result)
11. On any thrown error at any layer, control passes to next(err) -> errorMiddleware (middlewares/error.middleware.ts) formats the HTTP error response
```

### 2.2 `GET /api/v1/products/:id` (getById)

```
1. authenticate -> validate({ params: productIdParamSchema }) (UUID format check)
2. ProductController.getById (product.controller.ts:18-25) calls ProductService.getById(id)
3. ProductService.getById (product.service.ts:24-28) calls ProductRepository.findById -> Prisma findUnique
4. If null, throws NotFoundError('Product') (404) -> caught by controller's try/catch -> next(err) -> errorMiddleware
5. If found, controller returns res.status(200).json(product)
```

### 2.3 `POST /api/v1/products` (create)

```
1. authenticate -> validate({ body: createProductSchema }) — enforces sku/name length, non-negative price/stock, defaults for stockQuantity/active
2. ProductController.create (product.controller.ts:27-34) calls ProductService.create(req.body)
3. ProductService.create (product.service.ts:30-43):
   a. Looks up existing product by SKU via ProductRepository.findBySku
   b. If found, throws ConflictError('SKU already in use', 'SKU_ALREADY_USED') (409)
   c. Otherwise calls ProductRepository.create with normalized data (description defaults to null when absent)
4. ProductRepository.create -> Prisma product.create
5. Controller responds res.status(201).json(created)
```

### 2.4 `PATCH /api/v1/products/:id` (update)

```
1. authenticate -> validate({ params: productIdParamSchema, body: updateProductSchema }) — body schema is createProductSchema.partial(), all fields optional
2. ProductController.update (product.controller.ts:36-43) calls ProductService.update(id, req.body)
3. ProductService.update (product.service.ts:45-61):
   a. Calls this.getById(id) to assert existence (throws NotFoundError if missing) — result discarded
   b. If input.sku provided, checks findBySku for a conflicting product with a different id -> ConflictError if found
   c. Builds a partial update payload including only defined fields (undefined-guarded spreads)
   d. Calls ProductRepository.update
4. Controller responds res.status(200).json(updated)
```

### 2.5 `DELETE /api/v1/products/:id` (delete)

```
1. authenticate -> validate({ params: productIdParamSchema })
2. ProductController.delete (product.controller.ts:45-52) calls ProductService.delete(id)
3. ProductService.delete (product.service.ts:63-66): calls getById(id) to assert existence (NotFoundError if missing), then ProductRepository.delete
4. Controller responds res.status(204).send() with an empty body
```

Note: `ProductRepository.delete` performs a hard delete (`prisma.product.delete`). Referential integrity with `OrderItem` (which references `Product`) is not checked at the controller/service/repository level within this component; any FK constraint violation would surface as a Prisma error and be handled generically by `errorMiddleware` (not as a domain-specific error).

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authentication | All product endpoints require a valid Bearer JWT | `product.routes.ts:14`, `middlewares/auth.middleware.ts:26-46` |
| Authorization | No role restriction — any authenticated ADMIN or OPERATOR can perform all CRUD operations | `product.routes.ts:14-24` (absence of `requireRole`) |
| Validation | `sku` must be 2-64 characters | `product.schemas.ts:6` |
| Validation | `name` must be 2-200 characters | `product.schemas.ts:7` |
| Validation | `description` optional, max 2000 characters | `product.schemas.ts:8` |
| Validation | `priceCents` must be a non-negative integer | `product.schemas.ts:9` |
| Validation | `stockQuantity` must be a non-negative integer, defaults to 0 | `product.schemas.ts:10` |
| Validation | `active` defaults to `true` if not provided on create | `product.schemas.ts:11` |
| Validation | `id` route param must be a valid UUID | `product.schemas.ts:3` |
| Validation | List `page` defaults to 1, minimum 1 | `product.schemas.ts:17` |
| Validation | List `pageSize` defaults to 20, min 1, max 100 | `product.schemas.ts:18` |
| Validation | List `search` must be a non-empty trimmed string when provided | `product.schemas.ts:19` |
| Validation | List `active` filter accepts only the literal strings `'true'`/`'false'`, coerced to boolean | `product.schemas.ts:20-23` |
| Business Logic | SKU must be unique across all products | `product.service.ts:31-34` (create), `product.service.ts:47-52` (update) |
| Business Logic | Product must exist before update or delete | `product.service.ts:46`, `product.service.ts:64` |
| Business Logic | Update is a true partial update (PATCH semantics) — only supplied fields are changed | `product.service.ts:53-60` |
| Business Logic | Search filter matches against name OR sku using substring containment | `product.repository.ts:16-20` |
| Business Logic | Pagination is computed via `skip = (page - 1) * pageSize`, `take = pageSize` | `product.service.ts:18-19` |
| Business Logic | List results are ordered by `createdAt` descending (newest first) | `product.repository.ts:30` |
| Business Logic | List query and count run inside a single Prisma transaction for consistency | `product.repository.ts:27-35` |
| Business Logic | `description` normalized to `null` (not `undefined`) when absent on create | `product.service.ts:38` |
| Cross-component rule | Products referenced by `OrderItem` are not protected from deletion at this layer (hard delete) | `product.repository.ts:59-61` |
| Cross-component rule | `active` flag governs whether a product is orderable, enforced by `OrderService`, not `ProductController`/`ProductService` | `modules/orders/order.service.ts:66-71` (external to this component) |

### Detailed breakdown of the business rules

---

### Business Rule: SKU Uniqueness Enforcement

**Overview:**
Every product must have a globally unique SKU (Stock Keeping Unit). This is enforced both at the application layer (explicit lookup-before-write in `ProductService`) and at the database layer (a `@unique` constraint on the `sku` column in the Prisma schema, `prisma/schema.prisma:58`).

**Detailed description:**
When a client submits a `POST /api/v1/products` request, `ProductService.create` (product.service.ts:30-43) first performs a `findBySku` lookup against the repository before attempting to persist the new record. If a product with the same SKU already exists, the service throws a `ConflictError` with the message "SKU already in use" and error code `SKU_ALREADY_USED`, which the `errorMiddleware` converts into an HTTP 409 response. This pre-check exists purely as a fast, application-level, user-friendly rejection path — it is not the sole safety net, because the underlying database column also carries a unique constraint (`@unique` in the Prisma model), meaning a raced concurrent insert with the same SKU would still be rejected at the database layer (surfacing as a Prisma `P2002` error, which `errorMiddleware` (error.middleware.ts:37-47) separately catches and converts to a generic 409 "Unique constraint violation" response, distinct in shape/code from the application-level `ConflictError`).

The same rule is re-applied during updates. `ProductService.update` (product.service.ts:45-61) only performs the SKU-uniqueness check when the caller actually supplies a `sku` field in the PATCH body (`if (input.sku)`), since the schema treats all update fields as optional. When a new SKU is supplied, the service looks up any existing product with that SKU and, critically, excludes the current record from the conflict check (`sameSku.id !== id`) — this allows a client to "update" a product while resending its own unchanged SKU without triggering a false-positive conflict. Only when a different product owns that SKU does the operation get rejected with the same `ConflictError`/`SKU_ALREADY_USED` semantics used on create.

This rule matters for the broader Order Management domain because SKU is the natural/business key used for external references (barcodes, supplier catalogs, customer-facing documentation), while `id` (a UUID) is the internal primary key used for relational integrity (e.g., `OrderItem.productId`). Divergence between these two identifier systems means the uniqueness rule specifically protects against ambiguous or colliding external references, independent of the guaranteed-unique internal UUID. There is a subtle race window between the `findBySku` check and the subsequent `create`/`update` call (the check and the write are not wrapped in a single transaction or `SELECT ... FOR UPDATE`), which is why the database-level unique constraint remains the authoritative enforcement mechanism, with the service-level check acting only as an optimization for the common case (returning a clean domain error rather than leaking a raw database constraint error to most callers).

**Rule workflow:**
```
CREATE:
  input.sku received
    -> findBySku(input.sku)
       -> if found: throw ConflictError("SKU already in use", "SKU_ALREADY_USED") [HTTP 409]
       -> if not found: proceed to repository.create(...)
    -> [race fallback] DB unique constraint violation -> Prisma P2002 -> errorMiddleware -> HTTP 409 "Unique constraint violation"

UPDATE:
  input.sku present in PATCH body?
    -> no: skip SKU uniqueness check entirely
    -> yes: findBySku(input.sku)
         -> found AND found.id !== target id: throw ConflictError [HTTP 409]
         -> found AND found.id === target id: no conflict (self-match), proceed
         -> not found: proceed to repository.update(...)
```

---

### Business Rule: Existence Verification Before Mutation (Update/Delete)

**Overview:**
Both `update` and `delete` operations require the target product to already exist; otherwise, the operation is rejected with a 404 `NotFoundError` before any mutating database call is attempted.

**Detailed description:**
`ProductService.update` (product.service.ts:45-61) begins by calling `await this.getById(id)` (line 46), and `ProductService.delete` (product.service.ts:63-66) does the same (line 64). `getById` (product.service.ts:24-28) itself performs a `findById` lookup and throws `NotFoundError('Product')` — which `AppError`/`http-errors.ts:27-31` maps to HTTP 404 with the message "Product not found" and error code `NOT_FOUND` — if the repository returns `null`. Notably, in both `update` and `delete`, the return value of this existence check is intentionally discarded (`await this.getById(id);` with no assignment); it is used purely as a guard clause, not as a data source for the subsequent write.

This pattern provides two benefits over relying solely on Prisma's own "record not found" errors from `update`/`delete` calls. First, it gives the API a single, consistent error shape (`NotFoundError` -> `{ error: { code: 'NOT_FOUND', message: 'Product not found' } }`) regardless of which underlying Prisma operation would have failed, rather than depending on Prisma's `P2025` "Record not found" error code being separately caught in `errorMiddleware` (error.middleware.ts:48-53) for every possible operation. Second, and more subtly, it establishes a clear invariant for the rest of the service method: by the time the SKU-conflict check or the field-merging logic executes, the caller can assume the record exists, simplifying reasoning about the remaining code path. The cost of this pattern is an extra round-trip to the database (a `findUnique` before the actual `update`/`delete`), which is a minor performance trade-off favoring correctness/consistency of error semantics over minimizing query count.

This existence check is also implicitly relied upon by the SKU-uniqueness rule during update: because `getById` runs first and throws before any SKU checks occur, a request that supplies both a non-existent `id` and a conflicting `sku` will always surface as `NOT_FOUND` rather than `SKU_ALREADY_USED`, establishing a deterministic error-precedence order for API consumers.

**Rule workflow:**
```
UPDATE or DELETE product(id):
  getById(id)
    -> findById(id) [Prisma findUnique]
       -> null: throw NotFoundError('Product') [HTTP 404, code NOT_FOUND] -- operation aborted here
       -> found: continue
  (UPDATE only) proceed to SKU-uniqueness check, then repository.update(id, partialData)
  (DELETE only) proceed directly to repository.delete(id)
```

---

### Business Rule: Partial Update Semantics (PATCH Field Merging)

**Overview:**
The update endpoint implements true PATCH semantics: only fields explicitly present in the request body are modified; omitted fields retain their existing stored values.

**Detailed description:**
The route is registered as `router.patch('/:id', ...)` (product.routes.ts:19-23), and its body is validated against `updateProductSchema`, which is defined as `createProductSchema.partial()` (product.schemas.ts:14) — every field from the create schema (`sku`, `name`, `description`, `priceCents`, `stockQuantity`, `active`) becomes optional for updates, with no defaults applied (unlike the create schema, which defaults `stockQuantity` to 0 and `active` to true). Because Zod's `.partial()` produces `undefined` for any field the client did not send, `ProductService.update` (product.service.ts:53-60) must explicitly distinguish "field omitted" from "field explicitly set" for every property. It does this via a series of conditional spreads: `...(input.sku !== undefined ? { sku: input.sku } : {})`, repeated for each of the six mutable fields.

This construction is significant because it allows a genuine "no-op" partial update if the client sends `{}` (or any subset of fields) — the repository's `update` call will be issued with a data object containing only the keys the client actually intended to change, rather than silently overwriting fields with the value read from the current record (which is a common source of bugs in less careful PATCH implementations) or with `undefined`/schema-default values (which would be a correctness bug of a different kind: Prisma treats an explicit `undefined` value in an update payload as "no-op for that field" by default, so even a naive spread without the guard would happen to behave correctly for `undefined` — however, the explicit guards make the intent unambiguous and defensive against any future change to how the partial schema populates omitted keys, e.g. if the schema were changed to set explicit defaults).

One notable edge case is `description`: because it is a nullable field (`description ?? null` handling only exists on create, product.service.ts:38), the update path allows `description` to be explicitly set to any string the client provides but does not provide an explicit mechanism to clear/null out an existing description via update — since the guard is `input.description !== undefined`, sending `description: null` would need to pass Zod validation for the field type `z.string().max(2000).optional()`, which does not accept `null` as a valid value; therefore, a client cannot un-set a previously-set description once created, only replace it with another non-empty string within 2000 characters (or leave it untouched).

**Rule workflow:**
```
PATCH body received (already Zod-validated, all fields optional)
  build updateData = {}
  for each field in [sku, name, description, priceCents, stockQuantity, active]:
    if input[field] !== undefined:
      updateData[field] = input[field]
  repository.update(id, updateData)  -- Prisma applies only the supplied keys
```

---

### Business Rule: List Filtering, Search, and Pagination

**Overview:**
The list endpoint supports optional filtering by active status and free-text search across `name`/`sku`, combined with mandatory pagination bounded to a maximum page size of 100.

**Detailed description:**
Query parameters are validated and coerced by `listProductsQuerySchema` (product.schemas.ts:16-24). `page` and `pageSize` are coerced from string query params to numbers (`z.coerce.number()`), with `page` defaulting to 1 (minimum 1) and `pageSize` defaulting to 20 (minimum 1, maximum 100) — the upper bound on `pageSize` acts as a safeguard against clients requesting unbounded result sets that could degrade database performance. The `search` parameter, when present, must be a non-empty string after trimming whitespace (`z.string().trim().min(1)`); an empty or whitespace-only search string fails validation entirely (HTTP 400) rather than being silently ignored. The `active` filter is unusual in that it is typed as a Zod union of the literal strings `'true'`/`'false'` (not a JSON boolean, since HTTP query strings are always strings) and transformed into an actual boolean via `.transform()` — any other string value (e.g., `?active=yes`) fails validation.

`ProductService.list` (product.service.ts:14-22) translates the validated query into repository-level pagination parameters: `skip = (page - 1) * pageSize` and `take = pageSize`, a standard offset-based pagination formula. `ProductRepository.buildWhere` (product.repository.ts:13-23) constructs the Prisma `where` clause conditionally: the `active` filter is applied only if explicitly defined (`filters.active !== undefined`), and the `search` term, if present, is applied as an `OR` condition matching either `name` or `sku` via Prisma's `contains` operator (a case-sensitivity behavior dependent on the underlying database collation — not explicitly forced to case-insensitive in the code, so behavior may vary by database engine/collation configuration). When neither filter is supplied, the `where` clause is an empty object, matching all products.

The repository's `list` method (product.repository.ts:25-37) executes the paginated `findMany` and the total `count` as a single Prisma `$transaction`, ensuring the reported `total` (and consequently `totalPages`, computed in `shared/http/response.ts`) is consistent with the returned page even under concurrent writes — without the transaction, a product insert/delete occurring between the two separate queries could cause the total count to disagree with the actual page contents. Results are always ordered by `createdAt` descending, meaning newly created products always appear first on an unfiltered/unsorted list call; there is no client-configurable sort order exposed by this component.

**Rule workflow:**
```
GET /products?page=&pageSize=&search=&active=
  Zod validate + coerce query params (defaults: page=1, pageSize=20)
  service.list(query):
    skip = (page - 1) * pageSize
    take = pageSize
    repository.list({ search, active, skip, take }):
      where = {}
      if active !== undefined: where.active = active
      if search: where.OR = [{ name: contains search }, { sku: contains search }]
      run in $transaction:
        findMany(where, orderBy createdAt desc, skip, take)
        count(where)
      return { items, total }
    return paginated(items, page, pageSize, total)
      -> pagination.totalPages = ceil(total / pageSize)  [0 if pageSize is 0 -- not reachable given schema min(1)]
```

---

## 4. Component Structure

```
src/modules/products/
├── product.controller.ts   # HTTP handlers: list, getById, create, update, delete (thin adapter over ProductService)
├── product.service.ts      # Business logic: SKU uniqueness, existence checks, partial update merging, pagination assembly
├── product.repository.ts   # Data access: Prisma query building (buildWhere), list/findById/findBySku/findManyByIds/create/update/delete
├── product.routes.ts       # Express Router wiring: authenticate + validate middleware chains per route
└── product.schemas.ts      # Zod schemas and inferred types: createProductSchema, updateProductSchema, listProductsQuerySchema, productIdParamSchema
```

Supporting/boundary files outside the module directory that are integral to the controller's behavior:

```
src/
├── app.ts                                # buildControllers() wires ProductRepository -> ProductService -> ProductController; buildApp() mounts routes and global middleware/error handler
├── routes/index.ts                       # Mounts buildProductRouter under /api/v1/products
├── middlewares/
│   ├── auth.middleware.ts                # authenticate: JWT verification, populates req.user (id, email, role)
│   ├── validate.middleware.ts            # validate({ body, query, params }): Zod parsing, ValidationError on failure
│   └── error.middleware.ts               # errorMiddleware: converts AppError/ZodError/Prisma errors into JSON HTTP responses
├── shared/
│   ├── errors/
│   │   ├── app-error.ts                  # Base AppError class (statusCode, errorCode, details)
│   │   └── http-errors.ts                # NotFoundError, ConflictError, ValidationError, etc.
│   └── http/response.ts                  # paginated()/buildPagination() helpers used by ProductService.list
└── config/env.ts                         # JWT_SECRET consumed by auth.middleware.ts (indirect dependency)

prisma/schema.prisma                      # Product model definition (lines 56-72): fields, constraints, indexes, relation to OrderItem
```

---

## 5. Dependency Analysis

```
Internal Dependencies (compile-time imports):
ProductController → ProductService (product.controller.ts:2)
ProductController → ListProductsQuery type (product.controller.ts:3, from product.schemas.ts)
ProductService → ProductRepository (product.service.ts:2)
ProductService → ConflictError, NotFoundError (product.service.ts:3, from shared/errors)
ProductService → CreateProductInput, ListProductsQuery, UpdateProductInput types (product.service.ts:4-8)
ProductService → paginated(), PaginatedResponse type (product.service.ts:9, from shared/http/response)
ProductRepository → Prisma, PrismaClient, Product types (product.repository.ts:1, from @prisma/client)
product.routes.ts → authenticate (middlewares/auth.middleware.ts)
product.routes.ts → validate (middlewares/validate.middleware.ts)
product.routes.ts → createProductSchema, listProductsQuerySchema, productIdParamSchema, updateProductSchema (product.schemas.ts)
product.routes.ts → ProductController type (product.controller.ts)
src/app.ts → ProductRepository, ProductService, ProductController (constructs and wires the dependency chain with a shared PrismaClient instance)
src/routes/index.ts → ProductController type, buildProductRouter (mounts under /products)

Runtime/Indirect Dependencies:
OrderService (modules/orders/order.service.ts) reads/writes the same `Product` Prisma model directly (via its own transactions) — NOT via ProductRepository/ProductService/ProductController. This is a parallel data-access path to the same table, outside this component's boundary.

External Dependencies:
- express (4.21.1)      - HTTP server framework; RequestHandler type, Router
- @prisma/client (5.22.0) - ORM / database client; Product, Prisma, PrismaClient types; underlying MySQL/compatible DB (db.Char/db.VarChar/db.Text hints in schema)
- zod (3.23.8)           - Runtime schema validation and type inference for all product input/query/param shapes
- jsonwebtoken (9.0.2)   - JWT verification in the authenticate middleware (indirect dependency of the route's auth gate)
- pino / pino-http (indirect) - Structured logging used by error middleware and request logger (not directly imported by ProductController)
```

---

## 6. Afferent and Efferent Coupling

Coupling is assessed at the class/module level, consistent with this codebase's object-oriented TypeScript style (Express + class-based controllers/services/repositories).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|-------------------|
| ProductController | 2 (product.routes.ts, src/app.ts construction; src/routes/index.ts as type reference) | 2 (ProductService, product.schemas.ts types) | Low |
| ProductService | 1 (ProductController) | 4 (ProductRepository, shared/errors, product.schemas.ts types, shared/http/response) | Medium |
| ProductRepository | 1 (ProductService) | 1 (@prisma/client) | Low |
| product.schemas.ts | 3 (ProductController, ProductService, product.routes.ts) | 1 (zod) | Medium |
| product.routes.ts | 1 (src/routes/index.ts) | 4 (auth.middleware, validate.middleware, product.schemas.ts, ProductController) | Low |

Notes:
- `product.schemas.ts` has the highest afferent coupling within the module — it is the shared contract consumed by controller (types), service (types), and routes (both types and schema objects for validation), making it the most sensitive file to breaking changes.
- `ProductService` has the highest efferent coupling, consistent with its role as the business-logic orchestrator that touches error types, repository, schema types, and the shared pagination helper.
- No circular dependencies were observed within the module boundary.
- `ProductController` itself has low coupling in both directions, reflecting its intentionally thin "adapter" design — its criticality as a single point of failure is low because it contains no unique logic; risk is concentrated downstream in `ProductService`.

---

## 7. Endpoints

All endpoints are mounted under the base path `/api/v1/products` (src/routes/index.ts:27) and require authentication (`authenticate` middleware applied globally to the router at product.routes.ts:14). No route-specific role restriction (`requireRole`) is applied to any product endpoint — any authenticated user (ADMIN or OPERATOR) can perform all operations, including destructive ones (create/update/delete).

| Endpoint | Method | Description | Auth Required | Request Validation | Success Status | Handler |
|----------|--------|-------------|----------------|--------------------|-----------------|---------|
| /api/v1/products | GET | List products with pagination, optional `search` and `active` filters | Yes (any role) | Query: `listProductsQuerySchema` | 200 | `ProductController.list` (product.controller.ts:8-16) |
| /api/v1/products/:id | GET | Retrieve a single product by UUID | Yes (any role) | Params: `productIdParamSchema` | 200 | `ProductController.getById` (product.controller.ts:18-25) |
| /api/v1/products | POST | Create a new product | Yes (any role) | Body: `createProductSchema` | 201 | `ProductController.create` (product.controller.ts:27-34) |
| /api/v1/products/:id | PATCH | Partially update an existing product | Yes (any role) | Params: `productIdParamSchema`, Body: `updateProductSchema` | 200 | `ProductController.update` (product.controller.ts:36-43) |
| /api/v1/products/:id | DELETE | Delete a product (hard delete) | Yes (any role) | Params: `productIdParamSchema` | 204 | `ProductController.delete` (product.controller.ts:45-52) |

Error response shapes (produced by `errorMiddleware`, not by the controller itself):
- 400: `VALIDATION_ERROR` (Zod validation failures on body/query/params)
- 401: `UNAUTHORIZED` (missing/invalid/expired JWT)
- 404: `NOT_FOUND` (`Product not found`, or generic Prisma P2025 fallback)
- 409: `SKU_ALREADY_USED` (application-level) or `CONFLICT` (Prisma P2002 unique constraint fallback)
- 500: `INTERNAL_SERVER_ERROR` (unhandled/unexpected errors)

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|-----------------|
| MySQL-compatible database (via Prisma) | Internal Datastore | Product persistence (products table) | Prisma Client (SQL under the hood) | Relational rows mapped to Product TS type | Prisma errors (P2002, P2025) caught centrally in `errorMiddleware`; no component-level retry/circuit breaker |
| JWT Authentication | Cross-cutting internal service | Authenticates all product route requests | In-process middleware, HS-signed JWT verified via `jsonwebtoken` | Bearer token in `Authorization` header | Verification failure -> `UnauthorizedError` (401), handled uniformly |
| OrderService (modules/orders) | Internal module (indirect) | Reads Product rows (price, stock, active) when creating/paying/cancelling orders; mutates `stockQuantity` outside this component | Direct Prisma calls (bypasses ProductRepository/ProductService) | Prisma `Product` model | Handled within OrderService's own transactions; not visible to or managed by ProductController |
| Express Router / global error middleware | Internal framework integration | Standard request/response lifecycle and centralized error translation | HTTP (Express `RequestHandler`/`ErrorRequestHandler`) | JSON | `next(err)` pattern in every controller method delegates all error formatting to `errorMiddleware` |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered Architecture (Controller-Service-Repository) | ProductController -> ProductService -> ProductRepository | product.controller.ts, product.service.ts, product.repository.ts | Separation of HTTP concerns, business logic, and data access |
| Dependency Injection (manual constructor injection) | `constructor(private readonly products: ProductService)` and equivalent in Service/Repository | product.controller.ts:6, product.service.ts:12, product.repository.ts:11 | Testability and decoupling; dependencies wired centrally in `buildControllers()` (app.ts:26-53) |
| Repository Pattern | `ProductRepository` encapsulates all Prisma query construction | product.repository.ts | Abstracts persistence details (Prisma-specific `where`/`$transaction` usage) away from business logic |
| Factory Function | `buildProductRouter(controller)`, `buildControllers(prisma)`, `buildApp(deps)` | product.routes.ts:12, app.ts:26, app.ts:55 | Composition root pattern; explicit, testable construction of the dependency graph without a DI framework/container |
| Chain of Responsibility (Express middleware pipeline) | `authenticate` -> `validate(...)` -> controller handler -> `errorMiddleware` | product.routes.ts:14-24, error.middleware.ts | Cross-cutting concerns (auth, validation, error formatting) applied uniformly and composably per route |
| Schema-first Validation / Parse-don't-validate | Zod schemas define both runtime validation and static TypeScript types via `z.infer` | product.schemas.ts:26-28 | Single source of truth for input shape; eliminates manual type/validation duplication |
| Guard Clause | `if (!product) throw new NotFoundError(...)`, existence checks before mutation | product.service.ts:26, 46, 64 | Fail-fast error handling, keeps main logic path unindented/linear |
| DTO/Value Object via inferred types | `CreateProductInput`, `UpdateProductInput`, `ListProductsQuery` | product.schemas.ts:26-28 | Strongly-typed request payload contracts flowing from route to service |
| Centralized Error Hierarchy | `AppError` base class with subclasses per HTTP semantic (`NotFoundError`, `ConflictError`, etc.) | shared/errors/app-error.ts, shared/errors/http-errors.ts | Consistent error-to-HTTP-status mapping handled once in `errorMiddleware` rather than per controller |
| Uniform Controller Handler Shape | Every method: `try { ...; res.status(x).json(y) } catch (err) { next(err) }` | product.controller.ts:8-52 (all 5 handlers) | Predictable, low-complexity controller code; async errors correctly forwarded to Express error pipeline (Express 4 does not auto-catch async rejections) |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|-----------------|-------|--------|
| High | product.routes.ts:14-24 | No `requireRole` authorization guard on any product route — any authenticated OPERATOR can create, update, or delete products, an operation arguably reserved for ADMIN in many order-management domains (the `requireRole` helper exists in auth.middleware.ts:48-58 and is presumably used elsewhere, e.g. Users module, but not here) | Privilege escalation / unintended catalog mutation risk; inconsistent authorization posture across modules |
| High | Test suite (tests/) | No dedicated `products.test.ts` — ProductController's five endpoints (list/getById/create/update/delete) have zero direct integration test coverage; Product is only exercised as a dependency of Order tests via `createTestProduct` factory | Regressions in product CRUD behavior, validation, or SKU-conflict handling would not be caught by CI |
| Medium | product.repository.ts:59-61 | `delete()` performs a hard delete with no check for existing `OrderItem` references; relies entirely on the database FK constraint (if any) to prevent orphaned/broken order history, and any resulting Prisma FK error is not translated into a domain-specific error by `errorMiddleware` (only P2002/P2025 are special-cased) | Deleting a product referenced by historical orders could either silently corrupt order history (if no FK constraint) or surface an unfriendly generic 500 error to the client (if FK constraint exists and is untranslated) |
| Medium | product.service.ts:31-34, 47-52 | SKU uniqueness check-then-write is not transactional (separate `findBySku` and `create`/`update` calls) — a TOCTOU race is possible under concurrent requests, mitigated only by the DB-level unique constraint producing a different, less user-friendly error shape (generic `CONFLICT` vs. `SKU_ALREADY_USED`) | Under concurrent duplicate-SKU submissions, one caller receives the friendly domain error while the loser of the race may receive the generic Prisma-derived 409 instead, producing inconsistent client-facing error codes for what is logically the same failure |
| Medium | product.repository.ts:16-20 | Search uses Prisma `contains` without an explicit case-insensitivity mode; behavior is dependent on the underlying database's default collation, not guaranteed consistent across environments/DB engines | Search results may unexpectedly be case-sensitive or case-insensitive depending on deployment/DB configuration, with no test coverage to catch a regression |
| Low | product.service.ts:53-60 | Update's `description` field cannot be explicitly cleared to null/empty once set (schema only allows `optional()`, not `nullable()`), which may or may not be intended business behavior — undocumented in code | Product owners cannot remove a previously entered description without providing a replacement string; ambiguous whether this is a deliberate constraint or an oversight |
| Low | product.controller.ts (whole file) | No request-level authorization of resource ownership/tenant scoping is visible (single-tenant assumption); not necessarily a defect but worth flagging as an assumption if multi-tenancy is ever introduced | Would require rework if the system moves to multi-tenant product catalogs |

---

## 11. Test Coverage Analysis

The project uses `vitest` (2.1.4) with `supertest` (7.0.0) for HTTP-level integration testing. Test files live under `/tests` at the project root (not co-located with source modules).

| Test File | Scope | Direct ProductController Coverage | Notes |
|-----------|-------|-----------------------------------|-------|
| `tests/products.test.ts` | N/A — does not exist | None | No dedicated test file for the Products module was found anywhere in the repository (searched `tests/`, and via `.spec.ts`/`.test.ts` glob across the whole project excluding `node_modules`) |
| `tests/orders.test.ts` | Order creation/payment/cancellation flows | Indirect only | Uses `createTestProduct` (tests/helpers/factories.ts:62-74) to seed products consumed by Order endpoints; exercises `Product.stockQuantity` mutation and `Product.active`/stock-insufficiency behavior, but only as invoked through `OrderService`'s direct Prisma calls — never through `ProductController`/`ProductService`/`ProductRepository` |
| `tests/auth.test.ts` | Authentication flows | None | Unrelated to products, but establishes the `authenticate` middleware behavior relied upon by product routes |
| `tests/helpers/factories.ts` | Test data/app bootstrap helpers | Supporting only | Provides `createTestProduct()` (a direct Prisma factory, bypassing the controller/service/repository entirely) and `getTestApp()`/`bootstrapAuthenticatedUser()` used by other test files |
| `tests/setup.ts` | Test environment/database setup | N/A | Global test lifecycle hooks (not product-specific) |

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|--------------------|----------|---------------|
| ProductController (list/getById/create/update/delete handlers) | 0 | 0 | None (0%) | No assertions exist for any of the five HTTP endpoints exposed by this controller |
| ProductService (SKU uniqueness, existence guards, partial update, pagination) | 0 | 0 | None (0%) | Business rules documented in Section 3 are entirely unverified by automated tests |
| ProductRepository (query building, search filter, pagination) | 0 | 0 (indirect via `createTestProduct`, which does not exercise `ProductRepository`) | None (0%) | `buildWhere` search/active filtering logic is untested |
| product.schemas.ts (Zod validation rules) | 0 | 0 | None (0%) | No test verifies rejection of invalid SKU length, negative price/stock, malformed UUID, out-of-range pageSize, etc. |

**Summary assessment:** Test coverage for the `ProductController` component and its full boundary (`ProductService`, `ProductRepository`, `product.schemas.ts`) is effectively zero at the direct/unit level. The only exercise of the underlying `Product` Prisma model in the automated test suite happens through `orders.test.ts`, which creates products via a raw Prisma factory (`createTestProduct`) that bypasses this component's controller, service, and repository layers entirely — meaning none of the documented business rules (SKU uniqueness on create/update, existence checks, partial update field-merging, search/pagination behavior, or input validation boundaries) are verified by CI. This is flagged as a High risk in Section 10.

---
