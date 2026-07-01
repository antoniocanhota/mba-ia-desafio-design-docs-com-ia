# Component Deep Analysis Report: CustomerController

**Project:** order-management-api (Order Management System REST API)
**Component analyzed:** `CustomerController`
**Primary file:** `src/modules/customers/customer.controller.ts`
**Analysis date:** 2026-06-30

---

## 1. Executive Summary

`CustomerController` is the HTTP boundary (presentation layer) for the Customers bounded context in the Order Management REST API. It is a thin Express adapter class that translates HTTP requests into calls against `CustomerService` and translates the results (or thrown errors) back into HTTP responses. It exposes five request handlers — `list`, `getById`, `create`, `update`, `delete` — each implemented as a class property arrow function conforming to Express's `RequestHandler` type, which allows the methods to be passed directly as route handlers without losing `this` binding.

The controller itself contains **no business logic**. It performs no validation, no authorization checks, and no data transformation beyond passing `req.query`, `req.params.id`, and `req.body` through to the service layer and shaping the HTTP response (status code + JSON body). All business rules (uniqueness of email, existence checks, partial update semantics, pagination calculation) live in `CustomerService` (`src/modules/customers/customer.service.ts`), and all input-shape validation is delegated to Zod schemas (`src/modules/customers/customer.schemas.ts`) applied via the `validate` middleware before the controller is ever invoked. This makes `CustomerController` a textbook example of the Controller pattern in a layered/clean architecture: it exists purely to decouple the transport protocol (HTTP/Express) from the application/domain logic.

Key findings:
- The controller is stateless, has a single constructor dependency (`CustomerService`), and follows dependency injection wired manually in `src/app.ts`.
- Every handler uses an identical try/catch-and-forward-to-`next(err)` pattern, delegating all error translation to the centralized `errorMiddleware`.
- No route in this module is directly exercised by an automated test. Coverage of the controller is indirect (via the `createTestCustomer` factory that bypasses the HTTP layer and writes directly to Prisma) and via structural similarity with `orders.test.ts`, which does test a sibling controller extensively. This represents the most significant testing gap for this component.
- The component sits behind global JWT authentication (`authenticate` middleware) but has **no role-based authorization** (`requireRole`) applied to any of its routes, meaning both `ADMIN` and `OPERATOR` roles have full CRUD access to customer records — a decision that may or may not be intentional business policy.

---

## 2. Data Flow Analysis

Data flow is identical in shape across all five operations; each is documented below in the order a request would traverse the stack.

### 2.1 `GET /api/v1/customers` (list)
```
1. Request enters Express router at buildCustomerRouter (customer.routes.ts:16)
2. authenticate middleware verifies JWT bearer token, populates req.user (auth.middleware.ts:27)
3. validate({ query: listCustomersQuerySchema }) parses/coerces page, pageSize, search (validate.middleware.ts:11; customer.schemas.ts:25-29)
4. CustomerController.list executes (customer.controller.ts:8-16)
5. req.query is cast to ListCustomersQuery and passed to CustomerService.list (customer.service.ts:14-21)
6. CustomerService computes skip/take from page/pageSize and calls CustomerRepository.list (customer.service.ts:15-19)
7. CustomerRepository.buildWhere constructs a Prisma OR filter over name/email/document (customer.repository.ts:12-22)
8. CustomerRepository.list executes a Prisma $transaction with findMany + count (customer.repository.ts:24-36)
9. CustomerService wraps results via paginated() helper (customer.service.ts:20; shared/http/response.ts:22-24)
10. CustomerController responds 200 with JSON { data, pagination } (customer.controller.ts:12)
11. On any thrown error, next(err) forwards to errorMiddleware (customer.controller.ts:13-14; error.middleware.ts:14-65)
```

### 2.2 `GET /api/v1/customers/:id` (getById)
```
1. authenticate middleware validates JWT
2. validate({ params: customerIdParamSchema }) enforces id is a UUID (customer.schemas.ts:3-5)
3. CustomerController.getById reads req.params.id and calls CustomerService.getById (customer.controller.ts:18-25)
4. CustomerService.getById calls CustomerRepository.findById (customer.service.ts:23-27)
5. If Prisma returns null, CustomerService throws NotFoundError('Customer') (customer.service.ts:25; shared/errors/http-errors.ts:27-31)
6. On success, controller responds 200 with the Customer JSON object
7. On NotFoundError, errorMiddleware maps to 404 { error: { code: 'NOT_FOUND', message: 'Customer not found' } }
```

