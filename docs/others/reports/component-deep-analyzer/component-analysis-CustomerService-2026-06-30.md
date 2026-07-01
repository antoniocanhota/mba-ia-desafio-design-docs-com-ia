# Component Deep Analysis Report: CustomerService

**Project:** order-management-api (Order Management System REST API)
**Component analyzed:** `CustomerService`
**Primary file:** `src/modules/customers/customer.service.ts`
**Analysis date:** 2026-06-30

---

## 1. Executive Summary

`CustomerService` is the application/business-logic layer for the Customers bounded context within the Order Management System REST API. It sits between `CustomerController` (HTTP layer) and `CustomerRepository` (data-access layer), and is responsible for enforcing the domain invariants of the `Customer` entity — most notably **email uniqueness** — and for translating persistence-layer results (or their absence) into domain-level outcomes (`NotFoundError`, `ConflictError`).

The component is small (64 lines), has a single public responsibility set (CRUD operations for customers) and follows a layered/service-oriented architecture with constructor-based dependency injection. It has one direct dependency: `CustomerRepository`, injected via the constructor. It does not talk to the database, HTTP framework, or validation library directly — those concerns are handled by `CustomerRepository`, `CustomerController`/Express, and `customer.schemas.ts` (Zod) respectively.

Key findings:

- The component correctly delegates persistence to `CustomerRepository` and keeps the class free of ORM/Prisma-specific code, indicating good separation of concerns.
- Only one substantive business rule is enforced in this layer: email must be unique across customers, both on create and on update (rule verified again via a fresh repository lookup rather than relying on a query result at the boundary of validation).
- `update` performs a redundant double round-trip to the repository (`getById` then implicit `findByEmail`, then `update`) — acceptable for correctness but has minor performance implications under load (see Technical Debt).
- There are **no automated tests that directly exercise `CustomerService`** (unit or integration). The only test artifact referencing customers is a factory helper (`tests/helpers/factories.ts`) used indirectly by `tests/orders.test.ts` to seed customer records as a precondition for order tests — it does not exercise any `CustomerService` behavior (create validation, uniqueness conflict, not-found handling, update semantics, delete).
- Error handling is declarative and consistent: domain errors (`NotFoundError`, `ConflictError`) are thrown directly from the service and are expected to be caught by a global error-handling middleware (`error.middleware.ts`) rather than being handled locally.
- The service is stateless and free of side effects beyond repository calls, making it straightforward to reason about and to unit test (once tests are introduced).

---

## 2. Data Flow Analysis

### 2.1 `list(query)` flow

```
1. Request enters via CustomerController.list (customer.controller.ts:8-16)
2. Query string parsed/validated by validate() middleware using listCustomersQuerySchema (customer.routes.ts:16, customer.schemas.ts:25-29)
3. CustomerService.list(query) invoked (customer.service.ts:14-21)
4. Pagination offset computed: skip = (page - 1) * pageSize
5. CustomerRepository.list({ search, skip, take }) executed (customer.repository.ts:24-36)
   - Builds a Prisma "OR" filter across name/email/document if search is provided
   - Runs findMany + count inside a Prisma $transaction for consistency
6. Result { items, total } returned to CustomerService
7. paginated(items, page, pageSize, total) builds a PaginatedResponse (shared/http/response.ts:22-24)
8. CustomerController serializes response as JSON with HTTP 200
```

### 2.2 `getById(id)` flow

```
1. Request enters via CustomerController.getById (customer.controller.ts:18-25)
2. :id param validated as UUID by validate() middleware using customerIdParamSchema (customer.schemas.ts:3-5)
3. CustomerService.getById(id) invoked (customer.service.ts:23-27)
4. CustomerRepository.findById(id) queries Prisma customer.findUnique
5. If null -> throw NotFoundError('Customer') (propagates to error middleware, mapped to HTTP 404)
6. If found -> Customer entity returned
7. CustomerController responds 200 with JSON body
```

### 2.3 `create(input)` flow