### 2.3 `POST /api/v1/customers` (create)
```
1. authenticate middleware validates JWT
2. validate({ body: createCustomerSchema }) enforces name/email/phone/document/address shape (customer.schemas.ts:15-21)
3. CustomerController.create passes req.body (already parsed/typed) to CustomerService.create (customer.controller.ts:27-34)
4. CustomerService.create checks CustomerRepository.findByEmail for uniqueness (customer.service.ts:29-33)
5. If a customer with the same email exists, throws ConflictError('Email already registered', 'EMAIL_ALREADY_USED') (customer.service.ts:31-33)
6. Otherwise CustomerRepository.create persists the record via prisma.customer.create (customer.repository.ts:46-48)
7. Controller responds 201 with the created Customer JSON
8. Errors propagate to errorMiddleware (e.g., 409 for ConflictError, or Prisma P2002 unique-constraint race fallback at error.middleware.ts:37-47)
```

### 2.4 `PATCH /api/v1/customers/:id` (update)
```
1. authenticate middleware validates JWT
2. validate({ params: customerIdParamSchema, body: updateCustomerSchema }) validates id and partial body (customer.schemas.ts:23)
3. CustomerController.update calls CustomerService.update(id, req.body) (customer.controller.ts:36-43)
4. CustomerService.update first calls getById(id) to assert existence, throws NotFoundError otherwise (customer.service.ts:44)
5. If input.email is provided, checks findByEmail; throws ConflictError if a DIFFERENT customer owns that email (customer.service.ts:45-50)
6. CustomerService builds a sparse update object, including only defined fields (customer.service.ts:51-57)
7. CustomerRepository.update persists via prisma.customer.update (customer.repository.ts:50-52)
8. Controller responds 200 with the updated Customer JSON
9. Errors (404/409) propagate to errorMiddleware
```

### 2.5 `DELETE /api/v1/customers/:id` (delete)
```
1. authenticate middleware validates JWT
2. validate({ params: customerIdParamSchema }) validates id is UUID
3. CustomerController.delete calls CustomerService.delete(id) (customer.controller.ts:45-52)
4. CustomerService.delete calls getById(id) first to assert existence, throws NotFoundError otherwise (customer.service.ts:61)
5. CustomerRepository.delete executes prisma.customer.delete (customer.repository.ts:54-56)
6. Controller responds 204 with an empty body (res.status(204).send())
7. Errors propagate to errorMiddleware (e.g., 409 if a foreign key constraint from Order blocks deletion — not explicitly handled, falls through to generic Prisma error handling)
```

---

## 3. Business Rules & Logic

Note on scope and confidence: `CustomerController` itself is a pass-through layer and encodes no business rules directly. However, per the requirement to document ALL business rules governing this component's behavior, the rules below are extracted from the full request-processing pipeline that the controller triggers (schemas, service, repository), since these rules directly determine the controller's observable HTTP behavior (status codes, payloads, error responses). Each rule's source location is cited precisely.

## Overview of the business rules:

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation | Customer id path parameter must be a valid UUID | customer.schemas.ts:3-5 |
| Validation | Name must be a string between 2 and 150 characters | customer.schemas.ts:16 |
| Validation | Email must be a valid email format, max 255 characters | customer.schemas.ts:17 |
| Validation | Phone must be a string between 8 and 32 characters | customer.schemas.ts:18 |
| Validation | Document must be a string between 11 and 20 characters | customer.schemas.ts:19 |
| Validation | Address is a required nested object with street, number, city, state (2 chars), zipCode (5-15 chars) | customer.schemas.ts:7-13 |
| Validation | Update payload accepts any subset of create fields (fully partial) | customer.schemas.ts:23 |
| Validation | List pagination: page defaults to 1, minimum 1 | customer.schemas.ts:26 |
| Validation | List pagination: pageSize defaults to 20, min 1, max 100 | customer.schemas.ts:27 |
| Validation | Search term, if provided, is trimmed and must be non-empty | customer.schemas.ts:28 |
| Business Logic | Email must be unique across all customers on create | customer.service.ts:30-33 |
| Business Logic | Email must be unique across all customers on update, excluding the record being updated | customer.service.ts:45-50 |
| Business Logic | Customer must exist (404) before it can be read, updated, or deleted | customer.service.ts:25, 44, 61 |
| Business Logic | Update is a sparse/partial merge — only explicitly provided fields are sent to persistence | customer.service.ts:51-57 |
| Business Logic | Search filter matches partial, case-sensitive substring across name, email, or document (OR condition) | customer.repository.ts:12-22 |
| Business Logic | List results are always ordered by createdAt descending (newest first) | customer.repository.ts:29 |
| Business Logic | List pagination offset computed as (page - 1) * pageSize | customer.service.ts:17 |
| Business Logic | List retrieval and count are executed atomically in a single Prisma transaction | customer.repository.ts:26-34 |
| Authorization | All customer endpoints require a valid JWT bearer token; no role restriction (ADMIN and OPERATOR both permitted) | customer.routes.ts:14; auth.middleware.ts:27-47 |
| HTTP Contract | Create returns 201 with the created resource | customer.controller.ts:30 |
| HTTP Contract | Delete returns 204 with an empty body | customer.controller.ts:48 |
| HTTP Contract | List, getById, update return 200 with JSON payload | customer.controller.ts:12, 21, 39 |