```
1. Request enters via CustomerController.create (customer.controller.ts:27-34)
2. Request body validated against createCustomerSchema by validate() middleware (customer.schemas.ts:15-21)
   (name, email, phone, document, address structural/format validation)
3. CustomerService.create(input) invoked (customer.service.ts:29-41)
4. CustomerRepository.findByEmail(input.email) checks for an existing customer with the same email
5. If found -> throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED') (mapped to HTTP 409)
6. If not found -> CustomerRepository.create({name, email, phone, document, address}) persists via Prisma
7. Created Customer entity returned to controller
8. CustomerController responds 201 with JSON body
```

### 2.4 `update(id, input)` flow

```
1. Request enters via CustomerController.update (customer.controller.ts:36-43)
2. :id validated as UUID, body validated against updateCustomerSchema (partial of createCustomerSchema) (customer.routes.ts:19-23)
3. CustomerService.update(id, input) invoked (customer.service.ts:43-58)
4. this.getById(id) executed as an existence guard -> throws NotFoundError if absent (HTTP 404)
5. If input.email is present:
   a. CustomerRepository.findByEmail(input.email) is queried
   b. If a record is found AND its id differs from the target id -> throw ConflictError('EMAIL_ALREADY_USED') (HTTP 409)
6. A partial update payload is built, including only defined fields (name/email/phone/document/address)
7. CustomerRepository.update(id, partialData) persists changes via Prisma
8. Updated Customer entity returned to controller
9. CustomerController responds 200 with JSON body
```

### 2.5 `delete(id)` flow

```
1. Request enters via CustomerController.delete (customer.controller.ts:45-52)
2. :id validated as UUID by validate() middleware
3. CustomerService.delete(id) invoked (customer.service.ts:60-63)
4. this.getById(id) executed as an existence guard -> throws NotFoundError if absent (HTTP 404)
5. CustomerRepository.delete(id) executes Prisma customer.delete
6. CustomerController responds 204 No Content
```

### 2.6 Error propagation flow (cross-cutting)

```
1. CustomerService throws a typed AppError subclass (NotFoundError / ConflictError)
2. CustomerController catch block forwards the error via next(err) (all 5 handlers)
3. Express error-handling middleware chain routes to error.middleware.ts
4. error.middleware.ts (not part of this component) is expected to map AppError.statusCode/errorCode to the HTTP response
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|-------------------|----------|
| Uniqueness Constraint | Customer email must be unique across all customers at creation time | customer.service.ts:30-33 |
| Uniqueness Constraint | Customer email must remain unique across all customers when updated, excluding the record being updated itself | customer.service.ts:45-50 |
| Existence Guard | A customer must exist to be retrieved by ID, otherwise a 404-mapped error is raised | customer.service.ts:24-26 |
| Existence Guard | A customer must exist before it can be updated (pre-check before mutation) | customer.service.ts:44 |
| Existence Guard | A customer must exist before it can be deleted (pre-check before mutation) | customer.service.ts:61 |
| Partial Update Semantics | Update only applies fields explicitly provided in the input (undefined fields are not overwritten) | customer.service.ts:51-57 |
| Pagination Policy | List results are paginated using page/pageSize with a computed skip offset | customer.service.ts:15-20 |
| Search Policy (delegated) | Search term matches against name, email, or document fields (implemented in repository, invoked from service) | customer.repository.ts:12-22, customer.service.ts:15-19 |
| Data Boundary | Create only accepts and forwards a fixed whitelist of fields (name, email, phone, document, address) to the repository | customer.service.ts:34-40 |
| Input Validation (delegated, upstream) | Structural validation (string lengths, email format, UUID format, address shape) enforced at HTTP boundary via Zod schemas before reaching the service | customer.schemas.ts:1-33 |

### Detailed breakdown of the business rules

---

### Business Rule: Unique Email on Customer Creation

**Overview**:
A new customer cannot be created if another customer already exists with the same email address. This is enforced at the service layer via an explicit pre-check, independent of any database-level uniqueness constraint.

**Detailed description**:
When `CustomerService.create(input)` is invoked, the method first calls `this.customers.findByEmail(input.email)` to look up any existing customer sharing the same email. If a record is found, the service immediately throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (customer.service.ts:30-33), short-circuiting before any write occurs. This guarantees that no customer record with a duplicate email is created through this code path, and that the caller receives a semantically meaningful HTTP 409 response with a stable machine-readable error code (`EMAIL_ALREADY_USED`) rather than a generic database constraint violation.

This rule is a defense-in-depth complement to the database-level constraint declared in the Prisma schema (`email String @unique @db.VarChar(255)`, prisma/schema.prisma:43). The database constraint is the ultimate source of truth and prevents duplicates even under race conditions (e.g., two concurrent create requests for the same email), but it would surface as a raw Prisma unique-constraint violation (`P2002`) if relied upon alone. Because `CustomerService` does not appear to catch or translate Prisma-specific error codes, a race condition between the `findByEmail` check and the `create` call could still result in an unhandled Prisma exception bubbling up through the error middleware rather than a clean `ConflictError`. This is a check-then-act (TOCTOU) pattern, common in service-layer uniqueness checks, and is only as strong as the surrounding error-handling middleware's ability to gracefully degrade unexpected exceptions.

From a business perspective, this rule ensures each customer email functions as a de-facto natural key, which is important for customer support workflows (looking a customer up by email), for order attribution, and for preventing accidental duplicate customer profiles from bad client input or retried requests.

**Rule workflow**:
```
create(input)
  -> findByEmail(input.email)
     -> found?  -> throw ConflictError("Email already registered", "EMAIL_ALREADY_USED")  [HTTP 409]
     -> not found -> proceed to repository.create(whitelisted fields)