## Detailed breakdown of the business rules:
---

### Business Rule: Email Uniqueness on Creation

**Overview**:
A new customer cannot be created if another customer already exists with the same email address. This is enforced at the application/service layer, independent of (but backed by) the database's unique constraint.

**Detailed description**:
When `CustomerController.create` is invoked, it forwards the already-schema-validated request body directly to `CustomerService.create` (customer.service.ts:29-41). Before persisting anything, the service performs an explicit lookup via `CustomerRepository.findByEmail(input.email)`, which issues a `prisma.customer.findUnique({ where: { email } })` query (customer.repository.ts:42-44). If a record is found, the service throws a `ConflictError` with the message `"Email already registered"` and the machine-readable code `EMAIL_ALREADY_USED` (customer.service.ts:31-33; shared/errors/http-errors.ts:33-37), which the centralized `errorMiddleware` translates into an HTTP 409 response with a structured JSON error body.

This rule exists to guarantee that email functions as a natural, de facto identifier for customers in the system, presumably to support customer lookup/deduplication and to prevent duplicate customer profiles for the same person or organization. The rule is reinforced at the data layer: the Prisma schema defines `email String @unique @db.VarChar(255)` on the `Customer` model (prisma/schema.prisma:43), meaning even if the application-level check were bypassed (e.g., due to a race condition between concurrent requests), the database would reject the second insert. In that race scenario, Prisma raises a `PrismaClientKnownRequestError` with code `P2002`, which `errorMiddleware` separately catches and maps to a generic 409 `CONFLICT` response (error.middleware.ts:37-47) — note this fallback path produces a different error code/message (`CONFLICT` / "Unique constraint violation on: ...") than the primary application-level check (`EMAIL_ALREADY_USED`), which is a minor inconsistency in the error contract exposed to API consumers.

From the controller's perspective, this rule is entirely invisible — the controller has no branching logic for it. It is documented here because it directly determines the HTTP status code and payload that `CustomerController.create` will emit for a given request, and is essential to understanding the component's observable behavior as a black box.

**Rule workflow**:
```
1. POST /api/v1/customers with body containing email
2. Schema validation passes (email is syntactically valid)
3. CustomerController.create calls CustomerService.create(body)
4. CustomerService.create calls CustomerRepository.findByEmail(email)
5a. If found -> throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED')
     -> errorMiddleware -> HTTP 409 { error: { code: 'EMAIL_ALREADY_USED', message: '...' } }
5b. If not found -> CustomerRepository.create(...) -> HTTP 201 with created customer
```

---

### Business Rule: Email Uniqueness on Update (Excluding Self)

**Overview**:
When updating a customer's email, the new email must not belong to a different existing customer. The customer being updated is exempted from the uniqueness check against itself.

**Detailed description**:
`CustomerService.update` (customer.service.ts:43-58) only performs the email-uniqueness check conditionally — `if (input.email)` (line 45) — meaning updates that do not touch the email field skip this check entirely. When an email is supplied, the service calls `findByEmail(input.email)` and, critically, compares the found record's `id` against the `id` parameter of the update call: `if (sameEmail && sameEmail.id !== id)` (customer.service.ts:47). This distinguishes between "the email belongs to the very customer being updated" (a no-op, allowed) and "the email belongs to a different customer" (a conflict, rejected). Without this exclusion, a customer could never issue a PATCH request that included their own unchanged email, since the lookup would always find a match.

This rule works in tandem with the existence check: `update` first calls `await this.getById(id)` (customer.service.ts:44), which throws `NotFoundError('Customer')` if the target record does not exist, before any email-related logic runs. This ordering means a request for a non-existent customer ID will always surface a 404, even if the email in the body also happens to collide with another record — existence is checked first, so the 404 takes precedence over a possible 409.

The business intent mirrors the creation rule: email is treated as a soft unique business key, and the system actively prevents two customer records from ever converging on the same email through either creation or mutation paths. This is important for downstream consumers (e.g., the Orders module, which references `Customer` via `customerId` foreign keys per `prisma/schema.prisma:50`) since any process that resolves a customer by email can rely on the invariant that at most one active customer record maps to a given email at any time.

**Rule workflow**:
```
1. PATCH /api/v1/customers/:id with optional email field
2. CustomerController.update calls CustomerService.update(id, body)
3. CustomerService.update calls getById(id)
   -> if not found: throw NotFoundError('Customer') -> HTTP 404
4. If body.email is present:
     a. findByEmail(body.email)
     b. if found AND found.id !== id -> throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED') -> HTTP 409
     c. else (not found, or found.id === id) -> proceed
5. Build sparse update object with only defined fields
6. CustomerRepository.update(id, sparseObject) -> HTTP 200 with updated customer
```

---

### Business Rule: Existence Check Before Read, Update, or Delete

**Overview**:
Any operation targeting a specific customer by ID (`getById`, `update`, `delete`) first verifies the customer exists, returning a uniform 404 Not Found if it does not.

**Detailed description**:
`CustomerService.getById` (customer.service.ts:23-27) is the canonical existence-check implementation: it calls `CustomerRepository.findById(id)`, and if the result is `null`, throws `NotFoundError('Customer')` — which produces the message `"Customer not found"` with HTTP status 404 and error code `NOT_FOUND` (shared/errors/http-errors.ts:27-31). Both `update` (customer.service.ts:44) and `delete` (customer.service.ts:61) reuse this exact method as their first operation, ensuring consistent 404 semantics across all three ID-based operations rather than each having bespoke existence logic. This is a deliberate application of the DRY principle inside the service layer.

A consequence of this design is a double round-trip to the database for both `update` and `delete`: first `getById` performs a `findUnique`, and then the actual mutation (`update`/`delete`) performs a second query against the same row. This trades a small amount of query efficiency for code simplicity and consistent error semantics, and also protects against Prisma throwing a less user-friendly `P2025` "Record not found" error directly from the `update`/`delete` calls (which is handled as a fallback at `error.middleware.ts:48-53`, but produces a generic `"Resource not found"` message rather than the more specific `"Customer not found"`).

For `getById` specifically (the read path), this is the only rule applied — there is no additional authorization or filtering (e.g., no soft-delete flag, no ownership check) — so any authenticated user (regardless of role) can retrieve any customer record by ID as long as it exists.

**Rule workflow**:
```
1. Request targets /api/v1/customers/:id (GET, PATCH, or DELETE)
2. Route validation confirms :id is a syntactically valid UUID
3. Controller forwards to the relevant CustomerService method
4. Service calls getById(id) [directly for GET, or as a pre-check for PATCH/DELETE]
5. CustomerRepository.findById(id) queries Prisma
6a. If null -> throw NotFoundError('Customer') -> HTTP 404 { error: { code: 'NOT_FOUND', message: 'Customer not found' } }
6b. If found -> proceed with the requested operation
```

---

### Business Rule: Partial (Sparse) Update Semantics

**Overview**:
The update operation only modifies fields explicitly present in the request body; omitted fields retain their existing values.

**Detailed description**:
The `updateCustomerSchema` is derived from `createCustomerSchema.partial()` (customer.schemas.ts:23), which makes every field (`name`, `email`, `phone`, `document`, `address`) optional at the validation layer — a client may submit an empty object `{}` and it will pass validation. At the service layer, `CustomerService.update` builds the object passed to the repository using conditional spread syntax for every field: `...(input.name !== undefined ? { name: input.name } : {})` and equivalently for `email`, `phone`, `document`, and `address` (customer.service.ts:51-57). This ensures that `undefined` fields are never included in the object sent to `CustomerRepository.update`, and therefore Prisma's `update` call will not overwrite those columns with `undefined`/null.