```

---

### Business Rule: Unique Email on Customer Update (Self-Exclusion)

**Overview**:
When updating a customer's email, the new email must not belong to any *other* customer. The customer being updated is explicitly excluded from the conflict check so that a no-op email update (or update of other fields while keeping the same email) does not falsely trigger a conflict.

**Detailed description**:
`CustomerService.update(id, input)` only performs the uniqueness check when `input.email` is present (customer.service.ts:45-50), since `updateCustomerSchema` is a partial schema and email is optional on update. When an email is supplied, the service queries `findByEmail(input.email)` and evaluates two conditions together: a record was found (`sameEmail`) AND that record's `id` is different from the `id` being updated (`sameEmail.id !== id`). Only when both conditions hold does the method throw `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')`. This self-exclusion logic is essential: without it, a client re-submitting a customer's own unchanged email as part of a broader update (e.g., changing only the phone number but sending the full object) would always be rejected with a false-positive conflict, since `findByEmail` would find the very record being updated.

This rule runs *after* the existence guard (`await this.getById(id)` at line 44), meaning a 404 for a non-existent customer takes precedence over a 409 for email conflicts — the method fails fast on the more fundamental problem (resource does not exist) before evaluating business constraints on the update payload. The check also does not consider the newly resolved customer object from `getById` (a wasted opportunity: `getById(id)` already returns the current customer, but its result is discarded, and the method re-derives conflict information from a second query rather than comparing directly against the fetched record's own email). This is a candidate for a mostly-informational observation rather than a rule per se — see Technical Debt.

Functionally, this preserves data integrity: the email uniqueness invariant established at creation time is upheld throughout the customer lifecycle, not just at the moment of creation.

**Rule workflow**:
```
update(id, input)
  -> getById(id)                      -- existence guard; throws NotFoundError if missing [HTTP 404]
  -> if input.email is defined:
       -> findByEmail(input.email)
          -> found AND found.id !== id -> throw ConflictError("Email already registered", "EMAIL_ALREADY_USED")  [HTTP 409]
  -> repository.update(id, partial fields present in input)
```

---

### Business Rule: Existence Guard for Read, Update, and Delete

**Overview**:
Any operation targeting a specific customer by ID (`getById`, `update`, `delete`) must first confirm the customer exists; otherwise the operation fails with a `NotFoundError`.

**Detailed description**:
`getById(id)` is the canonical existence check and is reused (composed) by both `update` and `delete` (customer.service.ts:44, 61), which is a clear application of the DRY principle within the service. If `CustomerRepository.findById(id)` returns `null`, `CustomerService.getById` throws `NotFoundError('Customer')` (customer.service.ts:25), which the shared `NotFoundError` class maps to HTTP 404 with the message `"Customer not found"` and error code `NOT_FOUND` (shared/errors/http-errors.ts:27-31).

By reusing `getById` inside `update` and `delete`, the service guarantees consistent 404 semantics across all ID-targeted operations without duplicating the null-check logic. However, this also means `update` and `delete` incur an additional round-trip to the database purely for existence validation, beyond what is strictly required by the underlying Prisma operations (Prisma's `update`/`delete` would themselves throw if the record does not exist, via a `P2025` "Record not found" error). The service's choice to pre-check explicitly, rather than catch and translate Prisma's own not-found error, trades a bit of performance for clearer, centralized, and framework-agnostic error semantics — a defensible architectural decision given that it decouples business logic from ORM-specific error codes.

This rule directly shapes the API contract: consumers can rely on a 404 with a consistent shape for any operation against a nonexistent customer ID, rather than encountering inconsistent errors (e.g., a raw database error) depending on which endpoint was called.

**Rule workflow**:
```
getById(id) / update(id, ...) / delete(id)
  -> repository.findById(id)
     -> null  -> throw NotFoundError("Customer")  [HTTP 404, code NOT_FOUND]
     -> found -> proceed with operation
```

---

### Business Rule: Partial Update Field Whitelisting

**Overview**:
Updates only modify fields that are explicitly present (not `undefined`) in the input payload; omitted fields are left unchanged in the persisted record.

**Detailed description**:
`CustomerService.update` constructs its repository payload using a series of conditional spreads: `...(input.name !== undefined ? { name: input.name } : {})` and similarly for `email`, `phone`, `document`, and `address` (customer.service.ts:51-57). Because `UpdateCustomerInput` is derived from `createCustomerSchema.partial()` (customer.schemas.ts:23), every field is optional at the type level, and this construction pattern ensures only supplied fields are included in the object passed to `CustomerRepository.update`. This produces a proper "PATCH" semantic (as reflected in the HTTP method used in `customer.routes.ts:19`, `router.patch('/:id', ...)`) rather than a "PUT" full-replace semantic.

This approach also protects against accidentally nulling out fields: if the conditional spread were omitted and the raw `input` object were passed directly to Prisma's `update`, any field not included by the client (and thus `undefined` in JS) would generally be a no-op for Prisma's update semantics as well, but explicit whitelisting here makes the intent unambiguous and resilient to future changes in how the input object is constructed (e.g., if extra untrusted properties were ever present on `input`, they would not leak through to the repository call, since the four named fields are exhaustively enumerated).

One nuance worth flagging: fields whose value is explicitly `null` (as opposed to `undefined`) would still pass the `!== undefined` check and be forwarded — though the schema types (all `z.string()`-based, non-nullable) make this scenario unlikely to occur through the validated HTTP path, since Zod validation would reject a `null` value for these string fields before it ever reaches the service.

**Rule workflow**:
```
update(id, input)
  -> for each field in [name, email, phone, document, address]:
       if input.field !== undefined -> include field in update payload
  -> repository.update(id, assembledPartialPayload)
```

---

### Business Rule: Paginated Listing with Optional Search

**Overview**:
Customer listing is always paginated (page/pageSize-based) and optionally filtered by a free-text search term matched against name, email, or document.

**Detailed description**:
`CustomerService.list(query)` translates a 1-based `page` and a `pageSize` into a zero-based `skip` offset (`(query.page - 1) * query.pageSize`, customer.service.ts:17) and passes `search`, `skip`, and `take` (aliased from `pageSize`) to `CustomerRepository.list`. The repository builds a Prisma `OR` filter across `name`, `email`, and `document` using `contains` (substring match) when a `search` term is present (customer.repository.ts:12-22), and returns both the page of items and the total count computed within a single Prisma `$transaction` to ensure count and items are consistent with each other at the same point in time.

The service layer itself contains no pagination bounds-checking logic (e.g., capping `pageSize`) — that responsibility is delegated entirely to `listCustomersQuerySchema` at the HTTP boundary, which enforces `page >= 1`, `1 <= pageSize <= 100`, with defaults of `page=1` and `pageSize=20` (customer.schemas.ts:25-29). This means `CustomerService.list` implicitly trusts its caller (the controller, and transitively the validation middleware) to have already produced sane values; if `CustomerService` were invoked directly (e.g., from a different entry point, a script, or a future GraphQL resolver) without going through the Zod-validated HTTP path, no equivalent guard exists within the service itself.

The resulting `PaginatedResponse` shape (`{ data, pagination: { page, pageSize, total, totalPages } }`) is a shared, reusable envelope (`shared/http/response.ts`) used consistently across the API, indicating a project-wide convention for list endpoints rather than a customer-specific pattern.

**Rule workflow**:
```
list(query: { page, pageSize, search? })
  -> skip = (page - 1) * pageSize
  -> repository.list({ search, skip, take: pageSize })
     -> where = search ? OR[name/email/document contains search] : {}
     -> [items, total] = $transaction([findMany(where, skip, take), count(where)])
  -> paginated(items, page, pageSize, total)
```

---

## 4. Component Structure

```
src/modules/customers/
├── customer.controller.ts   # HTTP request/response adapter; delegates to CustomerService, forwards errors via next()
├── customer.repository.ts   # Prisma-backed data access; builds search filters, executes CRUD against `customer` table
├── customer.routes.ts       # Express Router wiring: auth middleware + Zod validation middleware + controller handlers
├── customer.schemas.ts      # Zod schemas: createCustomerSchema, updateCustomerSchema (partial), listCustomersQuerySchema, customerIdParamSchema
└── customer.service.ts      # ANALYZED COMPONENT — business logic: uniqueness rules, existence guards, pagination/search delegation
```

Related shared/cross-cutting files referenced by this component:

```
src/shared/
├── errors/
│   ├── app-error.ts          # AppError base class (statusCode, errorCode, details)
│   ├── http-errors.ts        # NotFoundError, ConflictError, and other typed AppError subclasses
│   └── index.ts              # Barrel export consumed by customer.service.ts
└── http/
    └── response.ts           # paginated()/buildPagination() shared pagination envelope builder

src/middlewares/
├── auth.middleware.ts        # JWT-based authenticate middleware applied to all customer routes
└── validate.middleware.ts    # Zod-based request validation middleware (params/query/body)

prisma/schema.prisma           # Customer model definition (id, name, email [unique], phone, document, address [Json], timestamps)
```

---

## 5. Dependency Analysis

```
Internal Dependencies (compile-time imports of CustomerService):
customer.service.ts
  -> @prisma/client (type-only: Customer)
  -> ./customer.repository.js (type-only: CustomerRepository)
  -> ../../shared/errors/index.js (ConflictError, NotFoundError)
  -> ./customer.schemas.js (type-only: CreateCustomerInput, ListCustomersQuery, UpdateCustomerInput)
  -> ../../shared/http/response.js (paginated, PaginatedResponse)

Runtime call chain:
CustomerController --> CustomerService --> CustomerRepository --> PrismaClient --> Database (Customer table)

Consumers of CustomerService (afferent, who imports it):
src/app.ts                              # composition root; instantiates CustomerService(customerRepository)
src/modules/customers/customer.controller.ts   # holds CustomerService as a constructor dependency

External Dependencies:
- @prisma/client            - Type import only (Customer entity type); no direct runtime Prisma calls in this file
- (transitively) Prisma ORM / underlying SQL database - via CustomerRepository, not directly imported here
- (transitively) zod        - via customer.schemas.ts types consumed as input parameter types (no runtime validation performed in this file)

Notably absent from this component:
- No direct Express/HTTP dependency (kept in Controller/Routes)
- No direct Prisma/database client dependency (kept in Repository)
- No logging, metrics, or tracing calls within the service
- No external service/API calls
```

---

## 6. Afferent and Efferent Coupling

Coupling is analyzed at the class level (object-oriented TypeScript), scoped to the Customers module and its immediate collaborators.

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-----------|------------------------|--------------------------|----------|
| CustomerService | 2 (CustomerController, app.ts composition root) | 3 (CustomerRepository, ConflictError, NotFoundError) | Medium |
| CustomerController | 1 (customer.routes.ts / app.ts) | 1 (CustomerService) | Low |
| CustomerRepository | 1 (CustomerService) | 1 (PrismaClient) | Medium |
| NotFoundError / ConflictError (shared/errors) | Many (used across multiple modules, e.g., orders, auth — outside this component's scope) | 1 (AppError) | Low (from this component's perspective) |
| paginated()/buildPagination() (shared/http/response) | Many (used across list endpoints project-wide) | 0 | Low |

Notes:
- `CustomerService` has low-to-medium efferent coupling (3 direct collaborators: one repository, two error types), consistent with a thin, focused service class.
- `CustomerService` has low afferent coupling within its own module (only the controller and the composition root reference it directly), which is expected for a leaf business-logic class not shared across modules. It is not reused by, e.g., the Orders module, even though Orders has a foreign-key relationship to Customer — Orders' own repository/service presumably queries the Customer table independently rather than depending on `CustomerService` (not confirmed within this component's scope; flagged as an assumption).
- Cohesion is high: all five public methods (`list`, `getById`, `create`, `update`, `delete`) operate exclusively on the `Customer` entity and its single collaborator (`CustomerRepository`), with no unrelated responsibilities mixed in.

---

## 7. Endpoints

`CustomerService` itself exposes no endpoints directly (it is not an HTTP entry point), but its public methods map one-to-one to the REST endpoints defined in `customer.routes.ts`, which is the entry point that invokes `CustomerService` via `CustomerController`. Documenting these endpoints is useful context for understanding the component's external contract.

| Endpoint | Method | Description | Auth | Validated Input | CustomerService Method Invoked |
|----------|--------|--------------|------|------------------|----------------------------------|
| /customers | GET | List customers with pagination and optional search | Bearer JWT (authenticate) | query: page, pageSize, search | list(query) |
| /customers/:id | GET | Get a single customer by ID | Bearer JWT (authenticate) | params: id (UUID) | getById(id) |
| /customers | POST | Create a new customer | Bearer JWT (authenticate) | body: name, email, phone, document, address | create(input) |
| /customers/:id | PATCH | Partially update a customer | Bearer JWT (authenticate) | params: id (UUID); body: partial of create schema | update(id, input) |
| /customers/:id | DELETE | Delete a customer | Bearer JWT (authenticate) | params: id (UUID) | delete(id) |

Note: The exact mount path (e.g., `/customers` vs `/api/customers`) depends on how `buildCustomerRouter` is mounted in `src/app.ts`, which is outside the scope of this single-component analysis; paths above are relative to that router's mount point. All routes require authentication via `authenticate` middleware (customer.routes.ts:14); no route-level role restriction (`requireRole`) is applied within this router, meaning any authenticated user (ADMIN or OPERATOR) can perform all five operations, including delete.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|-----------------|
| CustomerRepository | Internal collaborator | Persistence access for Customer entity (CRUD, search, uniqueness lookups) | In-process method calls (async/await) | Prisma `Customer` model / plain TS objects | Errors from repository (e.g., Prisma exceptions) are not caught locally in CustomerService; they propagate up uncaught unless the underlying Prisma call itself rejects, in which case the promise rejection bubbles to the controller's try/catch and then to next(err) |
| shared/errors (AppError hierarchy) | Internal collaborator | Standardized domain error signaling (NotFoundError, ConflictError) | In-process (thrown exceptions) | AppError instances with statusCode/errorCode/details | Caught downstream by Express error middleware (error.middleware.ts), not within this component |
| shared/http/response (paginated) | Internal collaborator | Consistent pagination envelope construction | In-process function call | Plain TS object { data, pagination } | N/A (pure function, no failure modes) |
| PostgreSQL/MySQL (via Prisma, indirect) | External data store | Underlying persistence for Customer records | SQL via Prisma Client (accessed only through CustomerRepository) | Relational rows mapped to Customer/Json (address stored as Json column) | Not handled within CustomerService; any DB-layer error (e.g., unique constraint P2002, connection errors) propagates unmodified through the repository call |

CustomerService has no direct external API integrations, no messaging/event-bus interactions, and no caching layer usage.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|-----------------|----------|---------|
| Layered Architecture (Controller-Service-Repository) | CustomerController -> CustomerService -> CustomerRepository | customer.controller.ts, customer.service.ts, customer.repository.ts | Separation of HTTP concerns, business logic, and data access |
| Dependency Injection (constructor injection) | `constructor(private readonly customers: CustomerRepository)` | customer.service.ts:12 | Decouples CustomerService from concrete repository instantiation; enables substitution (e.g., for testing) |
| Composition Root | Manual wiring of Repository -> Service -> Controller | src/app.ts:34-36 | Centralizes object graph construction outside the component itself |
| Guard Clause / Fail-Fast | Existence and uniqueness checks throw immediately before proceeding | customer.service.ts:25, 32, 48 | Keeps method bodies flat, avoids deep nesting, produces early, explicit error signaling |
| Typed Domain Errors | NotFoundError, ConflictError extending AppError | shared/errors/http-errors.ts, customer.service.ts:25,32,48 | Encodes HTTP-mapping metadata (statusCode, errorCode) into the exception itself, decoupling business logic from HTTP-layer knowledge of status codes |
| DTO / Schema-Derived Types | CreateCustomerInput, UpdateCustomerInput, ListCustomersQuery inferred from Zod schemas | customer.schemas.ts:31-33, imported into customer.service.ts:4-8 | Single source of truth for both runtime validation and compile-time typing of service method parameters |
| Partial Update / PATCH Semantics | Conditional field spreading based on `!== undefined` checks | customer.service.ts:51-57 | Implements idiomatic partial-update behavior distinct from full replacement |
| Method Reuse / Template-like Guard | `update` and `delete` both call `getById` for existence validation | customer.service.ts:44, 61 | DRY reuse of existence-check logic; consistent NotFoundError semantics across mutating operations |

No repository-pattern abstraction beyond the concrete `CustomerRepository` class was observed (i.e., no `ICustomerRepository` interface); `CustomerService` depends on the concrete class type, not an abstraction, which is a minor deviation from strict Dependency Inversion Principle (see Technical Debt).

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|-----------------|-------|--------|
| High | CustomerService (entire component) | No dedicated unit or integration tests exist for CustomerService's business logic (uniqueness rules, existence guards, partial update assembly) | Regressions in critical rules (e.g., email-uniqueness self-exclusion logic) would not be caught automatically; changes carry higher risk of silently breaking conflict/not-found semantics |
| Medium | create / update (email uniqueness check) | Check-then-act (TOCTOU) race condition: `findByEmail` followed by `create`/`update` is not atomic; concurrent requests with the same new email could both pass the check and one would fail on the database's unique constraint with an unhandled Prisma error rather than a clean ConflictError | Under concurrent load, a small percentage of duplicate-email create/update attempts may surface as unhandled 500-style errors instead of the intended 409 ConflictError, unless the global error middleware happens to generically handle unexpected exceptions |
| Low-Medium | update method | `getById(id)` result is fetched but discarded; the method re-derives "does this email belong to someone else" via a second `findByEmail` call rather than comparing against the already-fetched current customer record, or filtering out self in a single query | One extra database round-trip per update call with an email change; no correctness issue, but a minor efficiency and readability concern |
| Low | CustomerService constructor | Depends on the concrete `CustomerRepository` class rather than an interface/abstraction | Slightly weaker adherence to Dependency Inversion Principle; makes substituting an alternative persistence implementation (or a hand-rolled mock without relying on structural typing) marginally less explicit, though TypeScript's structural typing mitigates this in practice |
| Low | delete method | No handling for downstream referential-integrity failures (e.g., a customer with existing Orders, given the `Customer.orders Order[]` relation in the Prisma schema) | If the database enforces a foreign-key constraint preventing deletion of a customer with existing orders, this would surface as an unhandled database error rather than a clean, domain-specific error (e.g., a dedicated ConflictError for "customer has existing orders") |
| Low | list method | No explicit business-level cap or defensive check on pageSize/page within the service itself; relies entirely on upstream Zod validation | If CustomerService were ever invoked from a non-HTTP entry point (e.g., a script, another service, a queue consumer) without going through the Zod-validated route, unbounded or invalid pagination values could reach the repository unchecked |
| Informational | CustomerService (whole file) | No logging/observability hooks (no structured logging of create/update/delete outcomes) within the service layer | Reduced operational visibility into business-rule rejections (e.g., how often EMAIL_ALREADY_USED conflicts occur) unless captured elsewhere (e.g., request-logger middleware, which is generic and not customer-specific) |

---

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|---------------------|----------|----------------|
| CustomerService | 0 | 0 | 0% (no dedicated test file found) | N/A — no direct tests exist |
| CustomerController | 0 | 0 | 0% | N/A |
| CustomerRepository | 0 | 0 | 0% | N/A |
| customer.schemas.ts (Zod) | 0 | 0 | 0% (not directly tested; implicitly exercised via orders.test.ts factory-created customers, which always pass valid data) | N/A |

**Test files located in the project (full repository scan of `/tests`):**

- `tests/auth.test.ts` — Authentication flow tests; does not reference customers.
- `tests/orders.test.ts` — Order-management integration tests. Uses `createTestCustomer()` from `tests/helpers/factories.ts` purely as a **setup/fixture step** (to obtain a valid `customerId` for creating orders). It does not call any `CustomerService` method (list/getById/create/update/delete) as the subject under test, nor does it assert on customer-specific behavior (uniqueness conflicts, not-found errors, partial update semantics). References found at `tests/orders.test.ts:5,15,23,45,51,62,68,92,98,112,118,137,143,156,166,174,200,206`.
- `tests/helpers/factories.ts` — Contains `createTestCustomer(overrides)` (lines 42-49+), a Prisma-based factory that directly creates `Customer` rows via `prisma.customer.create(...)`, bypassing `CustomerService`/`CustomerController` entirely. This means even the customer data used in `orders.test.ts` does not exercise the component under analysis.
- `tests/setup.ts` — Generic test environment setup; no customer-specific content found.

**Coverage gaps identified (no test currently exists for):**
- `CustomerService.create` — happy path (successful creation)
- `CustomerService.create` — conflict path (duplicate email -> ConflictError/EMAIL_ALREADY_USED)
- `CustomerService.getById` — happy path and not-found path (NotFoundError)
- `CustomerService.update` — happy path, partial-field-only updates, not-found path, and the self-exclusion email-conflict logic (update without changing email; update changing email to another customer's email; update changing email to one's own unchanged email)
- `CustomerService.delete` — happy path and not-found path
- `CustomerService.list` — pagination math (skip calculation), empty search term behavior, and result shape (PaginatedResponse envelope correctness)
- End-to-end HTTP-level tests for the five `/customers` routes (status codes, response bodies, auth enforcement) — no `customers.test.ts`-equivalent file exists alongside `tests/orders.test.ts` and `tests/auth.test.ts`.

This represents a complete absence of direct automated verification for the component's business rules documented in Section 3, which is the most significant risk identified in this analysis (see Section 10, High-risk finding).

---

**Analysis scope note:** This report analyzes `CustomerService` (`src/modules/customers/customer.service.ts`) as the primary subject, with `CustomerController`, `CustomerRepository`, `customer.routes.ts`, `customer.schemas.ts`, and shared cross-cutting modules (`shared/errors`, `shared/http/response`, `middlewares/auth.middleware.ts`) examined only to the extent necessary to document the component's collaborators, data flow, and integration contract. No files were modified during this analysis.