This is a meaningful distinction from a naive `{ ...input }` spread, which in TypeScript/JavaScript would still include keys with `undefined` values in the resulting object (though Prisma generally ignores `undefined` values in its update payloads as well — this implementation is a defensive, explicit safeguard rather than a strict necessity given Prisma's behavior, but it makes the intent self-documenting in the code).

One nuance: the `address` field, if provided, is replaced wholesale rather than merged at the sub-field level. Since `address` is stored as a Prisma `Json` column (prisma/schema.prisma:46) and the schema for `address` in `updateCustomerSchema` inherits the full nested `addressSchema` object (street, number, city, state, zipCode) as a single unit via `.partial()` on the top-level object (not a deep partial), a client wishing to update just the `city` field, for example, must resend the ENTIRE address object with all five sub-fields, or the nested Zod validation will fail (all sub-fields of `addressSchema` are required, not optional, since `.partial()` is only applied to the top-level `createCustomerSchema`, not recursively into `addressSchema`).

**Rule workflow**:
```
1. PATCH /api/v1/customers/:id with body containing zero or more fields
2. updateCustomerSchema (fully partial) validates only the fields present
3. CustomerService.update builds sparse object: only defined input fields are included
4. CustomerRepository.update sends only those fields to Prisma's update()
5. Unspecified fields remain unchanged in the database
6. If address is included, it must be a complete address object (all 5 sub-fields required)
```

---

### Business Rule: Search and Pagination on List

**Overview**:
Listing customers supports optional free-text search across name, email, and document, combined with mandatory page-based pagination, always sorted by creation date descending.

**Detailed description**:
The `listCustomersQuerySchema` (customer.schemas.ts:25-29) defines three query parameters: `page` (coerced to integer, minimum 1, default 1), `pageSize` (coerced to integer, minimum 1, maximum 100, default 20), and an optional `search` string that is trimmed and must be non-empty if provided. These are coerced from the raw string query parameters that Express provides (`z.coerce.number()`), so `?page=2` in the URL becomes the number `2`, not the string `"2"`, before it ever reaches the service.

`CustomerService.list` (customer.service.ts:14-21) translates the page/pageSize pair into a Prisma-style `skip`/`take` pair via `skip: (query.page - 1) * query.pageSize` and `take: query.pageSize`, then delegates to `CustomerRepository.list`. The repository's `buildWhere` method (customer.repository.ts:12-22) constructs a Prisma `WHERE` clause only if `search` is present; when present, it builds an `OR` condition matching a `contains` (case-sensitive substring, as no `mode: 'insensitive'` option is passed) filter across `name`, `email`, and `document` simultaneously — meaning a search term appearing in ANY of those three fields will match a customer, not requiring it to appear in all three.

The repository executes the paginated fetch and the total count as a single atomic Prisma `$transaction` (customer.repository.ts:26-34), which guarantees that `total` (used to compute `totalPages` in the response) reflects a consistent snapshot relative to the `items` returned, even under concurrent writes — this prevents pagination metadata from becoming inconsistent with the actual returned page under load. Results are always ordered by `createdAt: 'desc'` (customer.repository.ts:29), meaning the newest-created customers appear first; there is no way for a client to override this sort order, as no sort parameter is exposed in the schema.

**Rule workflow**:
```
1. GET /api/v1/customers?page=2&pageSize=10&search=acme
2. listCustomersQuerySchema coerces/validates/defaults page, pageSize, search
3. CustomerController.list forwards typed query to CustomerService.list
4. CustomerService computes skip = (page-1)*pageSize, take = pageSize
5. CustomerRepository.buildWhere constructs OR filter over name/email/document if search present
6. CustomerRepository.list executes findMany + count in a single $transaction, ordered by createdAt desc
7. CustomerService wraps items via paginated() -> { data, pagination: { page, pageSize, total, totalPages } }
8. Controller responds 200 with JSON payload
```

---

## 4. Component Structure

```
src/modules/customers/
├── customer.controller.ts   # HTTP adapter: 5 RequestHandlers (list, getById, create, update, delete);
│                             # no business logic; delegates to CustomerService; forwards errors to next()
├── customer.service.ts      # Business logic: uniqueness checks, existence checks, pagination math,
│                             # sparse-update construction
├── customer.repository.ts   # Data access: Prisma queries (findMany/count/findUnique/create/update/delete),
│                             # WHERE-clause construction for search
├── customer.routes.ts       # Express Router wiring: auth + validate middlewares bound to each handler
└── customer.schemas.ts      # Zod schemas: request shape validation, coercion, and defaults;
                              # exported inferred TypeScript types consumed by service/controller

Related cross-cutting files (outside module boundary, referenced by the controller's pipeline):
src/middlewares/
├── auth.middleware.ts        # JWT authentication (authenticate) and RBAC helper (requireRole) — the latter is
│                              # NOT applied to customer routes
├── validate.middleware.ts    # Generic Zod-based request validator (body/query/params)
└── error.middleware.ts       # Centralized error-to-HTTP-response translation

src/shared/
├── errors/                   # AppError hierarchy: NotFoundError, ConflictError, ValidationError, etc.
└── http/response.ts          # Pagination/PaginatedResponse helper types and builder functions

src/routes/index.ts            # Mounts buildCustomerRouter under /api/v1/customers
src/app.ts                     # Dependency injection wiring: constructs Repository -> Service -> Controller
prisma/schema.prisma            # Customer Prisma model (source of truth for persisted shape and DB constraints)
```

---

## 5. Dependency Analysis

```
Internal Dependencies (compile-time imports):

CustomerController
  -> CustomerService (constructor-injected; customer.controller.ts:2,6)
  -> ListCustomersQuery (type-only import from customer.schemas.ts:3)
  -> express.RequestHandler (type-only import)

CustomerService
  -> CustomerRepository (constructor-injected)
  -> ConflictError, NotFoundError (shared/errors)
  -> CreateCustomerInput, ListCustomersQuery, UpdateCustomerInput (types from customer.schemas.ts)
  -> paginated, PaginatedResponse (shared/http/response.ts)
  -> Customer (Prisma generated type)

CustomerRepository
  -> PrismaClient, Customer, Prisma (namespace) (@prisma/client)

customer.routes.ts
  -> authenticate (middlewares/auth.middleware.ts)
  -> validate (middlewares/validate.middleware.ts)
  -> createCustomerSchema, updateCustomerSchema, customerIdParamSchema, listCustomersQuerySchema
  -> CustomerController (type-only)

src/routes/index.ts
  -> buildCustomerRouter, CustomerController (mounts under /api/v1/customers)

src/app.ts
  -> Instantiates CustomerRepository(prisma) -> CustomerService(repo) -> CustomerController(service)
  -> Registers errorMiddleware globally, which handles all errors thrown/forwarded by CustomerController

Full call chain:
app.ts (DI) -> routes/index.ts (mount) -> customer.routes.ts (middleware chain) ->
  CustomerController -> CustomerService -> CustomerRepository -> Prisma Client -> PostgreSQL/DB

External Dependencies:
- express (^4.x, exact version not pinned in files reviewed) - HTTP routing and RequestHandler typing
- @prisma/client - ORM client and generated types (Customer, Prisma.CustomerWhereInput,
  Prisma.CustomerUncheckedCreateInput, Prisma.CustomerUncheckedUpdateInput, PrismaClient)
- zod - Schema validation, type inference, and coercion (used by customer.schemas.ts, validate.middleware.ts)
- jsonwebtoken - Used transitively via auth.middleware.ts (authenticate) for JWT verification

Note: package.json was not located within the analyzed scope to confirm exact pinned versions;
      versions above are inferred from import usage patterns only.
```

---

## 6. Afferent and Efferent Coupling

Coupling is analyzed at the class/module level, consistent with this project's object-oriented TypeScript structure.

```
| Component            | Afferent Coupling | Efferent Coupling | Critical |
|-----------------------|-------------------|--------------------|----------|
| CustomerController    | 2 (routes/index.ts, customer.routes.ts) | 1 (CustomerService) | Low |
| CustomerService        | 1 (CustomerController) | 3 (CustomerRepository, shared/errors, shared/http/response) | Medium |
| CustomerRepository     | 1 (CustomerService) | 1 (PrismaClient / @prisma/client) | Medium |
| customer.schemas.ts    | 3 (CustomerController [types], CustomerService [types], customer.routes.ts) | 1 (zod) | Low |
| customer.routes.ts     | 1 (routes/index.ts) | 4 (CustomerController, auth.middleware, validate.middleware, customer.schemas.ts) | Low |
```

`CustomerController` itself exhibits low afferent coupling (only referenced by the module's own router and the top-level route index) and minimal efferent coupling (a single dependency on `CustomerService`), which is the expected and desirable profile for a controller in a layered architecture — it should be a thin, easily replaceable adapter. `CustomerService` is the most structurally significant node in this module, as it is the sole point of contact between the transport layer and the persistence layer, and encapsulates all business rules; it warrants Medium criticality because any defect there directly affects all five controller operations.

---

## 7. Endpoints

All endpoints are mounted under `/api/v1/customers` (see `src/routes/index.ts:26`) and require a valid JWT bearer token via the `authenticate` middleware (`customer.routes.ts:14`). No role-based restriction (`requireRole`) is applied to any customer route — both `ADMIN` and `OPERATOR` roles have equal access.

| Endpoint | Method | Description | Auth Required | Request Validation |
|----------|--------|--------------|----------------|---------------------|
| /api/v1/customers | GET | List customers with pagination and optional search (name/email/document) | Yes (any authenticated role) | Query: page, pageSize, search |
| /api/v1/customers/:id | GET | Retrieve a single customer by UUID | Yes (any authenticated role) | Params: id (UUID) |
| /api/v1/customers | POST | Create a new customer | Yes (any authenticated role) | Body: name, email, phone, document, address |
| /api/v1/customers/:id | PATCH | Partially update an existing customer | Yes (any authenticated role) | Params: id (UUID); Body: any subset of create fields |
| /api/v1/customers/:id | DELETE | Delete a customer by UUID | Yes (any authenticated role) | Params: id (UUID) |

Response status codes observed in controller code:
- `200 OK` — list, getById, update (customer.controller.ts:12, 21, 39)
- `201 Created` — create (customer.controller.ts:30)
- `204 No Content` — delete (customer.controller.ts:48)
- Error responses (400/401/403/404/409/422/500) are produced exclusively by `errorMiddleware`, never directly by the controller.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|-----------------|
| CustomerService | Internal (in-process) | Business logic execution (uniqueness, existence checks, pagination) | Direct method call (TypeScript) | In-memory objects / typed interfaces | Exceptions (AppError subclasses) propagated via try/catch to next(err) |
| PostgreSQL (via Prisma) | External Datastore | Persistence of Customer entities | Prisma Client (SQL under the hood) | Relational rows mapped to Customer type / JSON column for address | Prisma errors (P2002, P2025) caught centrally in errorMiddleware; not handled at controller/service level |
| JWT Auth Provider (self-issued tokens) | Cross-cutting / Internal | Authenticates the caller before reaching the controller | HTTP Header (Authorization: Bearer) | JWT (signed with env.JWT_SECRET) | UnauthorizedError thrown by auth.middleware.ts on missing/invalid/expired token |
| Zod Validation Layer | Internal | Validates/coerces request query, params, and body before controller execution | In-process middleware | JSON (HTTP body/query/params) | ValidationError (400) with field-level details on failure |
| Orders module (indirect, via Customer.orders relation) | Internal / Data Relationship | Orders reference customers by customerId foreign key | Not called directly by CustomerController; relationship enforced at DB schema level (prisma/schema.prisma:50) | N/A | Not handled by CustomerController; a DELETE of a customer with existing orders would surface as a raw Prisma foreign-key error, likely falling through to the generic 500 handler since it is not one of the explicitly handled Prisma error codes (P2002/P2025) in error.middleware.ts |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|-----------------|----------|---------|
| Layered Architecture (Controller-Service-Repository) | CustomerController -> CustomerService -> CustomerRepository | customer.controller.ts, customer.service.ts, customer.repository.ts | Separation of transport, business logic, and persistence concerns |
| Dependency Injection (manual, constructor-based) | `constructor(private readonly customers: CustomerService)` | customer.controller.ts:6 | Decouples controller from concrete service implementation; enables testability |
| Repository Pattern | CustomerRepository wraps all Prisma access | customer.repository.ts | Abstracts persistence details from the service layer |
| Middleware Chain / Chain of Responsibility | authenticate -> validate -> controller handler -> errorMiddleware | customer.routes.ts:14-24; app.ts:73 | Cross-cutting concerns (auth, validation, error formatting) applied uniformly without controller involvement |
| Centralized Error Handling | try/catch + next(err) in every handler, single errorMiddleware | customer.controller.ts (all handlers); error.middleware.ts | Consistent HTTP error contract across the entire API, not just this module |
| Schema-Derived Types (Zod-first design) | `z.infer<typeof createCustomerSchema>` etc. | customer.schemas.ts:31-33 | Single source of truth for both runtime validation and compile-time types |
| DTO/Partial Update Pattern | `createCustomerSchema.partial()` + conditional spread in service | customer.schemas.ts:23; customer.service.ts:51-57 | Supports PATCH semantics without accidental field overwrites |
| Command Query Separation (informal) | list/getById are pure reads; create/update/delete are mutating commands | customer.controller.ts | Clear read/write boundary within the same controller |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|------------------|-------|--------|
| High | customer.routes.ts | No `requireRole` authorization applied to any customer route; any authenticated user (ADMIN or OPERATOR) can create, update, or delete customer records | Potential unauthorized data modification if OPERATOR role is intended to have restricted CRUD access (cannot confirm intended policy from code alone — flagged as ambiguous business rule) |
| High | Test coverage (project-wide, affecting this component) | No dedicated test file exists for customer.controller.ts, customer.service.ts, or customer.routes.ts; customer creation in tests bypasses the HTTP layer entirely via direct Prisma writes (tests/helpers/factories.ts:42-60) | The controller's HTTP contract (status codes, error mapping, validation behavior) is entirely unverified by automated tests |
| Medium | customer.controller.ts:20,38,47 | `req.params.id!` uses a non-null assertion; if Express routing somehow provided an undefined id (e.g., misconfigured route), this would produce a runtime TypeError rather than a graceful error response, though in practice validate.middleware.ts should always populate params.id given the schema requires it | Low likelihood but poor failure mode if it occurs; a defensive check would be more robust |
| Medium | customer.repository.ts:17-19 | Search filter uses `contains` without `mode: 'insensitive'`, making search case-sensitive; this may not match user expectations for a "search" feature (e.g., searching "Acme" would not match "acme corp") | Reduced usability of the list/search endpoint; potential for confusing empty results |
| Medium | error.middleware.ts vs customer.service.ts | Two different error codes/messages can be returned for the same underlying "duplicate email" scenario depending on whether the application-level check or the database race-condition fallback fires (`EMAIL_ALREADY_USED` vs generic `CONFLICT`) | Inconsistent API error contract for API consumers under concurrent request scenarios |
| Low | customer.schemas.ts:23 | `updateCustomerSchema` only applies `.partial()` at the top level; the nested `address` object is not deep-partial, forcing clients to resend the entire address object even to update a single sub-field | Suboptimal API ergonomics for partial address updates; not a correctness bug |
| Low | Foreign key relationship | Customer.orders relation (prisma/schema.prisma:50) is not explicitly guarded against in customer.service.ts's delete method; deleting a customer with existing orders would rely entirely on the database's referential integrity behavior and the generic Prisma error fallback | Could produce a confusing 500 error instead of a clear 409 Conflict if a customer with orders is deleted, since P2003 (foreign key constraint) is not explicitly handled in error.middleware.ts |
| Low | customer.controller.ts | No request/response logging, tracing, or metrics instrumentation is present within the controller methods themselves (relies entirely on the global requestLogger middleware) | Reduced observability for debugging customer-specific issues, though this is consistent with the project's overall logging strategy |

---

## 11. Test Coverage Analysis

Test files located in the project (searched via `*.test.ts` and `*.spec.ts` patterns, excluding `node_modules`):
- `tests/auth.test.ts`
- `tests/orders.test.ts`
- `tests/setup.ts` (global test lifecycle: `prisma.customer.deleteMany()` cleanup on line 14)
- `tests/helpers/factories.ts` (test data factories, including `createTestCustomer`)

No file named `customer.test.ts`, `customers.test.ts`, or equivalent was found anywhere in the project.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|---------------------|----------|----------------|
| CustomerController | 0 | 0 | 0% (no direct tests) | N/A — component is entirely untested at the HTTP layer |
| CustomerService | 0 | 0 | 0% (no direct tests) | N/A — business rules (email uniqueness, existence checks, sparse update) are unverified by automated tests |
| CustomerRepository | 0 | 0 (indirect only) | Indirectly exercised via `createTestCustomer` factory, which calls `prisma.customer.create` directly, bypassing `CustomerRepository` entirely | N/A |
| customer.schemas.ts | 0 | 0 | 0% | Zod validation rules (min/max lengths, UUID format, email format, pagination bounds) are unverified |
| customer.routes.ts | 0 | 0 | 0% | Middleware wiring (authenticate + validate) is unverified for this module specifically |

**Observations:**
- `tests/orders.test.ts` uses `createTestCustomer()` (tests/helpers/factories.ts:42-60) purely as setup/fixture data to create orders — it inserts customer rows directly into the database via `prisma.customer.create`, completely bypassing `CustomerController`, `CustomerService`, and `CustomerRepository`. This means the Customers HTTP API surface (all 5 endpoints) has zero direct or indirect automated test coverage.
- By contrast, `tests/orders.test.ts` extensively exercises the sibling `OrderController` through actual HTTP requests via `supertest` (e.g., `request(app).post(...)`), demonstrating that the project's testing pattern and infrastructure (via `tests/helpers/factories.ts:10-15` `getTestApp()`) is fully capable of testing `CustomerController` in the same style — the absence of such tests appears to be a coverage gap rather than a tooling limitation.
- No unit tests (with mocked dependencies) exist for `CustomerService`'s business rules (email uniqueness, existence checks, partial update construction) or for `CustomerRepository`'s query construction (`buildWhere`), meaning these rules — documented in Section 3 — are currently validated only by manual code review, not automated verification.
- This represents the most significant risk identified in this analysis: a change to `CustomerController`, `CustomerService`, or `customer.schemas.ts` could silently break create/update/delete/list/getById behavior (e.g., breaking the email uniqueness check, breaking pagination math, or breaking a validation rule) without any automated test suite detecting the regression.

---

*End of report.*
